

## Deep Reasoning: Multi-Action Output for a 2B Model

Let me think about this very carefully. The challenge is:

1. A 2B model can barely produce one structured action reliably
2. We want it to produce *multiple* actions when appropriate
3. We must not constrain it in code — if it outputs 5 actions, we execute 5
4. But it needs to understand *when* multiple actions make sense vs. when single-step is safer
5. The linguistic framing must be natural enough for a 2B model to internalize

### When Are Multiple Actions Appropriate?

Think about human computer usage — there are moments of **high certainty** and moments of **low certainty**:

**High certainty (multi-action safe):**
- Typing a filename in a save dialog (you know the dialog is there, you type, you click save)
- A sequence like: click a menu → click a known sub-item (if you've already seen the menu contents)
- Closing a dialog and clicking something else visible underneath
- Drawing multiple strokes in Paint (each drag is predictable)

**Low certainty (single-action required):**
- Clicking something and waiting to see what opens
- First time interacting with an unfamiliar UI state
- After an error message appears
- When you're not sure what's on screen

### The Linguistic Key for a 2B Model

I can't use abstract concepts like "confidence thresholds." A 2B model needs **concrete, physical metaphors**:

- "If you can **see everything you need** on screen right now to do 2 or 3 steps, write all of them."
- "If the next step will **change what's on screen** (opening a menu, switching windows), stop after that one step and wait to see the result."
- "Think of it like walking: on a clear straight path, you can take several steps. At a corner, stop and look."

The walking metaphor is good for small models — it's grounded, physical, universally understood.

### How to Structure Multi-Action Output

The model needs a format that's natural for it to produce multiple actions. Options:

1. **One per line** — simplest: `left_click(500,300)` then on next line `type("hello")`
2. Numbered — `1. left_click(500,300)` then `2. type("hello")` — adds complexity
3. Code block with multiple lines — model already tends to do this

Option 1 is best for a 2B model. Our `_FUNC_RE.finditer()` already finds ALL function calls in the response. We just need to execute them all in order.

### What Changes in the Code

- **`parse_response`** already uses `finditer` — but the previous version only returned the first match. Change it to return ALL matches as an ordered list.
- **`execute_action` loop** — iterate through all commands, with a brief delay between them.
- **No cap, no filter** — if the model outputs 7 actions, we run 7. The experiment is to trust the model's judgment as guided by the prompt.
- **System prompt** — add the walking metaphor and examples of multi-action output.
- **Visualisation** — show ALL executed actions on the next screenshot, not just the last one. This gives the VLM visual feedback about the whole batch.

### Visualisation of Multiple Actions

This is important for the feedback loop. If we executed 3 clicks, the next screenshot should show crosshairs at all 3 positions, numbered or color-coded. The model needs to see what its batch *did*.

I'll use a numbering approach: store all executed actions as a list, and the visualization function draws all of them with sequential numbering.

### Mental Simulation: Multi-Action with 2B Model

**Scenario: Save dialog is open, filename field is focused**

Prompt context says: Goal is to save as cat.jpg on desktop.

**Model output (ideal multi-action):**
```
type("cat.jpg")
left_click(750, 580)
I see the Save As dialog with the filename field ready. I am typing the filename and clicking Save in one smooth motion because both targets are visible right now.
```

This is exactly 2 actions that make sense together. The model can see the text field and the Save button simultaneously.

**Model output (appropriate single action):**
```
left_click(200, 400)
I see the File menu. I am clicking it to open the save options. The menu will change the screen so I need to wait and see what appears.
```

Here the model correctly limits itself to one action because clicking File will change what's visible.

**Model output (overly ambitious, but we allow it):**
```
left_click(50, 30)
left_click(200, 350)
type("cat")
left_click(600, 500)
I see the Paint window. I am clicking File, then Save As, typing the name, and clicking save all at once.
```

This is risky — the model is assuming menu positions before the menu opens. But we don't block it. If it fails, the next screenshot shows the actual state, and the narrative-based recovery kicks in. This is the experimental nature of the feature.

### The `last_action` Tracking for Multi-Action

Instead of storing just one string, I'll store the full list of executed command strings. The user prompt will show all of them so the model knows what its last batch was. The visualization will draw all of them.

Let me write the final code now.

```python
# main.py
"""FRANZ - AI-driven Windows automation agent with smooth, human-like interactions.

Operates in a continuous perception-action loop using a small VLM.
Mode (executing vs monitoring) is inferred from the model's chosen actions.
Goal can be updated by the model writing NEW GOAL in its response.
Supports multi-action output when the model judges the screen state is predictable.
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
BETWEEN_ACTIONS_DELAY: Final = 0.3  # pause between actions in a multi-action batch

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
You are Franz, a computer assistant that looks at the screen and performs actions.

You can use these functions (coordinates are 0 to 1000):
  left_click(x, y)
  right_click(x, y)
  double_left_click(x, y)
  drag(x1, y1, x2, y2)
  type("text")
  observe()

Use observe() when there is nothing to do and you are just watching.

How many actions to write:
Think of walking down a path. On a clear straight road where you can see \
everything ahead, you can take several steps at once. But when you reach a \
corner or a door, you stop and take just one step, then look again.

Same rule here. If everything you need for the next 2 or 3 steps is already \
visible on screen right now, write all those function calls, one per line. \
But if your action will change the screen (opening a menu, switching a window, \
clicking a button that opens a dialog), write only that one action and wait \
for the next screenshot.

Never guess what will appear after a screen change. Only act on what you see.

Your response must contain:
1. One or more function calls, each on its own line.
2. A short story (2 to 3 sentences) about what you see, what you are doing, \
and what comes next.
3. If the current goal is finished, write DONE on its own line.
4. If you want to set a new goal, write NEW GOAL followed by the goal text.

Example with one action (screen will change after this click):
left_click(50, 30)
I see the Paint window with a blank canvas. I am clicking File to open the \
menu. I need to wait and see the menu options before choosing one.

Example with multiple actions (everything needed is visible right now):
left_click(350, 250)
type("cat.jpg")
left_click(700, 530)
The Save As dialog is open and I can see the filename field and the Save \
button. I am clicking the field, typing the filename, and clicking Save in \
one smooth sequence because all three targets are visible on screen.

Example when a task is finished:
observe()
The file cat.jpg is now visible on the desktop. The drawing task is complete.
DONE

Example when spotting a new task while watching:
left_click(500, 980)
I see a note in Notepad asking me to open Chrome. I am clicking the taskbar \
to begin searching. Next I will type Chrome in the search box.
NEW GOAL Open the Chrome browser as requested by the user.
"""

USER_PROMPT_TEMPLATE: Final = """\
Goal: {goal}

Story so far: {story}

Last actions: {last}

Look at the screenshot. Respond with your actions and a short story.\
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
        commands:  list of (func_name, params) in order of appearance
        story:     str
        done:      bool
        new_goal:  str or None
    """
    result = {
        "commands": [],
        "story": "",
        "done": False,
        "new_goal": None,
    }

    # Find all function calls in order of appearance
    for match in _FUNC_RE.finditer(content):
        func_name = match.group(1)
        raw_args = match.group(2).strip()

        if func_name == "observe":
            result["commands"].append(("observe", []))
            continue

        params = _parse_args(func_name, raw_args)
        if params is not None:
            result["commands"].append((func_name, params))

    # Check for DONE
    if _DONE_RE.search(content):
        result["done"] = True

    # Check for NEW GOAL
    goal_match = _NEW_GOAL_RE.search(content)
    if goal_match:
        new_goal = goal_match.group(1).strip()
        new_goal = _FUNC_RE.sub("", new_goal).strip()
        new_goal = _DONE_RE.sub("", new_goal).strip()
        if new_goal:
            result["new_goal"] = new_goal

    # Story: everything that is not a function call, DONE, or NEW GOAL
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

def create_visualization(last_actions: list[str]) -> callable:
    """Create drawing callback that annotates all actions from the last batch.

    Args:
        last_actions: list of function-call strings from the previous iteration
    """
    def draw_annotations(rgba: bytes, width: int, height: int) -> bytes:
        if not ENABLE_VISUAL_FEEDBACK:
            return rgba

        action_index = 0
        for action_str in last_actions:
            if action_str in ("init", "observe()"):
                continue

            match = _FUNC_RE.search(action_str)
            if not match:
                continue

            func_name = match.group(1)
            raw_args = match.group(2).strip()
            params = _parse_args(func_name, raw_args)
            if params is None:
                continue

            action_index += 1

            # Use different colors for sequential actions in a batch
            colors = [
                (drawing.RED, drawing.GREEN),
                (drawing.BLUE, drawing.YELLOW),
                (drawing.GREEN, drawing.RED),
                (drawing.YELLOW, drawing.BLUE),
            ]
            primary, secondary = colors[(action_index - 1) % len(colors)]

            if func_name in ("left_click", "double_left_click") and len(params) == 2:
                x = drawing.normalize_coord(params[0], width)
                y = drawing.normalize_coord(params[1], height)
                rgba = drawing.draw_crosshair(
                    rgba, width, height, x, y,
                    size=25, color=primary, thickness=3
                )
                rgba = drawing.draw_circle(
                    rgba, width, height, x, y,
                    radius=40, color=secondary, filled=False
                )
                # Draw action number
                rgba = _draw_action_number(
                    rgba, width, height, x + 30, y - 30, action_index, primary
                )

            elif func_name == "right_click" and len(params) == 2:
                x = drawing.normalize_coord(params[0], width)
                y = drawing.normalize_coord(params[1], height)
                rgba = drawing.draw_crosshair(
                    rgba, width, height, x, y,
                    size=25, color=drawing.BLUE, thickness=3
                )
                rgba = drawing.draw_circle(
                    rgba, width, height, x, y,
                    radius=40, color=drawing.YELLOW, filled=False
                )
                rgba = _draw_action_number(
                    rgba, width, height, x + 30, y - 30, action_index, drawing.BLUE
                )

            elif func_name == "drag" and len(params) == 4:
                x1 = drawing.normalize_coord(params[0], width)
                y1 = drawing.normalize_coord(params[1], height)
                x2 = drawing.normalize_coord(params[2], width)
                y2 = drawing.normalize_coord(params[3], height)
                rgba = drawing.draw_arrow(
                    rgba, width, height, x1, y1, x2, y2,
                    color=primary, thickness=4
                )
                rgba = drawing.draw_circle(
                    rgba, width, height, x1, y1,
                    radius=15, color=secondary, filled=True
                )
                rgba = drawing.draw_circle(
                    rgba, width, height, x2, y2,
                    radius=15, color=primary, filled=True
                )
                rgba = _draw_action_number(
                    rgba, width, height, x1 + 20, y1 - 20, action_index, primary
                )

        return rgba

    return draw_annotations


# Simple digit rendering for action numbers (5x7 pixel font)
_DIGITS: Final = {
    1: [
        "  #  ",
        " ##  ",
        "  #  ",
        "  #  ",
        "  #  ",
        "  #  ",
        " ### ",
    ],
    2: [
        " ### ",
        "#   #",
        "    #",
        "  ## ",
        " #   ",
        "#    ",
        "#####",
    ],
    3: [
        " ### ",
        "#   #",
        "    #",
        "  ## ",
        "    #",
        "#   #",
        " ### ",
    ],
    4: [
        "   # ",
        "  ## ",
        " # # ",
        "#  # ",
        "#####",
        "   # ",
        "   # ",
    ],
    5: [
        "#####",
        "#    ",
        "#### ",
        "    #",
        "    #",
        "#   #",
        " ### ",
    ],
    6: [
        " ### ",
        "#    ",
        "#    ",
        "#### ",
        "#   #",
        "#   #",
        " ### ",
    ],
    7: [
        "#####",
        "    #",
        "   # ",
        "  #  ",
        "  #  ",
        "  #  ",
        "  #  ",
    ],
    8: [
        " ### ",
        "#   #",
        "#   #",
        " ### ",
        "#   #",
        "#   #",
        " ### ",
    ],
    9: [
        " ### ",
        "#   #",
        "#   #",
        " ####",
        "    #",
        "    #",
        " ### ",
    ],
    0: [
        " ### ",
        "#   #",
        "#   #",
        "#   #",
        "#   #",
        "#   #",
        " ### ",
    ],
}


def _draw_action_number(
    rgba: bytes, width: int, height: int,
    x: int, y: int, number: int,
    color: tuple[int, int, int, int]
) -> bytes:
    """Draw a small action number at the given position."""
    data = bytearray(rgba)
    r, g, b, a = color

    # Draw each digit
    digit_offset = 0
    for ch in str(number):
        glyph = _DIGITS.get(int(ch))
        if glyph is None:
            continue
        for row_idx, row in enumerate(glyph):
            for col_idx, pixel in enumerate(row):
                if pixel == "#":
                    # Scale up 2x for visibility
                    for sy in range(2):
                        for sx in range(2):
                            px = x + digit_offset + col_idx * 2 + sx
                            py = y + row_idx * 2 + sy
                            if 0 <= px < width and 0 <= py < height:
                                idx = (py * width + px) * 4
                                data[idx:idx + 4] = bytes([r, g, b, a])
        digit_offset += 12  # 5 cols * 2 scale + 2 spacing

    return bytes(data)


# --- Input Execution ---------------------------------------------------------

def get_cursor_position() -> tuple[int, int]:
    """Get current cursor position."""
    point = ctypes.wintypes.POINT()
    ctypes.windll.user32.GetCursorPos(ctypes.byref(point))
    return point.x, point.y


def smooth_move_to(
    target_x: int, target_y: int,
    steps: int = MOVEMENT_STEPS, delay: float = STEP_DELAY
) -> None:
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
    print(f'    typing: "{text}"')

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
        print("    observing (no interaction)")
        return

    screen_w = ctypes.windll.user32.GetSystemMetrics(0)
    screen_h = ctypes.windll.user32.GetSystemMetrics(1)

    if func_name == "left_click" and len(params) == 2:
        tx = int((params[0] / 1000.0) * screen_w)
        ty = int((params[1] / 1000.0) * screen_h)
        print(f"    moving to ({tx}, {ty})...")
        smooth_move_to(tx, ty)
        time.sleep(CLICK_SETTLE_DELAY)
        print("    left click")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)

    elif func_name == "right_click" and len(params) == 2:
        tx = int((params[0] / 1000.0) * screen_w)
        ty = int((params[1] / 1000.0) * screen_h)
        print(f"    moving to ({tx}, {ty})...")
        smooth_move_to(tx, ty)
        time.sleep(CLICK_SETTLE_DELAY)
        print("    right click")
        ctypes.windll.user32.mouse_event(0x0008, 0, 0, 0, 0)
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0010, 0, 0, 0, 0)

    elif func_name == "double_left_click" and len(params) == 2:
        tx = int((params[0] / 1000.0) * screen_w)
        ty = int((params[1] / 1000.0) * screen_h)
        print(f"    moving to ({tx}, {ty})...")
        smooth_move_to(tx, ty)
        time.sleep(CLICK_SETTLE_DELAY)
        print("    double click")
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

        print(f"    moving to drag start ({x1}, {y1})...")
        smooth_move_to(x1, y1)
        time.sleep(0.1)
        print("    grabbing...")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.1)

        print(f"    dragging to ({x2}, {y2})...")
        sx, sy = get_cursor_position()
        for i in range(MOVEMENT_STEPS + 1):
            t = i / MOVEMENT_STEPS
            t = t * t * (3 - 2 * t)
            x = int(sx + (x2 - sx) * t)
            y = int(sy + (y2 - sy) * t)
            ctypes.windll.user32.SetCursorPos(x, y)
            time.sleep(STEP_DELAY)

        print("    releasing...")
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
    """Main loop with behavior-inferred mode transitions and multi-action support."""
    dump_dir = Path("dump") / datetime.now().strftime("run_%Y%m%d_%H%M%S")
    dump_dir.mkdir(parents=True, exist_ok=True)

    # Agent state
    mode = MODE_EXECUTING
    goal = INITIAL_GOAL
    story = "Franz starting up. Looking at the desktop for the first time."
    last_actions: list[str] = ["init"]
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
        print(f"  last:  {', '.join(last_actions)}")

        # -- Capture with all previous actions visualised --
        draw_func = create_visualization(last_actions)
        img = screenshot.capture_screen_png(
            TARGET_WIDTH, TARGET_HEIGHT, draw_func=draw_func
        )

        ts = int(time.time() * 1000)
        (dump_dir / f"{ts}.png").write_bytes(img)

        # -- Build the last-actions summary for the prompt --
        if last_actions == ["init"]:
            last_summary = "init (first iteration)"
        elif last_actions == ["observe()"]:
            last_summary = "observe (watched the screen, no interaction)"
        else:
            last_summary = " then ".join(last_actions)

        # -- Infer --
        print("  sending to VLM...")
        content = infer(img, goal, story, last_summary)
        print(f"  VLM response:\n{content}\n")

        # -- Parse --
        parsed = parse_response(content)

        # -- Classify actions --
        commands = parsed["commands"]
        has_real_action = any(fn != "observe" for fn, _ in commands)

        # -- Handle DONE --
        if parsed["done"]:
            if mode == MODE_EXECUTING:
                print("  >> task marked DONE, switching to monitoring")
                mode = MODE_MONITORING
                goal = MONITORING_GOAL
                idle_count = 0

        # -- Handle NEW GOAL --
        if parsed["new_goal"]:
            goal = parsed["new_goal"]
            mode = MODE_EXECUTING
            idle_count = 0
            print(f"  >> new goal: {goal}")

        # -- Infer mode from behavior --
        if mode == MODE_MONITORING and has_real_action:
            mode = MODE_EXECUTING
            print("  >> action detected in monitoring, switching to executing")
            if not parsed["new_goal"]:
                goal = parsed["story"] if parsed["story"] else "Responding to activity on screen."
                print(f"  >> inferred goal: {goal}")

        # -- Execute all commands in order --
        executed: list[str] = []

        if commands:
            action_count = len(commands)
            batch_label = "action" if action_count == 1 else f"{action_count} actions"
            print(f"  >> executing {batch_label}:")

            for idx, (func_name, params) in enumerate(commands, 1):
                cmd_str = format_command(func_name, params)
                print(f"  [{idx}/{action_count}] {cmd_str}")
                execute_action(func_name, params)
                executed.append(cmd_str)

                # Brief pause between actions in a multi-action batch
                if idx < action_count:
                    time.sleep(BETWEEN_ACTIONS_DELAY)
        else:
            print("  >> no valid action parsed, observing")
            executed.append("observe()")

        last_actions = executed

        # -- Track idle --
        if mode == MODE_MONITORING and not has_real_action:
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
            "last_actions": last_actions,
            "idle_count": idle_count,
            "action_count": len(commands),
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

## What Changed and Why

### System Prompt: The Walking Metaphor

The core addition:

> *Think of walking down a path. On a clear straight road where you can see everything ahead, you can take several steps at once. But when you reach a corner or a door, you stop and take just one step, then look again.*

This teaches the 2B model a **physical intuition** for action batching without using technical terms like "confidence" or "state prediction." The key phrases:

- **"everything you need is already visible"** — anchors multi-action to visual certainty
- **"will change the screen"** — anchors single-action to UI transitions
- **"never guess what will appear"** — hard prohibition on speculative batching

The three examples show: single action (menu click), multi-action (save dialog with visible targets), task completion, and new task detection. Small models learn patterns from examples far better than from abstract rules.

### Parser: Returns All Commands

`parse_response()` now returns **all** function calls found via `finditer()`, in order of appearance. No cap, no filter. If the model writes 5 actions, all 5 are returned.

### Execution Loop: Batched with Inter-Action Delays

All parsed commands execute sequentially with `BETWEEN_ACTIONS_DELAY` (300ms) between them. This gives the OS time to process each action (e.g., a click focusing a field before a `type()` sends keystrokes to it).

### `last_actions`: List Instead of String

The previous `last_action: str` is now `last_actions: list[str]`. The full batch is:

- **Shown in the next prompt** as a natural-language summary joined with " then " (e.g., `left_click(350,250) then type("cat.jpg") then left_click(700,530)`)
- **Visualised on the next screenshot** with numbered, color-coded annotations so the VLM can see what its entire batch did

### Visualisation: Multi-Action Overlays

`create_visualization()` now accepts a list of action strings and draws **all of them** with:
- **Rotating color pairs** (red/green, blue/yellow, green/red, yellow/blue) so actions are visually distinguishable
- **Action numbers** rendered as 2x-scaled 5x7 pixel digits next to each annotation

This is critical for the feedback loop: when the model batched 3 actions, it needs to see all 3 results to judge whether the batch succeeded or whether it needs to correct course.

### The `_DIGITS` Bitmap Font

A minimal 5x7 pixel font (digits 0-9) rendered at 2x scale for visibility. Used only for action numbering in visualisation overlays. Pure pixel manipulation, no dependencies.

### No Code Constraints on Action Count

The code **never** limits how many actions the model can produce. There is no `commands = commands[:1]` or `if len(commands) > N`. The model's judgment (as shaped by the walking metaphor) is the only governor. If the prompt engineering works, the model self-regulates. If it doesn't, the next screenshot reveals the error and the narrative recovery mechanism corrects course.

### `BETWEEN_ACTIONS_DELAY`

A new timing constant (300ms) inserted between sequential actions in a batch. Without this, rapid-fire actions can fail — for example, `left_click` on a text field followed immediately by `type()` might send keystrokes before the field gains focus. The delay gives the OS time to process each action before the next begins.