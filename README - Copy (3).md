

## Analysis

Let me trace through the workflow and identify all issues:

### Workflow Understanding
This is a stateless AI computer-usage agent that:
1. Captures a screenshot each iteration
2. Sends it with a user prompt to a VLM (Vision Language Model) via a local API
3. Parses the response for action commands and a narrative "story"
4. Executes the actions (click, type, drag, etc.)
5. Loops, sending a new screenshot each time

### Issues Found

**1. Dead code: `STORY_START`/`STORY_END` parsing in `parse_response()`**
The system prompt tells the AI to output function calls like `left_click(x,y)` and a 3-sentence narrative. It never mentions `STORY_START`/`STORY_END` tags, triple-quoted docstrings, or the old command format (`click 500 300`). The entire `parse_response()` is built around a dead format. Must be rewritten to match what the system prompt actually asks for.

**2. Command format mismatch between system prompt and parser**
The system prompt defines: `left_click(x,y)`, `right_click(x,y)`, `double_left_click(x,y)`, `drag(x1,y1,x2,y2)`, `type(text)`. But `parse_response()` and `parse_action()` look for space-separated words like `click 500 300`, `rightclick`, `doubleclick`, `drag`, `type`. These will **never match** the AI's output. Must be aligned.

**3. `USER_PROMPT_TEMPLATE` uses `.format(story=story, last=last)` but has no `{story}` or `{last}` placeholders**
The story and last-action context are never actually sent to the VLM. The user message is always the same static string. Must add placeholders so the AI knows the current story state and what happened last.

**4. Story is never sent in full to the VLM**
Related to #3 — since `{story}` isn't in the template, the accumulated story is lost. The design intent is that the story is replaced each iteration and sent in full. Must fix the template.

**5. `parse_response()` command matching is fragile**
Lines like `ln.strip().lower().split()[0] in ("click", ...)` will crash on empty lines and won't match function-call syntax like `left_click(500,300)`.

**6. `parse_action()` doesn't understand function-call syntax**
It splits on whitespace expecting `click 500 300`, but the AI will output `left_click(500,300)`.

**7. `create_visualization()` calls `parse_action()` with old format assumptions**
Must be updated to work with the new function-call format that the AI actually outputs.

### What I'm Modifying

| Area | Change |
|---|---|
| `USER_PROMPT_TEMPLATE` | Add `{story}` and `{last}` placeholders |
| `parse_response()` | Rewrite to extract function calls like `left_click(500,300)` and narrative text (non-code lines) |
| `parse_action()` | Rewrite to parse `left_click(500,300)` style commands |
| `create_visualization()` | Update action type strings to match new parsed format |
| `execute_action()` | Update action type strings to match new parsed format |
| Remove all `STORY_START`/`STORY_END` and triple-quote docstring parsing | Dead code |

### Full Corrected Code

```python
# main.py
"""FRANZ - AI-driven Windows automation agent with smooth, human-like interactions."""

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

# Configuration
API: Final = "http://localhost:1234/v1/chat/completions"
MODEL: Final = "qwen3-vl-2b-instruct-1m"
TARGET_WIDTH: Final = 1536
TARGET_HEIGHT: Final = 864
SAMPLING: Final = {"temperature": 0.7, "top_p": 0.9, "max_tokens": 1000}
ENABLE_VISUAL_FEEDBACK: Final = True

# Animation settings
MOVEMENT_STEPS: Final = 20
STEP_DELAY: Final = 0.01  # 10ms per step = 200ms total movement time
CLICK_SETTLE_DELAY: Final = 0.15  # Pause after arriving before clicking
TYPING_CHAR_DELAY: Final = 0.08  # 80ms between characters (realistic typing)
TYPING_WORD_DELAY: Final = 0.15  # 150ms pause after spaces (thinking)

SYSTEM_PROMPT: Final = """
You are a visual interpreter with storytelling power that processes visual data into meaningful insights that guide human actions, a narrator of system state, reasoning through what's shown to help the user move forward.

Your internal process:
- see (components),
- understand (behavior and context),
- tell a story that explains why something matters,
- give actionable advice, based on that narrative.

Using below python functions, to interact with the computer: (values: 0-1000 normalized coords)
left_click(x,y)
right_click(x,y)
double_left_click(x,y)
drag(start_pos_x,start_pos_y,end_pos_x,end_pos_y)
type(text)

Output:
- guaranteed single step toward directing the User towards their ultimate goal.
- 3 sentences narrative story explaining why that recommendation is logical and what is the expected outcome
"""

USER_PROMPT_TEMPLATE: Final = """
I want to draw a cat in ms paint, then save it on desktop as a "cat.jpg" file, start an iterative process (a story) that will bring me iteratively step-by-step towards my end goal.

Current story so far:
{story}

Last action performed:
{last}

Look at the current screenshot and decide the single next action to take. Output exactly one function call inside a ```python``` code block, followed by your 3-sentence narrative story.
"""


def infer(png_data: bytes, story: str, last: str) -> str:
    """Send inference request to API."""
    payload = {
        "model": MODEL,
        "messages": [
            {"role": "system", "content": SYSTEM_PROMPT},
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": USER_PROMPT_TEMPLATE.format(story=story, last=last)},
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


# Regex matching function calls: left_click(500,300), type("hello world"), drag(10,20,30,40), etc.
_FUNC_CALL_RE = re.compile(
    r"\b(left_click|right_click|double_left_click|drag|type)\s*\(([^)]*)\)"
)


def parse_response(content: str) -> tuple[list[tuple[str, list]], str]:
    """Parse AI response for commands and story.

    Returns:
        Tuple of (list of (action_name, params), narrative_story_text)
    """
    # Try to extract from a fenced code block first
    fence = re.search(r"```(?:python)?\s*(.*?)```", content, re.DOTALL | re.IGNORECASE)
    code_block = fence.group(1) if fence else content

    # Find all function calls
    commands: list[tuple[str, list]] = []
    for match in _FUNC_CALL_RE.finditer(code_block):
        func_name = match.group(1)
        raw_args = match.group(2).strip()
        params = _parse_args(func_name, raw_args)
        if params is not None:
            commands.append((func_name, params))

    # Extract narrative: everything outside the code fence, or non-function-call lines
    if fence:
        story_parts = []
        # Text before the code block
        before = content[:fence.start()].strip()
        if before:
            story_parts.append(before)
        # Text after the code block
        after = content[fence.end():].strip()
        if after:
            story_parts.append(after)
        story = "\n".join(story_parts).strip()
    else:
        # No code fence — take lines that aren't function calls
        story_lines = []
        for line in content.splitlines():
            stripped = line.strip()
            if stripped and not _FUNC_CALL_RE.search(stripped):
                story_lines.append(stripped)
        story = " ".join(story_lines).strip()

    # Clean up common markdown/comment artifacts from story
    story = re.sub(r"^[#*>\-]+\s*", "", story, flags=re.MULTILINE).strip()

    return commands, story


def _parse_args(func_name: str, raw_args: str) -> list | None:
    """Parse raw argument string into a typed parameter list."""
    if func_name == "type":
        # Expect a single string argument, possibly quoted
        text = raw_args.strip()
        if (text.startswith('"') and text.endswith('"')) or (text.startswith("'") and text.endswith("'")):
            text = text[1:-1]
        return [text] if text else None

    # All other commands expect numeric arguments
    nums = re.findall(r"-?\d+", raw_args)

    if func_name in ("left_click", "right_click", "double_left_click") and len(nums) >= 2:
        return [int(nums[0]), int(nums[1])]
    elif func_name == "drag" and len(nums) >= 4:
        return [int(nums[0]), int(nums[1]), int(nums[2]), int(nums[3])]

    return None


def create_visualization(last_action: str) -> callable:
    """Create drawing function based on last action."""
    def draw_annotations(rgba: bytes, width: int, height: int) -> bytes:
        if not ENABLE_VISUAL_FEEDBACK or last_action in ("init", "observe"):
            return rgba

        # Try to parse the last action as a function call
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
        # Ease-in-out interpolation for more natural movement
        t = t * t * (3 - 2 * t)  # Smoothstep function

        x = int(start_x + (target_x - start_x) * t)
        y = int(start_y + (target_y - start_y) * t)

        ctypes.windll.user32.SetCursorPos(x, y)
        time.sleep(delay)


def press_key(vk_code: int) -> None:
    """Press and release a virtual key."""
    ctypes.windll.user32.keybd_event(vk_code, 0, 0, 0)  # Key down
    time.sleep(0.02)
    ctypes.windll.user32.keybd_event(vk_code, 0, 2, 0)  # Key up (KEYEVENTF_KEYUP = 2)


def type_text(text: str) -> None:
    """Type text with natural human-like timing."""
    print(f'  → Typing: "{text}"')

    for char in text:
        if char == ' ':
            press_key(0x20)  # VK_SPACE
            time.sleep(TYPING_WORD_DELAY)
        elif char == '\n':
            press_key(0x0D)  # VK_RETURN
            time.sleep(TYPING_WORD_DELAY)
        else:
            vk = ctypes.windll.user32.VkKeyScanW(ord(char))
            if vk != -1:
                shift_needed = (vk >> 8) & 1
                vk_code = vk & 0xFF

                if shift_needed:
                    ctypes.windll.user32.keybd_event(0x10, 0, 0, 0)  # Shift down
                    time.sleep(0.01)

                press_key(vk_code)

                if shift_needed:
                    ctypes.windll.user32.keybd_event(0x10, 0, 2, 0)  # Shift up

            time.sleep(TYPING_CHAR_DELAY)


def execute_action(func_name: str, params: list) -> None:
    """Execute action with smooth, human-like behavior."""
    screen_w = ctypes.windll.user32.GetSystemMetrics(0)
    screen_h = ctypes.windll.user32.GetSystemMetrics(1)

    if func_name == "left_click" and len(params) == 2:
        target_x = int((params[0] / 1000.0) * screen_w)
        target_y = int((params[1] / 1000.0) * screen_h)

        print(f"  → Moving to ({target_x}, {target_y})...")
        smooth_move_to(target_x, target_y)
        time.sleep(CLICK_SETTLE_DELAY)

        print(f"  → Left clicking...")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)  # MOUSEEVENTF_LEFTDOWN
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)  # MOUSEEVENTF_LEFTUP

    elif func_name == "right_click" and len(params) == 2:
        target_x = int((params[0] / 1000.0) * screen_w)
        target_y = int((params[1] / 1000.0) * screen_h)

        print(f"  → Moving to ({target_x}, {target_y})...")
        smooth_move_to(target_x, target_y)
        time.sleep(CLICK_SETTLE_DELAY)

        print(f"  → Right clicking...")
        ctypes.windll.user32.mouse_event(0x0008, 0, 0, 0, 0)  # MOUSEEVENTF_RIGHTDOWN
        time.sleep(0.05)
        ctypes.windll.user32.mouse_event(0x0010, 0, 0, 0, 0)  # MOUSEEVENTF_RIGHTUP

    elif func_name == "double_left_click" and len(params) == 2:
        target_x = int((params[0] / 1000.0) * screen_w)
        target_y = int((params[1] / 1000.0) * screen_h)

        print(f"  → Moving to ({target_x}, {target_y})...")
        smooth_move_to(target_x, target_y)
        time.sleep(CLICK_SETTLE_DELAY)

        print(f"  → Double clicking...")
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

        print(f"  → Grabbing...")
        ctypes.windll.user32.mouse_event(0x0002, 0, 0, 0, 0)
        time.sleep(0.1)

        print(f"  → Dragging to ({x2}, {y2})...")
        start_x, start_y = get_cursor_position()
        for i in range(MOVEMENT_STEPS + 1):
            t = i / MOVEMENT_STEPS
            t = t * t * (3 - 2 * t)  # Smoothstep

            x = int(start_x + (x2 - start_x) * t)
            y = int(start_y + (y2 - start_y) * t)

            ctypes.windll.user32.SetCursorPos(x, y)
            time.sleep(STEP_DELAY)

        print(f"  → Releasing...")
        time.sleep(0.1)
        ctypes.windll.user32.mouse_event(0x0004, 0, 0, 0, 0)

    elif func_name == "type" and len(params) == 1:
        type_text(params[0])


def _format_command(func_name: str, params: list) -> str:
    """Format a parsed command back into its function-call string for logging/tracking."""
    if func_name == "type":
        return f'type("{params[0]}")'
    return f"{func_name}({','.join(str(p) for p in params)})"


def main() -> None:
    """Main automation loop."""
    dump_dir = Path("dump") / datetime.now().strftime("run_%Y%m%d_%H%M%S")
    dump_dir.mkdir(parents=True, exist_ok=True)

    story = "FRANZ observing desktop."
    last_action = "init"

    iteration = 0

    print("\n" + "=" * 60)
    print("FRANZ - Smooth Human-Like Automation Agent")
    print("=" * 60)
    print(f"Screenshots: {dump_dir}")
    print(f"Movement: {MOVEMENT_STEPS} steps @ {STEP_DELAY * 1000}ms = {MOVEMENT_STEPS * STEP_DELAY * 1000}ms")
    print(f"Click settle: {CLICK_SETTLE_DELAY * 1000}ms")
    print(f"Typing: {TYPING_CHAR_DELAY * 1000}ms/char, {TYPING_WORD_DELAY * 1000}ms/word")
    print("=" * 60 + "\n")

    while True:
        iteration += 1
        print(f"\n{'=' * 60}")
        print(f"Iteration {iteration}")
        print(f"{'=' * 60}")
        print(f"Story: {story}")
        print(f"Last: {last_action}")

        draw_func = create_visualization(last_action)
        img = screenshot.capture_screen_png(TARGET_WIDTH, TARGET_HEIGHT, draw_func=draw_func)

        timestamp = int(time.time() * 1000)
        (dump_dir / f"{timestamp}.png").write_bytes(img)

        print("Sending to AI...")
        content = infer(img, story, last_action)
        print(f"AI response:\n{content}\n")

        commands, story_candidate = parse_response(content)

        if commands:
            print(f"Executing {len(commands)} command(s):")
            for idx, (func_name, params) in enumerate(commands, 1):
                cmd_str = _format_command(func_name, params)
                print(f"\n[{idx}] {cmd_str}")
                execute_action(func_name, params)
                last_action = cmd_str
                time.sleep(0.2)
        else:
            print("Observing only (no commands)")
            last_action = "observe"

        if story_candidate:
            story = story_candidate
            (dump_dir / "story.txt").write_text(story, encoding="utf-8")
            print(f"\nStory updated: {story}")

        time.sleep(0.5)


if __name__ == "__main__":
    main()
```

### Summary of All Changes

| # | What Changed | Why |
|---|---|---|
| 1 | Added `{story}` and `{last}` placeholders to `USER_PROMPT_TEMPLATE` | Story and last action were never sent to the VLM — the `.format()` call existed but had no placeholders to fill |
| 2 | Added explicit instructions in the user template telling the AI to output one function call in a code block + narrative | Guides the model to produce parseable output |
| 3 | Rewrote `parse_response()` completely | Removed all dead `STORY_START`/`STORY_END` tag parsing, triple-quote docstring extraction, and space-separated command parsing. Now uses regex to find function calls like `left_click(500,300)` |
| 4 | Added `_FUNC_CALL_RE` compiled regex | Single source of truth for matching the function-call syntax defined in the system prompt |
| 5 | Added `_parse_args()` helper | Cleanly parses numeric args for clicks/drags and string args for `type()` |
| 6 | Removed `parse_action()` entirely | It parsed the old dead format (`click 500 300`). Replaced by `_parse_args()` working with the regex matches |
| 7 | Rewrote `execute_action()` to accept `(func_name, params)` | Instead of re-parsing a string, it now receives already-parsed data. Action names match the system prompt: `left_click`, `right_click`, `double_left_click`, `drag`, `type` |
| 8 | Rewrote `create_visualization()` | Uses `_FUNC_CALL_RE` + `_parse_args()` to parse the last action string, matching the new function-call format |
| 9 | Added `_format_command()` helper | Reconstructs a function-call string from parsed data for logging and `last_action` tracking |
| 10 | Removed dummy `keybd_event` call in `type_text()` | The `keybd_event(0,0,0,0)` "dummy to ensure focus" does nothing useful |
| 11 | Added `import ctypes.wintypes` at top level | Was implicitly relied upon but not explicitly imported |
| 12 | `parse_response()` returns `list[tuple[str, list]]` | Commands are now pre-parsed tuples, eliminating redundant re-parsing downstream — no data slicing anywhere |