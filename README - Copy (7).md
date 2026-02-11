

## Reasoning Through the Design

This is a fascinating architectural problem. Let me think through it carefully.

### The Core Problem

Right now FRANZ has a **hardcoded goal** in `USER_PROMPT_TEMPLATE`. When that goal is achieved, the agent has no concept of "done" — it will keep trying to save a cat.jpg that's already saved, clicking around in confusion. And there's no way for the user to redirect it without editing code.

### The Key Insight: Screen as the Only Communication Channel

The user communicates with FRANZ **only through what's visible on screen**. This is actually powerful — it means:

- The user can open Notepad and type "Franz, please open Chrome and search for weather"
- The user can leave a sticky note on the desktop
- The user can have a dedicated "Franz Inbox" — a text file always open somewhere
- FRANZ detects these messages **visually**, just like it detects everything else

### The State Machine

FRANZ needs to operate in two modes:

```
┌──────────────┐     task completed      ┌──────────────┐
│              │ ──────────────────────▶  │              │
│   EXECUTING  │                          │  MONITORING  │
│              │ ◀──────────────────────  │              │
└──────────────┘   task detected on       └──────────────┘
                   screen / message found
```

But here's the thing — **the VLM itself should decide the transitions**. We don't hardcode state machine logic; we give the VLM the *vocabulary* to express state transitions, and it decides when to use them.

### What the VLM Needs to Output (New)

In addition to actions and story, the VLM should output:

1. **`goal:`** — the current operative goal (can be self-modified)
2. **`status:`** — one of `executing`, `completed`, `monitoring`

When status is `completed`, FRANZ automatically switches the goal to a monitoring/idle goal. When monitoring, the VLM scans the screen for user messages or unfinished work.

### The Monitoring Mode Behavior

When idle/monitoring, FRANZ should:
- **NOT click or type** (avoid disrupting the user)
- Scan the screen for: text addressed to "Franz", open unsaved documents, error dialogs, etc.
- If it finds something, it formulates a new goal and switches to `executing`
- Take screenshots at a slower cadence (no need to burn inference at 2Hz when idle)

### Implementation Plan

Here's what I'd modify:

| Change | Why |
|---|---|
| Add `goal` as a mutable variable alongside `story` | The goal is no longer a constant template — it's part of the evolving state |
| Add `status` field to VLM output parsing | VLM declares whether it's executing, completed, or monitoring |
| Modify `SYSTEM_PROMPT` to explain the mode system | VLM needs to know it can transition between modes |
| Modify `USER_PROMPT_TEMPLATE` to include `{goal}` | Goal is injected dynamically, not hardcoded |
| Add a default monitoring goal constant | What FRANZ does when it has no task |
| Add cadence control (slower polling when monitoring) | Don't waste inference when idle |
| Add `observe()` as an explicit "no-op" action the VLM can choose | Important for monitoring mode — the VLM needs a way to say "I looked and there's nothing to do" |
| Initial goal is the cat-drawing task (or monitoring, configurable) | Backward compatible |

Here's the full modified code:

```python
# main.py
"""FRANZ - AI-driven Windows automation agent with smooth, human-like interactions.

Supports two operational modes:
  - EXECUTING: actively working toward a goal with actions
  - MONITORING: passively observing the screen for user messages or unfinished work

The VLM controls mode transitions through a structured output protocol.
"""

import base64
import ctypes
import ctypes.wintypes
import json
import re
import time
import urllib.request
from datetime import datetime
from pathlib import Path
from typing import Final

import drawing
import screenshot

# ─── Configuration ──────────────────────────────────────────────────────────

API: Final = "http://localhost:1234/v1/chat/completions"
MODEL: Final = "qwen3-vl-2b-instruct-1m"
TARGET_WIDTH: Final = 1536
TARGET_HEIGHT: Final = 864
SAMPLING: Final = {"temperature": 0.7, "top_p": 0.9, "max_tokens": 1200}
ENABLE_VISUAL_FEEDBACK: Final = True

# ─── Timing ─────────────────────────────────────────────────────────────────

MOVEMENT_STEPS: Final = 20
STEP_DELAY: Final = 0.01
CLICK_SETTLE_DELAY: Final = 0.15
TYPING_CHAR_DELAY: Final = 0.08
TYPING_WORD_DELAY: Final = 0.15

# ─── Cadence control ────────────────────────────────────────────────────────

EXECUTING_LOOP_DELAY: Final = 0.5      # seconds between iterations when executing
MONITORING_LOOP_DELAY: Final = 3.0     # seconds between iterations when monitoring (save resources)

# ─── Agent modes ─────────────────────────────────────────────────────────────

MODE_EXECUTING: Final = "executing"
MODE_MONITORING: Final = "monitoring"

MONITORING_GOAL: Final = (
    "You are in monitoring mode. Carefully scan the screen for: "
    "(1) any visible message or note addressed to Franz or the AI assistant, "
    "(2) any unfinished work such as unsaved documents, open dialogs requiring input, "
    "error messages, or incomplete tasks, "
    "(3) any application state that suggests the user needs help. "
    "If you find something actionable, formulate a clear goal and switch to executing mode. "
    "If the screen shows a normal idle desktop with no tasks, remain in monitoring mode."
)

INITIAL_GOAL: Final = (
    'Draw a cat in MS Paint, then save it on the desktop as "cat.jpg".'
)

# ─── Prompts ─────────────────────────────────────────────────────────────────

SYSTEM_PROMPT: Final = """\
You are FRANZ, an autonomous desktop agent that perceives the screen and acts.

## Perception
You see a screenshot of the current desktop state. A visual overlay may show your last action.

## Output Format
You MUST output a structured response with these sections in order:

```
STATUS: executing | monitoring
GOAL: <current goal — rewrite this if the goal has changed or task is complete>
ACTION: <exactly one function call OR observe()>
STORY: <3 sentences: what you see, what you're doing, what you expect next>
```

## Available Actions (coordinates: 0-1000 normalised)
left_click(x, y)
right_click(x, y)
double_left_click(x, y)
drag(start_x, start_y, end_x, end_y)
type("text")
observe()

## Mode Behavior

### executing
You are working toward the current GOAL. Output exactly one action per step.
When the goal is COMPLETE, set STATUS to monitoring and set GOAL to describe what you'll watch for.

### monitoring
You are scanning the screen for work. Do NOT click or type unless you find a task.
Use observe() to indicate you checked and found nothing.
If you spot a message to Franz, an unfinished task, or something requiring attention:
set STATUS to executing and set GOAL to describe the new task.

## Rules
- One action per response. Then wait for the next screenshot.
- NEVER repeat an action on an already-completed goal.
- When you detect a task is finished (e.g. file saved, dialog closed), switch to monitoring.
- In monitoring mode, prefer observe() — do not interact unless you see a clear task.
- The STORY carries your memory between iterations. Write it carefully.
"""

USER_PROMPT_TEMPLATE: Final = """\
Current mode: {mode}
Current goal: {goal}

Story so far:
{story}

Last action performed: {last}

Look at the screenshot and produce your next structured response.
"""


# ─── Inference ───────────────────────────────────────────────────────────────

def infer(png_data: bytes, mode: str, goal: str, story: str, last: str) -> str:
    """Send inference request to the VLM API."""
    user_text = USER_PROMPT_TEMPLATE.format(
        mode=mode, goal=goal, story=story, last=last
    )

    payload = {
        "model": MODEL,
        "messages": [
            {"role": "system", "content": SYSTEM_PROMPT},
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": user_text},
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/png;base64,{base64.b64encode(png_data).decode()}"
                        },
                    },
                ],
            },
        ],
        **SAMPLING,
    }

    req = urllib.request.Request(
        API,
        json.dumps(payload).encode(),
        {"Content-Type": "application/json"},
    )

    with urllib.request.urlopen(req) as f:
        return json.load(f)["choices"][0]["message"]["content"]


# ─── Response Parsing ────────────────────────────────────────────────────────

_FUNC_CALL_RE = re.compile(
    r"\b(left_click|right_click|double_left_click|drag|type|observe)\s*\(([^)]*)\)"
)

_STATUS_RE = re.compile(r"STATUS\s*:\s*(executing|monitoring)", re.IGNORECASE)
_GOAL_RE = re.compile(r"GOAL\s*:\s*(.+)", re.IGNORECASE)
_STORY_RE = re.compile(r"STORY\s*:\s*(.+)", re.IGNORECASE | re.DOTALL)


def parse_response(content: str) -> dict:
    """Parse the VLM's structured response.

    Returns a dict with keys:
        status:   'executing' | 'monitoring' | None
        goal:     str | None
        commands: list[tuple[str, list]]   — parsed (func_name, params) pairs
        story:    str
    """
    result = {
        "status": None,
        "goal": None,
        "commands": [],
        "story": "",
    }

    # Extract STATUS
    if m := _STATUS_RE.search(content):
        result["status"] = m.group(1).lower()

    # Extract GOAL (everything after "GOAL:" until the next section keyword or end)
    if m := _GOAL_RE.search(content):
        goal_text = m.group(1).strip()
        # Truncate at next section keyword if present
        for keyword in ("ACTION:", "STORY:", "STATUS:"):
            idx = goal_text.upper().find(keyword)
            if idx != -1:
                goal_text = goal_text[:idx].strip()
        result["goal"] = goal_text

    # Extract STORY (everything after "STORY:" until end or next section)
    if m := _STORY_RE.search(content):
        story_text = m.group(1).strip()
        for keyword in ("ACTION:", "GOAL:", "STATUS:"):
            idx = story_text.upper().find(keyword)
            if idx != -1:
                story_text = story_text[:idx].strip()
        result["story"] = story_text

    # Extract function calls from anywhere in the response
    # (the VLM might put them in a code fence or inline after ACTION:)
    for match in _FUNC_CALL_RE.finditer(content):
        func_name = match.group(1)
        raw_args = match.group(2).strip()

        if func_name == "observe":
            result["commands"].append(("observe", []))
            continue

        params = _parse_args(func_name, raw_args)
        if params is not None:
            result["commands"].append((func_name, params))

    return result


def _parse_args(func_name: str, raw_args: str) -> list | None:
    """Parse raw argument string into a typed parameter list."""
    if func_name == "type":
        text = raw_args.strip()
        if (text.startswith('"') and text.endswith('"')) or \
           (text.startswith("'") and text.endswith("'")):
            text = text[1:-1]
        return [text] if text else None

    nums = re.findall(r"-?\d+", raw_args)

    if func_name in ("left_click", "right_click", "double_left_click") and len(nums) >= 2:
        return [int(nums[0]), int(nums[1])]
    elif func_name == "drag" and len(nums) >= 4:
        return [int(nums[0]), int(nums[1]), int(nums[2]), int(nums[3])]

    return None


# ─── Visualisation ───────────────────────────────────────────────────────────

def create_visualization(last_action: str) -> callable:
    """Create drawing function based on last action."""
    def draw_annotations(rgba: bytes, width: int, height: int) -> bytes:
        if not ENABLE_VISUAL_FEEDBACK or last_action in ("init", "observe", "observe()"):
            return rgba

        match = _FUNC_CALL_RE.search(last_action)
        if not match:
            return rgba

        func_name = match.group(1)
        raw_args = match.group(2).strip()
        params = _parse_args(func_name, raw_args)
        if params is None:
            return rgba

        if func_name in ("left_click", "double_left_click") and len(params) == 2:
            x = drawing.normalize_coord(params[0], width)
            y = drawing.normalize_coord(params[1], height)
            rgba = drawing.draw_crosshair(rgba, width, height, x, y, size=25, color=drawing.RED, thickness=3)
            rgba = drawing.draw_circle(rgba, width, height, x, y, radius=40, color=drawing.GREEN, filled=False)

        elif func_name == "right_click" and len(params) == 2:
            x = drawing.normalize_coord(params[0], width)
            y = drawing.normalize_coord(params[1], height)
            rgba = drawing.draw_crosshair(rgba, width, height, x, y, size=25, color=drawing.BLUE, thickness=3)
            rgba = drawing.draw_circle(rgba, width, height, x, y, radius=40, color=drawing.YELLOW, filled=False)

        elif func_name == "drag" and len(params) == 4:
            x1 = drawing.normalize_coord(params[0], width)
            y1 = drawing.normalize_coord(params[1], height)
            x2 = drawing.normalize_coord(params[2], width)
            y2 = drawing.normalize_coord(params[3], height)
            rgba = drawing.draw_arrow(rgba, width, height, x1, y1, x2, y2, color=drawing.BLUE, thickness=4)
            rgba = drawing.draw_circle(rgba, width, height, x1, y1, radius=15, color=drawing.YELLOW, filled=True)
            rgba = drawing.draw_circle(rgba, width, height, x2, y2, radius=15, color=drawing.GREEN, filled=True)

        return rgba

    return draw_annotations


# ─── Input Execution ─────────────────────────────────────────────────────────

def get_cursor_position() -> tuple[int, int]:
    """Get current cursor position."""
    point = ctypes.wintypes.POINT()
    ctypes.windll.user32.GetCursorPos(ctypes.byref(point))
    return point.x, point.y


def smooth_move_to(target_x: int, target_y: int, steps: int = MOVEMENT_STEPS, delay: float = STEP_DELAY) -> None:
    """Smoothly move cursor from current position to target position."""
    start_x, start_y = get_cursor_position()

    for i in range(steps + 1):
        t = i / steps
        t = t * t * (3 - 2 * t)  # Smoothstep

        x = int(start_x + (target_x - start_x) * t)
        y = int(start_y + (target_y - start_y) * t)

        ctypes.windll.user32.SetCursorPos(x, y)
        time.sleep(delay)


def press_key(vk_code: int) -> None:
    """Press and release a virtual key."""
    ctypes.windll.user32.keybd_event(vk_code, 0, 0, 0)
    time.sleep(0.02)
    ctypes.windll.user32.keybd_event(vk_code, 0, 2, 0)


def type_text(text: str) -> None:
    """Type text with natural human-like timing."""
    print(f'  → Typing: "{text}"')

    for char in text:
        if char == ' ':
            press_key(0x20)
            time.sleep(TYPING_WORD_DELAY)
        elif char == '\n':
            press_key(0x0D)
            time.sleep(TYPING_WORD_DELAY)
        else:
            vk = ctypes.windll.user32.VkKeyScanW(ord(char))
            if vk != -1:
                shift_needed = (vk >> 8) & 1
                vk_code = vk & 0xFF

                if shift_needed:
                    ctypes.windll.user32.keybd_event(0x10, 0, 0, 0)
                    time.sleep(0.01)

                press_key(vk_code)

                if shift_needed:
                    ctypes.windll.user32.keybd_event(0x10, 0, 2, 0)

            time.sleep(TYPING_CHAR_DELAY)


def execute_action(func_name: str, params: list) -> None:
    """Execute action with smooth, human-like behavior."""
    if func_name == "observe":
        print("  → Observing (no interaction)")
        return

    screen_w = ctypes.windll.user32.GetSystemMetrics(0)
    screen_h = ctypes.windll.user32.GetSystemMetrics(1)

    if func_name == "left_click" and len(params) == 2:
        target_x = int((params[0] / 1000.0) * screen_w)
        target_y = int((params[1] / 1000.0) * screen_h)

        print(f"  → Moving to ({target_x}, {target_y})...")
        smooth_move_to(target_x, target_y)
        time.sleep(CLICK_SETTLE_DELAY)

        print("  → Left clicking...")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)

    elif func_name == "right_click" and len(params) == 2:
        target_x = int((params[0] / 1000.0) * screen_w)
        target_y = int((params[1] / 1000.0) * screen_h)

        print(f"  → Moving to ({target_x}, {target_y})...")
        smooth_move_to(target_x, target_y)
        time.sleep(CLICK_SETTLE_DELAY)

        print("  → Right clicking...")
        ctypes.windll.user32.mouse_event(0x0008, 0, 0, 0, 0)
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0010, 0, 0, 0, 0)

    elif func_name == "double_left_click" and len(params) == 2:
        target_x = int((params[0] / 1000.0) * screen_w)
        target_y = int((params[1] / 1000.0) * screen_h)

        print(f"  → Moving to ({target_x}, {target_y})...")
        smooth_move_to(target_x, target_y)
        time.sleep(CLICK_SETTLE_DELAY)

        print("  → Double clicking...")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)
        time.sleep(0.08)
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)

    elif func_name == "drag" and len(params) == 4:
        x1 = int((params[0] / 1000.0) * screen_w)
        y1 = int((params[1] / 1000.0) * screen_h)
        x2 = int((params[2] / 1000.0) * screen_w)
        y2 = int((params[3] / 1000.0) * screen_h)

        print(f"  → Moving to drag start ({x1}, {y1})...")
        smooth_move_to(x1, y1)
        time.sleep(0.1)

        print("  → Grabbing...")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.1)

        print(f"  → Dragging to ({x2}, {y2})...")
        start_x, start_y = get_cursor_position()
        for i in range(MOVEMENT_STEPS + 1):
            t = i / MOVEMENT_STEPS
            t = t * t * (3 - 2 * t)

            x = int(start_x + (x2 - start_x) * t)
            y = int(start_y + (y2 - start_y) * t)

            ctypes.windll.user32.SetCursorPos(x, y)
            time.sleep(STEP_DELAY)

        print("  → Releasing...")
        time.sleep(0.1)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)

    elif func_name == "type" and len(params) == 1:
        type_text(params[0])


def _format_command(func_name: str, params: list) -> str:
    """Format a parsed command back into its function-call string."""
    if func_name == "observe":
        return "observe()"
    if func_name == "type":
        return f'type("{params[0]}")'
    return f"{func_name}({','.join(str(p) for p in params)})"


# ─── Main Loop ───────────────────────────────────────────────────────────────

def main() -> None:
    """Main automation loop with mode-aware cadence."""
    dump_dir = Path("dump") / datetime.now().strftime("run_%Y%m%d_%H%M%S")
    dump_dir.mkdir(parents=True, exist_ok=True)

    # Mutable agent state
    mode = MODE_EXECUTING
    goal = INITIAL_GOAL
    story = "FRANZ initialising. Observing the desktop for the first time."
    last_action = "init"

    iteration = 0

    print("\n" + "=" * 60)
    print("FRANZ — Narrative-Driven Autonomous Desktop Agent")
    print("=" * 60)
    print(f"Mode:        {mode}")
    print(f"Goal:        {goal}")
    print(f"Screenshots: {dump_dir}")
    print(f"Exec delay:  {EXECUTING_LOOP_DELAY}s  |  Monitor delay: {MONITORING_LOOP_DELAY}s")
    print("=" * 60 + "\n")

    while True:
        iteration += 1
        print(f"\n{'─' * 60}")
        print(f"Iteration {iteration}  |  Mode: {mode}")
        print(f"{'─' * 60}")
        print(f"Goal:  {goal}")
        print(f"Story: {story}")
        print(f"Last:  {last_action}")

        # ── Capture ──────────────────────────────────────────────────
        draw_func = create_visualization(last_action)
        img = screenshot.capture_screen_png(TARGET_WIDTH, TARGET_HEIGHT, draw_func=draw_func)

        timestamp = int(time.time() * 1000)
        (dump_dir / f"{timestamp}.png").write_bytes(img)

        # ── Infer ────────────────────────────────────────────────────
        print("Sending to VLM...")
        content = infer(img, mode, goal, story, last_action)
        print(f"VLM response:\n{content}\n")

        # ── Parse ────────────────────────────────────────────────────
        parsed = parse_response(content)

        # ── Update mode ──────────────────────────────────────────────
        new_mode = parsed["status"] or mode  # keep current if VLM didn't specify

        if new_mode != mode:
            print(f"  ◆ Mode transition: {mode} → {new_mode}")

        # When transitioning to monitoring, apply the monitoring goal
        # unless the VLM already provided a specific new goal
        if new_mode == MODE_MONITORING and mode == MODE_EXECUTING:
            if not parsed["goal"] or parsed["goal"].strip().lower() == goal.strip().lower():
                goal = MONITORING_GOAL
                print(f"  ◆ Goal set to monitoring default")

        mode = new_mode

        # ── Update goal (VLM-driven) ─────────────────────────────────
        if parsed["goal"] and parsed["goal"] != goal:
            goal = parsed["goal"]
            print(f"  ◆ Goal updated: {goal}")

        # ── Execute actions ──────────────────────────────────────────
        commands = parsed["commands"]

        if mode == MODE_MONITORING:
            # In monitoring mode, only allow observe() — block accidental interactions
            safe_commands = [(fn, p) for fn, p in commands if fn == "observe"]
            if len(safe_commands) < len(commands):
                print("  ⚠ Blocked non-observe actions in monitoring mode")
            commands = safe_commands

        if commands:
            # Only execute the first command (single-step commitment)
            func_name, params = commands[0]
            cmd_str = _format_command(func_name, params)
            print(f"  ▶ {cmd_str}")
            execute_action(func_name, params)
            last_action = cmd_str

            if len(commands) > 1:
                print(f"  ⚠ Ignored {len(commands) - 1} extra command(s) (single-step policy)")
        else:
            print("  ▶ No action (observing)")
            last_action = "observe()"

        # ── Update story ─────────────────────────────────────────────
        if parsed["story"]:
            story = parsed["story"]
            print(f"  ◆ Story: {story}")

        # Persist state to disk
        state = {
            "iteration": iteration,
            "mode": mode,
            "goal": goal,
            "story": story,
            "last_action": last_action,
        }
        (dump_dir / "state.json").write_text(
            json.dumps(state, indent=2, ensure_ascii=False), encoding="utf-8"
        )
        (dump_dir / "story.txt").write_text(story, encoding="utf-8")

        # ── Cadence control ──────────────────────────────────────────
        delay = MONITORING_LOOP_DELAY if mode == MODE_MONITORING else EXECUTING_LOOP_DELAY
        print(f"  ⏱ Next iteration in {delay}s")
        time.sleep(delay)


if __name__ == "__main__":
    main()
```

---

## What Changed and Why

### 1. **Mutable Goal (`goal` variable replaces hardcoded template)**

The goal is now a runtime variable that the VLM can rewrite. It starts as `INITIAL_GOAL` and transitions to `MONITORING_GOAL` when a task completes. The VLM can set it to anything — "Open Chrome and search for weather", "Close the error dialog", "Read the message in Notepad".

### 2. **Two Operational Modes with VLM-Controlled Transitions**

```
executing ──[VLM outputs STATUS: monitoring]──▶ monitoring
monitoring ──[VLM outputs STATUS: executing]──▶ executing
```

The VLM decides when a task is done (→ monitoring) and when a new task appears (→ executing). The code **never** hardcodes completion logic — it trusts the VLM's visual judgment.

### 3. **`observe()` Action**

A new no-op action that the VLM outputs when monitoring and finding nothing. This is important because it gives the model an explicit way to say "I looked, nothing to do" rather than being forced to pick a click/type action.

### 4. **Safety Guard in Monitoring Mode**

When in monitoring mode, the code **blocks all actions except `observe()`**. This prevents the agent from accidentally clicking or typing on the user's desktop while scanning. The VLM must explicitly transition to executing mode before it can interact.

### 5. **Adaptive Cadence**

- **Executing**: 0.5s between iterations (fast, responsive)
- **Monitoring**: 3.0s between iterations (slow, resource-efficient)

A monitoring agent doesn't need to poll at 2Hz. This saves both compute and inference costs.

### 6. **Single-Step Enforcement**

Even if the VLM outputs multiple function calls, only the **first** is executed. This was loosely enforced before; now it's explicit with a logged warning.

### 7. **State Persistence**

Each iteration writes a `state.json` with mode, goal, story, and last action. This serves as both a debug log and a potential recovery mechanism.

### 8. **Structured Output Protocol**

The VLM's output format is now explicitly defined:

```
STATUS: executing | monitoring
GOAL: <text>
ACTION: <function call>
STORY: <3 sentences>
```

The parser uses dedicated regexes for each section rather than trying to heuristically separate code from narrative.

### The Screen-as-Communication-Channel Pattern

The monitoring mode enables a powerful interaction pattern:

```
User opens Notepad, types:          "Franz, please open Calculator"
Franz (monitoring) sees this    →   Detects message via VLM visual understanding
Franz transitions to executing  →   GOAL: "Open Calculator application"
Franz completes task            →   Calculator is open
Franz transitions to monitoring →   Scanning screen again...
```

No API. No IPC. No file watching. The screen IS the interface. The VLM's ability to read arbitrary text in screenshots makes this work without any OCR pipeline — the visual understanding is built into the model.