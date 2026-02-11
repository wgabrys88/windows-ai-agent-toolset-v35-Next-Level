

## Mental Simulation with a 2B VLM

Let me think through this very carefully. A 2B parameter vision-language model has severe limitations:

- **Short attention span**: complex multi-section formats get jumbled
- **Instruction drift**: if the prompt is verbose or has many rules, the model latches onto fragments
- **Format fragility**: special characters like `|`, `◆`, `▶`, backtick fences within fences, piped alternatives — all confuse small models
- **Repetition loops**: the model will copy patterns it sees in the prompt verbatim
- **Overloaded output schemas**: asking for 4 distinct labeled sections (`STATUS:`, `GOAL:`, `ACTION:`, `STORY:`) is a lot for 2B — it will forget one, merge two, or hallucinate extra sections

Let me simulate three multi-turn scenarios.

---

### Scenario 1: Initial Cat Drawing Task (First 3 Iterations)

**Iteration 1** — Desktop is visible, Paint is not open.

The prompt sends:
```
Current mode: executing
Current goal: Draw a cat in MS Paint, then save it on the desktop as "cat.jpg".

Story so far:
FRANZ initialising. Observing the desktop for the first time.

Last action performed: init
```

**Likely 2B output:**
```
STATUS: executing
GOAL: Draw a cat in MS Paint, then save it on the desktop as "cat.jpg".
ACTION: ```python
left_click(500, 980)
```
STORY: I can see the desktop with the taskbar at the bottom. I need to click on the Start menu to find MS Paint. After clicking, I expect the Start menu to open.
```

**Problems detected:**
1. The model wrapped ACTION in a code fence even though the prompt said `exactly one function call`. The parser handles this — `_FUNC_CALL_RE` will find `left_click(500, 980)` inside the fence. ✅
2. The model repeated the GOAL verbatim. The parser will see the goal hasn't changed and keep it. ✅
3. But wait — the model put triple backticks *inside* its response while our system prompt also uses backtick formatting. A 2B model might get confused about when to open/close code blocks. **This is a risk.**

**Iteration 2** — Start menu is open.

**Likely 2B output:**
```
STATUS: executing
GOAL: Draw a cat in MS Paint, then save it on the desktop as "cat.jpg".
ACTION: type("Paint")
STORY: The Start menu is open. I need to type Paint to search for the application. I expect the search results to show MS Paint.
```

This works. ✅ But notice the model is outputting `ACTION:` as a label before the function. Our parser uses `_FUNC_CALL_RE` which scans the whole content, so it finds `type("Paint")` fine. ✅

**Iteration 3** — Paint appeared in search results.

**Likely 2B output (PROBLEM):**
```
STATUS executing
GOAL Draw a cat in MS Paint
ACTION left_click(450 320)
STORY I see Paint in the search results. I will click on it to open the application.
```

**Problems detected:**
1. **Missing colons** after section labels. A 2B model is inconsistent with punctuation. Our regex `STATUS\s*:\s*(executing|monitoring)` requires a colon. **This will fail to parse status.** ❌
2. **Missing comma** in `left_click(450 320)`. Our `_parse_args` uses `re.findall(r"-?\d+", ...)` which finds numbers regardless of delimiter. ✅ This works.
3. **GOAL truncated** — the model shortened it. Our parser picks it up fine. ✅

**Fix needed:** Make section-label regexes tolerant of missing colons.

---

### Scenario 2: Task Completion → Monitoring Transition

**Setup:** The cat.jpg has been saved. Desktop shows the file.

**Iteration N** — cat.jpg visible on desktop, Paint is closed.

**Likely 2B output:**
```
STATUS: monitoring
GOAL: The cat drawing has been saved. I will now watch the screen for new tasks.
ACTION: observe()
STORY: The task is complete. I saved cat.jpg to the desktop. I will now monitor the screen for any new instructions or tasks.
```

This is ideal. But more realistically from a 2B model:

```
The task is complete! The cat.jpg file is now saved on the desktop.

STATUS: monitoring
GOAL: Watch for new tasks
ACTION: observe()
STORY: I have completed the cat drawing task. The file cat.jpg is visible on the desktop. I will now scan for new instructions.
```

**Problems detected:**
1. Extra narrative text BEFORE the structured output. Our parser scans for section keywords anywhere in the text, so this works. ✅
2. But what if the model does this?

```
I see the cat.jpg is saved. STATUS monitoring. GOAL none. observe().
The story continues as I watch the desktop for new tasks from the user.
```

3. Now the model has **inlined everything on one jumbled line.** `_STATUS_RE` would match `STATUS monitoring` only if we make the colon optional. `observe()` would be found by `_FUNC_CALL_RE`. GOAL would fail because `none` is mixed into the sentence. **Partial failure.** ❌

**Fix needed:** Simpler output format. Fewer sections. Less structure for the model to break.

---

### Scenario 3: Monitoring → User Message Detection → New Task

**Setup:** Franz is monitoring. User opens Notepad and types "Franz please open Chrome."

**Iteration M** — Notepad visible with the message.

**Likely 2B output:**
```
STATUS: executing
GOAL: Open Chrome browser as requested by the user
ACTION: left_click(500, 980)
STORY: I can see a message in Notepad asking me to open Chrome. I will click the taskbar to find Chrome. The user is communicating through the screen.
```

This works perfectly. ✅ But a 2B model might also do:

```
I see a Notepad window with text "Franz please open Chrome". The user wants me to open Chrome.

left_click(500, 980)

I will click on the taskbar to search for Chrome. The status is executing and my goal is to open Chrome for the user.
```

**Problems detected:**
1. **No STATUS/GOAL/STORY labels at all.** The model just wrote free-form text with a function call in the middle. ❌
2. `_FUNC_CALL_RE` finds `left_click(500, 980)`. ✅
3. `_STATUS_RE` finds nothing → mode stays as `monitoring` → **the safety guard blocks the click!** ❌❌ The agent is stuck in monitoring even though it correctly identified a task.

**This is a critical failure mode.** The monitoring safety guard blocks non-observe actions, but the model failed to emit the STATUS label.

**Fix needed:** 
- If the model outputs an action other than `observe()` while technically in monitoring mode, that itself IS evidence of a mode transition. The code should infer the transition rather than blocking.
- Alternatively, simplify the format so much that the model can't fail to emit mode signals.

---

## Key Findings from Simulation

| Problem | Severity | Solution |
|---|---|---|
| Model omits colons after labels | High | Make colons optional in regexes |
| Model omits labels entirely | Critical | Infer mode from action type (action ≠ observe → must be executing) |
| Model wraps action in code fences inconsistently | Low | Already handled by regex scanning full content |
| Multi-section structured output is too complex for 2B | High | Reduce to fewer sections, simpler format |
| Special characters in prompt (pipes `\|`, bullets, backticks) confuse model | Medium | Use only plain ASCII, simple dashes and newlines |
| Monitoring guard blocks legitimate task pickup | Critical | Use action-type inference as fallback for missing STATUS |
| Model repeats/truncates GOAL unpredictably | Medium | Only update goal when model explicitly changes it; use fuzzy matching |
| Model merges STORY and GOAL into one blob | Medium | Make STORY the catch-all — if only one block of text is found, treat it as story |
| Verbose system prompt causes drift | High | Shorten drastically; use examples not rules |

## Design Decisions for 2B

1. **Drop the 4-section structured format.** Instead: the model outputs ONE function call (or `observe()`) plus free-form text. We extract the function call with regex. Everything else is the story. Mode is inferred from behavior.

2. **Remove explicit STATUS/GOAL labels from model output.** The model doesn't need to write `STATUS: executing`. Instead:
   - If the model outputs `observe()`, it's monitoring.
   - If the model outputs any other action, it's executing.
   - The GOAL is managed by the code based on simple heuristics + story content.

3. **Goal modification via a simpler mechanism:** The model can write `NEW GOAL: ...` in its story text. A simple regex catches this. If absent, goal stays.

4. **Simplify the system prompt to under 200 words.** No markdown tables, no bullet lists with sub-items, no pipes.

5. **Use a single concrete example** in the system prompt showing the expected output format.

```python
# main.py
"""FRANZ - AI-driven Windows automation agent with smooth, human-like interactions.

Operates in a continuous perception-action loop using a small VLM.
Mode (executing vs monitoring) is inferred from the model's chosen action.
Goal can be updated by the model writing NEW GOAL in its response.
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

# --- Configuration -----------------------------------------------------------

API: Final = "http://localhost:1234/v1/chat/completions"
MODEL: Final = "qwen3-vl-2b-instruct-1m"
TARGET_WIDTH: Final = 1536
TARGET_HEIGHT: Final = 864
SAMPLING: Final = {"temperature": 0.7, "top_p": 0.9, "max_tokens": 800}
ENABLE_VISUAL_FEEDBACK: Final = True

# --- Timing ------------------------------------------------------------------

MOVEMENT_STEPS: Final = 20
STEP_DELAY: Final = 0.01
CLICK_SETTLE_DELAY: Final = 0.15
TYPING_CHAR_DELAY: Final = 0.08
TYPING_WORD_DELAY: Final = 0.15

# --- Cadence -----------------------------------------------------------------

EXECUTING_DELAY: Final = 0.5
MONITORING_DELAY: Final = 3.0

# --- Mode constants ----------------------------------------------------------

MODE_EXECUTING: Final = "executing"
MODE_MONITORING: Final = "monitoring"

MONITORING_GOAL: Final = (
    "Scan the screen for any message to Franz, unfinished work, "
    "open dialogs, errors, or anything that needs attention."
)

INITIAL_GOAL: Final = (
    'Draw a cat in MS Paint, then save it on the desktop as "cat.jpg".'
)

# --- Prompts -----------------------------------------------------------------

SYSTEM_PROMPT: Final = """\
You are Franz, a computer assistant that looks at the screen and performs one action at a time.

You can use these functions (coordinates are 0 to 1000):
  left_click(x, y)
  right_click(x, y)
  double_left_click(x, y)
  drag(x1, y1, x2, y2)
  type("text")
  observe()

Use observe() when there is nothing to do and you are just watching.

Your response must contain:
1. One function call on its own line.
2. A short story (2 to 3 sentences) about what you see, what you are doing, and what comes next.
3. If the current goal is finished, write DONE on its own line.
4. If you want to set a new goal, write NEW GOAL followed by the goal.

Example response:
left_click(450, 320)
I see MS Paint in the search results. I am clicking it to open the application. Next I will start drawing the cat.

Another example when a task is finished:
observe()
The file cat.jpg is now visible on the desktop. The drawing task is complete.
DONE

Another example when spotting a new task while watching:
left_click(500, 980)
I see a note in Notepad asking me to open Chrome. I am clicking the taskbar to begin. Next I will search for Chrome.
NEW GOAL Open the Chrome browser as requested by the user.
"""

USER_PROMPT_TEMPLATE: Final = """\
Goal: {goal}

Story so far: {story}

Last action: {last}

Look at the screenshot. Respond with one function call and a short story.\
"""


# --- Inference ---------------------------------------------------------------

def infer(png_data: bytes, goal: str, story: str, last: str) -> str:
    """Send one stateless inference request to the VLM."""
    user_text = USER_PROMPT_TEMPLATE.format(goal=goal, story=story, last=last)

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


# --- Parsing -----------------------------------------------------------------

_FUNC_RE = re.compile(
    r"\b(left_click|right_click|double_left_click|drag|type|observe)\s*\(([^)]*)\)"
)

_NEW_GOAL_RE = re.compile(r"(?:^|\n)\s*NEW GOAL[:\s]*(.+)", re.IGNORECASE)
_DONE_RE = re.compile(r"(?:^|\n)\s*DONE\s*$", re.IGNORECASE | re.MULTILINE)


def parse_response(content: str) -> dict:
    """Parse VLM response.

    Returns dict with:
        command:  (func_name, params) or None
        story:    str
        done:     bool
        new_goal: str or None
    """
    result = {
        "command": None,
        "story": "",
        "done": False,
        "new_goal": None,
    }

    # Find the first function call
    match = _FUNC_RE.search(content)
    if match:
        func_name = match.group(1)
        raw_args = match.group(2).strip()

        if func_name == "observe":
            result["command"] = ("observe", [])
        else:
            params = _parse_args(func_name, raw_args)
            if params is not None:
                result["command"] = (func_name, params)

    # Check for DONE
    if _DONE_RE.search(content):
        result["done"] = True

    # Check for NEW GOAL
    goal_match = _NEW_GOAL_RE.search(content)
    if goal_match:
        new_goal = goal_match.group(1).strip()
        # Clean up: remove trailing DONE or function calls from the goal text
        new_goal = _FUNC_RE.sub("", new_goal).strip()
        new_goal = _DONE_RE.sub("", new_goal).strip()
        if new_goal:
            result["new_goal"] = new_goal

    # Story: everything that is not the function call, DONE, or NEW GOAL line
    story_lines = []
    for line in content.splitlines():
        stripped = line.strip()
        if not stripped:
            continue
        if _FUNC_RE.search(stripped):
            continue
        if re.match(r"^\s*DONE\s*$", stripped, re.IGNORECASE):
            continue
        if re.match(r"^\s*NEW GOAL", stripped, re.IGNORECASE):
            continue
        # Skip lines that are just code fence markers
        if stripped.startswith("```"):
            continue
        story_lines.append(stripped)

    result["story"] = " ".join(story_lines).strip()

    return result


def _parse_args(func_name: str, raw_args: str) -> list | None:
    """Parse function arguments."""
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


# --- Visualisation -----------------------------------------------------------

def create_visualization(last_action: str) -> callable:
    """Create drawing callback for screenshot annotation."""
    def draw_annotations(rgba: bytes, width: int, height: int) -> bytes:
        if not ENABLE_VISUAL_FEEDBACK or last_action in ("init", "observe()"):
            return rgba

        match = _FUNC_RE.search(last_action)
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


# --- Input Execution ---------------------------------------------------------

def get_cursor_position() -> tuple[int, int]:
    """Get current cursor position."""
    point = ctypes.wintypes.POINT()
    ctypes.windll.user32.GetCursorPos(ctypes.byref(point))
    return point.x, point.y


def smooth_move_to(target_x: int, target_y: int, steps: int = MOVEMENT_STEPS, delay: float = STEP_DELAY) -> None:
    """Smoothly move cursor using smoothstep interpolation."""
    start_x, start_y = get_cursor_position()

    for i in range(steps + 1):
        t = i / steps
        t = t * t * (3 - 2 * t)

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
    """Type text with human-like timing."""
    print(f'  typing: "{text}"')

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
    """Execute a single action with smooth human-like behavior."""
    if func_name == "observe":
        print("  observing (no interaction)")
        return

    screen_w = ctypes.windll.user32.GetSystemMetrics(0)
    screen_h = ctypes.windll.user32.GetSystemMetrics(1)

    if func_name == "left_click" and len(params) == 2:
        tx = int((params[0] / 1000.0) * screen_w)
        ty = int((params[1] / 1000.0) * screen_h)
        print(f"  moving to ({tx}, {ty})...")
        smooth_move_to(tx, ty)
        time.sleep(CLICK_SETTLE_DELAY)
        print("  left click")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)

    elif func_name == "right_click" and len(params) == 2:
        tx = int((params[0] / 1000.0) * screen_w)
        ty = int((params[1] / 1000.0) * screen_h)
        print(f"  moving to ({tx}, {ty})...")
        smooth_move_to(tx, ty)
        time.sleep(CLICK_SETTLE_DELAY)
        print("  right click")
        ctypes.windll.user32.mouse_event(0x0008, 0, 0, 0, 0)
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0010, 0, 0, 0, 0)

    elif func_name == "double_left_click" and len(params) == 2:
        tx = int((params[0] / 1000.0) * screen_w)
        ty = int((params[1] / 1000.0) * screen_h)
        print(f"  moving to ({tx}, {ty})...")
        smooth_move_to(tx, ty)
        time.sleep(CLICK_SETTLE_DELAY)
        print("  double click")
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

        print(f"  moving to drag start ({x1}, {y1})...")
        smooth_move_to(x1, y1)
        time.sleep(0.1)
        print("  grabbing...")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.1)

        print(f"  dragging to ({x2}, {y2})...")
        sx, sy = get_cursor_position()
        for i in range(MOVEMENT_STEPS + 1):
            t = i / MOVEMENT_STEPS
            t = t * t * (3 - 2 * t)
            x = int(sx + (x2 - sx) * t)
            y = int(sy + (y2 - sy) * t)
            ctypes.windll.user32.SetCursorPos(x, y)
            time.sleep(STEP_DELAY)

        print("  releasing...")
        time.sleep(0.1)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)

    elif func_name == "type" and len(params) == 1:
        type_text(params[0])


def format_command(func_name: str, params: list) -> str:
    """Format a command as a function call string for logging."""
    if func_name == "observe":
        return "observe()"
    if func_name == "type":
        return f'type("{params[0]}")'
    return f"{func_name}({','.join(str(p) for p in params)})"


# --- Main Loop ---------------------------------------------------------------

def main() -> None:
    """Main loop with behavior-inferred mode transitions."""
    dump_dir = Path("dump") / datetime.now().strftime("run_%Y%m%d_%H%M%S")
    dump_dir.mkdir(parents=True, exist_ok=True)

    # Agent state
    mode = MODE_EXECUTING
    goal = INITIAL_GOAL
    story = "Franz starting up. Looking at the desktop for the first time."
    last_action = "init"
    idle_count = 0

    iteration = 0

    print()
    print("=" * 60)
    print("FRANZ - Narrative-Driven Autonomous Desktop Agent")
    print("=" * 60)
    print(f"  Mode:      {mode}")
    print(f"  Goal:      {goal}")
    print(f"  Dump:      {dump_dir}")
    print("=" * 60)
    print()

    while True:
        iteration += 1
        print(f"\n--- iteration {iteration} | mode: {mode} ---")
        print(f"  goal:  {goal}")
        print(f"  story: {story}")
        print(f"  last:  {last_action}")

        # -- Capture --
        draw_func = create_visualization(last_action)
        img = screenshot.capture_screen_png(TARGET_WIDTH, TARGET_HEIGHT, draw_func=draw_func)

        ts = int(time.time() * 1000)
        (dump_dir / f"{ts}.png").write_bytes(img)

        # -- Infer --
        print("  sending to VLM...")
        content = infer(img, goal, story, last_action)
        print(f"  VLM response:\n{content}\n")

        # -- Parse --
        parsed = parse_response(content)

        # -- Determine action --
        command = parsed["command"]
        is_observe = command is None or command[0] == "observe"

        # -- Infer mode from behavior --
        if parsed["done"]:
            # Model explicitly said DONE
            if mode == MODE_EXECUTING:
                print("  >> task marked DONE, switching to monitoring")
                mode = MODE_MONITORING
                goal = MONITORING_GOAL
                idle_count = 0

        if parsed["new_goal"]:
            # Model wants a new goal - switch to executing
            goal = parsed["new_goal"]
            mode = MODE_EXECUTING
            idle_count = 0
            print(f"  >> new goal: {goal}")

        if mode == MODE_MONITORING:
            if not is_observe:
                # Model chose an action while monitoring -> it found a task
                mode = MODE_EXECUTING
                print("  >> action detected in monitoring mode, switching to executing")
                if not parsed["new_goal"]:
                    # Model didn't write NEW GOAL but is acting - use story as context
                    goal = parsed["story"] if parsed["story"] else "Responding to something on screen."
                    print(f"  >> inferred goal from story: {goal}")

        # -- Execute --
        if command is not None:
            func_name, params = command
            cmd_str = format_command(func_name, params)
            print(f"  >> {cmd_str}")
            execute_action(func_name, params)
            last_action = cmd_str
        else:
            print("  >> no valid action parsed, observing")
            last_action = "observe()"

        # -- Track idle in monitoring --
        if mode == MODE_MONITORING and is_observe:
            idle_count += 1
        else:
            idle_count = 0

        # -- Update story --
        if parsed["story"]:
            story = parsed["story"]
            print(f"  >> story: {story}")

        # -- Persist state --
        state = {
            "iteration": iteration,
            "mode": mode,
            "goal": goal,
            "story": story,
            "last_action": last_action,
            "idle_count": idle_count,
        }
        (dump_dir / "state.json").write_text(
            json.dumps(state, indent=2, ensure_ascii=False), encoding="utf-8"
        )
        (dump_dir / "story.txt").write_text(story, encoding="utf-8")

        # -- Cadence --
        delay = MONITORING_DELAY if mode == MODE_MONITORING else EXECUTING_DELAY
        print(f"  >> next in {delay}s (idle: {idle_count})")
        time.sleep(delay)


if __name__ == "__main__":
    main()
```

---

## Summary of Changes vs Previous Version

| Change | Rationale |
|---|---|
| **Replaced 4-section structured output with freeform** | A 2B model cannot reliably produce `STATUS:`, `GOAL:`, `ACTION:`, `STORY:` labels consistently. Now: one function call + freeform text + optional `DONE` + optional `NEW GOAL` |
| **Mode inferred from behavior, not declared** | If model outputs `observe()` or nothing, it's monitoring. If it outputs an action, it's executing. No reliance on the model labeling its own mode. |
| **`DONE` keyword instead of `STATUS: monitoring`** | A single uppercase word on its own line is far easier for a 2B model to produce than a structured label-value pair |
| **`NEW GOAL` keyword instead of `GOAL:` section** | Only appears when the goal actually changes. Model doesn't need to repeat the current goal every time (which it would truncate/modify unpredictably) |
| **System prompt reduced from ~350 words to ~180 words** | Less instruction drift, more room for the model to think |
| **Three concrete examples in system prompt** | Small models learn from examples better than from rules. Shows: normal action, task completion with DONE, and task detection with NEW GOAL |
| **No special characters in prompts** | Removed all `|`, `◆`, `▶`, `⚠`, `⏱` from prompt text. Plain ASCII only. |
| **Removed monitoring safety guard that blocks actions** | The guard caused a deadlock when the model found a task but didn't emit STATUS. Now: if the model acts during monitoring, that IS the mode switch. |
| **Story extracted as everything that isn't a function call, DONE, or NEW GOAL** | Robust: even if model mixes structure and prose, the story captures the narrative intent |
| **Single command enforcement is now structural** | `parse_response` returns one command (the first match), not a list. Cannot execute multiple. |
| **`idle_count` tracking added** | Counts consecutive observe-only iterations. Can be used for future features (e.g., increasing monitoring delay logarithmically). |
| **Removed `mode` from VLM prompt** | The model doesn't need to know what "mode" it's in — that's an internal bookkeeping concept. It just gets the goal and story. If the goal says "scan for tasks", it scans. If it says "draw a cat", it draws. |