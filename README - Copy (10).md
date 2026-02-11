

```markdown
# FRANZ — Narrative-Driven Autonomous Desktop Agent

> A stateless, vision-language-model-powered computer-use agent that maintains coherent
> long-running autonomy through an evolving narrative memory — not conversation history.
> Supports adaptive multi-action batching governed by visual certainty.

---

## System Overview

FRANZ is an autonomous desktop automation agent that operates in a continuous
perception–reasoning–action loop against a **stateless** VLM API endpoint. Rather than
accumulating an ever-growing conversation history (which is both token-expensive and
context-window-limited), FRANZ maintains a **compressed narrative story** that is
**replaced in full** on every iteration. The story functions as an adaptive episodic
memory: it summarises what has happened, what the current state is, and what remains
to be done — all in natural language, all within a fixed token budget.

This architecture has a critical property: **the agent can run indefinitely**. Because
the narrative is regenerated each cycle rather than appended, context never overflows.
The VLM receives a fresh screenshot, the current story, and the last actions taken — a
complete situational snapshot — and produces one or more next actions plus an updated
narrative. The loop is therefore **temporally unbounded**.

### Key Architectural Innovations

| Concept | Description |
|---|---|
| **Narrative Memory** | A natural-language story replaces traditional chat history. It is rewritten each iteration, compressing past context into a fixed-size representation that the VLM can both read and write. |
| **Stateless Inference** | Every API call is self-contained: system prompt + user prompt (with story and last actions) + screenshot. No session state is maintained server-side. |
| **Perceptual Grounding** | Every decision is grounded in a current screenshot, not stale state. The agent re-perceives the world before every action. |
| **Adaptive Multi-Action Batching** | The VLM decides how many actions to emit per cycle based on visual certainty. If all targets are visible on screen, it batches multiple steps. If the next action will change the screen, it commits to one step and waits. No code-level cap is imposed. |
| **Behavior-Inferred Mode Switching** | Two modes (executing and monitoring) are inferred from the model's chosen actions rather than declared by the model via structured labels. Action output means executing; observe-only means monitoring. |
| **Human-Mimetic Execution** | Actions are executed with smooth cursor trajectories (smoothstep interpolation), realistic typing cadence, natural inter-action delays, and configurable pauses between batched actions. |
| **Visual Batch Feedback** | All actions from the previous batch are rendered as numbered, color-coded overlays on the screenshot sent to the VLM, providing the model with explicit visual evidence of what the entire batch did. |
| **Self-Modifying Goals** | The VLM can rewrite its own goal at any time by emitting `NEW GOAL` in its response. Combined with the `DONE` signal for task completion, this enables the agent to autonomously transition between tasks. |

### Operational Modes

| Mode | Behavior | Triggered By |
|---|---|---|
| **Executing** | Actively working toward a goal. Produces click/type/drag actions. Fast polling cadence (0.5s). | Initial start, `NEW GOAL` in VLM output, or any non-observe action while monitoring. |
| **Monitoring** | Passively scanning the screen for messages, unfinished work, errors. Produces only `observe()`. Slow polling cadence (3.0s). | `DONE` in VLM output, indicating the current task is complete. |

Mode transitions are **inferred from behavior**, not declared. This makes the system
robust to small VLMs that cannot reliably produce structured mode labels.

### The Walking Metaphor (Multi-Action Governance)

The multi-action batching is governed entirely by the VLM's judgment, trained through
a physical metaphor in the system prompt:

> Think of walking down a path. On a clear straight road where you can see everything
> ahead, you can take several steps at once. But when you reach a corner or a door,
> you stop and take just one step, then look again.

This maps to:
- **Clear road** = all UI targets for the next 2-3 steps are currently visible on screen → batch
- **Corner/door** = the next action will change what's on screen (menu, dialog, window switch) → single step

The code imposes **no limits** on action count. The metaphor is the only governor. If
it fails, the next screenshot reveals the real state and the narrative recovery
mechanism corrects course.

---

## Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FRANZ MAIN LOOP                              │
│                                                                     │
│  ┌───────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │  CAPTURE   │───▶│  ANNOTATE    │───▶│  ENCODE PNG              │  │
│  │ screenshot │    │ ALL actions  │    │  (screenshot.py)         │  │
│  │  (GDI)     │    │ from last    │    │                          │  │
│  │            │    │ batch with   │    │                          │  │
│  │            │    │ numbers and  │    │                          │  │
│  │            │    │ colors       │    │                          │  │
│  │            │    │ (drawing.py) │    │                          │  │
│  └───────────┘    └──────────────┘    └────────────┬─────────────┘  │
│                                                     │                │
│                                                     ▼                │
│                                       ┌──────────────────────────┐  │
│                                       │  COMPOSE API REQUEST     │  │
│                                       │                          │  │
│                                       │  ┌────────────────────┐  │  │
│                                       │  │ system prompt      │  │  │
│                                       │  ├────────────────────┤  │  │
│                                       │  │ user prompt with:  │  │  │
│                                       │  │  • current goal    │  │  │
│                                       │  │  • full story      │  │  │
│                                       │  │  • last actions    │  │  │
│                                       │  │    summary         │  │  │
│                                       │  │  • base64 PNG      │  │  │
│                                       │  └────────────────────┘  │  │
│                                       └────────────┬─────────────┘  │
│                                                     │                │
│                                                     ▼                │
│                                       ┌──────────────────────────┐  │
│                                       │  VLM INFERENCE           │  │
│                                       │  (stateless endpoint)    │  │
│                                       │                          │  │
│                                       │  POST /v1/chat/          │  │
│                                       │       completions        │  │
│                                       └────────────┬─────────────┘  │
│                                                     │                │
│                                                     ▼                │
│                                       ┌──────────────────────────┐  │
│                                       │  PARSE RESPONSE          │  │
│                                       │                          │  │
│                                       │  Extract:                │  │
│                                       │  • ALL function calls    │  │
│                                       │    (in order)            │  │
│                                       │  • narrative story       │  │
│                                       │  • DONE signal           │  │
│                                       │  • NEW GOAL (optional)   │  │
│                                       └────────────┬─────────────┘  │
│                                                     │                │
│                                       ┌─────────────┴────────────┐  │
│                                       ▼                          ▼   │
│                          ┌──────────────────┐  ┌───────────────────┐│
│                          │ EXECUTE BATCH    │  │ UPDATE STATE      ││
│                          │                  │  │                   ││
│                          │ for each action: │  │ • story replaced  ││
│                          │  • smooth move   │  │ • goal updated    ││
│                          │  • click/type/   │  │   (if DONE or     ││
│                          │    drag/observe  │  │    NEW GOAL)      ││
│                          │  • 300ms pause   │  │ • mode inferred   ││
│                          │    between       │  │   from behavior   ││
│                          │    actions       │  │ • state.json      ││
│                          └────────┬─────────┘  └──────┬────────────┘│
│                                   │                   │             │
│                                   └─────────┬─────────┘             │
│                                             │                       │
│                                             ▼                       │
│                                   ┌─────────────────┐               │
│                                   │ CADENCE DELAY   │               │
│                                   │ 0.5s executing  │               │
│                                   │ 3.0s monitoring  │               │
│                                   └────────┬────────┘               │
│                                            │                        │
│  ◄─────────────────────────────────────────┘                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow per Iteration

```
  goal(n)  +  story(n-1)  +  last_actions(n-1)  +  screenshot(n)
                           │
                           ▼
                      ┌─────────┐
                      │   VLM   │
                      └────┬────┘
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
     actions(n)      story(n)     signals(n)
     [1..N]                      DONE / NEW GOAL
             │             │             │
             ▼             │             ▼
     execute on OS         │      update mode/goal
             │             │             │
             └─────────────┴─────────────┘
                           ▼
                     next iteration
```

### Memory Model

```
  Traditional Agent          FRANZ
  ─────────────────          ─────

  msg[0]  system             system prompt     (fixed)
  msg[1]  user + img_1       user prompt       (fixed template)
  msg[2]  assistant_1          + goal(n)       (mutable, VLM-driven)
  msg[3]  user + img_2         + story(n)      (replaced each cycle)
  msg[4]  assistant_2          + last_actions  (replaced each cycle)
  msg[5]  user + img_3         + screenshot(n) (replaced each cycle)
  ...     (grows forever)
  msg[2n]  context overflow  CONSTANT SIZE --- never overflows
```

### Multi-Action Batch Lifecycle

```
  VLM sees: save dialog with filename field and Save button both visible

  VLM outputs:
    left_click(350, 250)      action 1: click filename field
    type("cat.jpg")           action 2: type the filename
    left_click(700, 530)      action 3: click Save button

  Execution:
    [1/3] left_click(350,250)   -- smooth move, click
          ... 300ms pause ...
    [2/3] type("cat.jpg")       -- character-by-character typing
          ... 300ms pause ...
    [3/3] left_click(700,530)   -- smooth move, click

  Next screenshot annotation:
    ① red crosshair + green circle at (350,250)
    ② blue crosshair + yellow circle at (700,530)
    (type actions have no spatial annotation)

  Prompt to VLM:
    Last actions: left_click(350,250) then type("cat.jpg") then left_click(700,530)
```

---

## File Reference

### `main.py` — Agent Core

The orchestration module implementing the perception–reasoning–action loop with
multi-action batching and behavior-inferred mode switching.

#### Constants and Configuration

| Constant | Type | Description |
|---|---|---|
| `API` | `str` | VLM inference endpoint URL |
| `MODEL` | `str` | Model identifier for the API |
| `TARGET_WIDTH` | `int` | Screenshot width sent to VLM (pixels) |
| `TARGET_HEIGHT` | `int` | Screenshot height sent to VLM (pixels) |
| `SAMPLING` | `dict` | Inference sampling parameters (`temperature`, `top_p`, `max_tokens`) |
| `ENABLE_VISUAL_FEEDBACK` | `bool` | Toggle action-overlay annotations on screenshots |
| `MOVEMENT_STEPS` | `int` | Number of interpolation steps for cursor movement |
| `STEP_DELAY` | `float` | Delay between movement steps (seconds) |
| `CLICK_SETTLE_DELAY` | `float` | Pause after cursor arrives before clicking (seconds) |
| `TYPING_CHAR_DELAY` | `float` | Delay between individual keystrokes (seconds) |
| `TYPING_WORD_DELAY` | `float` | Delay after space/newline (seconds) |
| `BETWEEN_ACTIONS_DELAY` | `float` | Pause between actions in a multi-action batch (seconds) |
| `EXECUTING_DELAY` | `float` | Loop delay when in executing mode (seconds) |
| `MONITORING_DELAY` | `float` | Loop delay when in monitoring mode (seconds) |
| `MODE_EXECUTING` | `str` | String constant `"executing"` |
| `MODE_MONITORING` | `str` | String constant `"monitoring"` |
| `MONITORING_GOAL` | `str` | Default goal text when transitioning to monitoring |
| `INITIAL_GOAL` | `str` | Starting goal for the agent |
| `SYSTEM_PROMPT` | `str` | VLM system prompt with walking metaphor and examples |
| `USER_PROMPT_TEMPLATE` | `str` | Template with `{goal}`, `{story}`, `{last}` placeholders |

#### Functions

| Function | Signature | Description |
|---|---|---|
| `infer` | `(png_data: bytes, goal: str, story: str, last: str) -> str` | Sends a single stateless inference request. Returns raw VLM response text. |
| `parse_response` | `(content: str) -> dict` | Extracts all function calls (in order), narrative story, `DONE` flag, and `NEW GOAL` text from VLM output. Returns `{"commands", "story", "done", "new_goal"}`. |
| `_parse_args` | `(func_name: str, raw_args: str) -> list or None` | Parses raw argument strings. Handles numeric args for spatial commands, string args for `type()`. |
| `create_visualization` | `(last_actions: list[str]) -> Callable` | Returns a drawing callback that overlays numbered, color-coded annotations for all actions from the previous batch. |
| `_draw_action_number` | `(rgba, width, height, x, y, number, color) -> bytes` | Renders a digit (0-9) as a 2x-scaled 5x7 bitmap at the given position. Used for action numbering in visualisation. |
| `execute_action` | `(func_name: str, params: list) -> None` | Executes a single parsed action with smooth human-like behavior. |
| `format_command` | `(func_name: str, params: list) -> str` | Reconstructs a function-call string from parsed components for logging. |
| `get_cursor_position` | `() -> tuple[int, int]` | Returns current cursor position via Win32 API. |
| `smooth_move_to` | `(target_x, target_y, steps, delay) -> None` | Moves cursor using smoothstep interpolation. |
| `press_key` | `(vk_code: int) -> None` | Presses and releases a virtual key. |
| `type_text` | `(text: str) -> None` | Types a string with human-like timing. |
| `main` | `() -> None` | Entry point. Runs the infinite perception-action loop. |

#### VLM Output Protocol

The VLM produces freeform text containing:

| Element | Format | Required | Description |
|---|---|---|---|
| Actions | `left_click(x,y)`, `type("text")`, etc. | At least one | One or more function calls, each on its own line |
| Story | Any non-action, non-signal text | Yes | 2-3 sentence narrative (extracted automatically) |
| `DONE` | The word `DONE` on its own line | No | Signals current task completion, triggers monitoring mode |
| `NEW GOAL` | `NEW GOAL` followed by goal text | No | Sets a new goal, triggers executing mode |

#### Supported Actions

| Command | Format | Coordinates | Description |
|---|---|---|---|
| `left_click` | `left_click(x, y)` | 0-1000 normalised | Single left mouse click |
| `right_click` | `right_click(x, y)` | 0-1000 normalised | Single right mouse click |
| `double_left_click` | `double_left_click(x, y)` | 0-1000 normalised | Double left mouse click |
| `drag` | `drag(x1, y1, x2, y2)` | 0-1000 normalised | Click-drag from start to end |
| `type` | `type("text")` | N/A | Keystroke input |
| `observe` | `observe()` | N/A | No-op, indicates passive observation |

---

### `screenshot.py` — Screen Capture and Encoding

A self-contained Windows screen capture module using GDI with DPI awareness. Usable
as both an importable module and a standalone CLI tool.

#### CLI Usage

```
python screenshot.py                       # Full screen to screenshot.png
python screenshot.py capture.png           # Full screen to capture.png
python screenshot.py out.png 1920 1080     # Capture and resize to out.png
```

#### Classes

| Class | Description |
|---|---|
| `BITMAPINFOHEADER` | ctypes Structure for Win32 `BITMAPINFOHEADER`. |
| `BITMAPINFO` | ctypes Structure for Win32 `BITMAPINFO` (header + color table). |
| `Win32Context` | Frozen dataclass wrapping `user32`, `gdi32`, and `shcore` DLL handles. Sets DPI awareness on creation. |

#### Functions

| Function | Signature | Description |
|---|---|---|
| `get_screen_size` | `() -> tuple[int, int]` | Returns `(width, height)` of primary display in physical pixels. |
| `capture` | `(sw: int, sh: int) -> bytes` | Captures screen as raw BGRA using GDI `BitBlt` with `CAPTUREBLT`. |
| `downsample` | `(src, sw, sh, dw, dh) -> bytes` | Resizes BGRA bitmap using `StretchBlt` with `HALFTONE` mode. |
| `bgra_to_rgba` | `(bgra: bytes) -> bytes` | Swizzles BGRA to RGBA, sets alpha to 255. |
| `encode_png` | `(rgba: bytes, sw: int, sh: int) -> bytes` | Manual PNG encoder (IHDR + IDAT + IEND). No external dependencies. |
| `capture_screen_png` | `(target_width, target_height, draw_func) -> bytes` | **Primary API.** Capture, downsample, convert, annotate, encode. |
| `main` | `() -> None` | CLI entry point. |

#### Callback Protocol

```python
def draw_func(rgba: bytes, width: int, height: int) -> bytes:
    """Receives RGBA data, returns modified RGBA data."""
```

---

### `drawing.py` — Pixel-Level Annotation Primitives

Pure-Python drawing utilities operating on RGBA byte buffers. No external dependencies.

#### Color Constants

| Constant | Value (R, G, B, A) | Used For |
|---|---|---|
| `RED` | `(255, 0, 0, 255)` | Action 1 primary (left click crosshair) |
| `GREEN` | `(0, 255, 0, 255)` | Action 1 secondary (click circle) |
| `BLUE` | `(0, 150, 255, 255)` | Action 2 primary, right click, drag arrow |
| `YELLOW` | `(255, 255, 0, 255)` | Action 2 secondary, right click circle, drag start |
| `WHITE` | `(255, 255, 255, 255)` | Available for custom annotations |
| `BLACK` | `(0, 0, 0, 255)` | Available for custom annotations |

#### Functions

| Function | Signature | Description |
|---|---|---|
| `draw_crosshair` | `(rgba, w, h, x, y, size, color, thickness) -> bytes` | Crosshair with center dot at target. |
| `draw_circle` | `(rgba, w, h, x, y, radius, color, filled) -> bytes` | Filled or hollow circle. |
| `draw_line` | `(rgba, w, h, x1, y1, x2, y2, color, thickness) -> bytes` | Bresenham line with thickness. |
| `draw_arrow` | `(rgba, w, h, x1, y1, x2, y2, color, thickness) -> bytes` | Line with arrowhead at endpoint. |
| `draw_rectangle` | `(rgba, w, h, x1, y1, x2, y2, color, thickness) -> bytes` | Axis-aligned rectangle outline. Available for future use. |
| `normalize_coord` | `(coord: int, max_val: int) -> int` | Maps 0-1000 normalised coordinate to pixel space. |

---

## Coordinate Pipeline

```
VLM output:  left_click(650, 420)        normalised 0-1000
                    |
                    v
Visualisation:  normalize_coord(650, 1536) = 998px   for annotation overlay
                normalize_coord(420, 864)  = 363px
                    |
                    v
Execution:      (650/1000) * screen_w = actual pixel  for OS cursor/click
                (420/1000) * screen_h = actual pixel
```

---

## Mode Transition Logic

```
                 START
                   |
                   v
            +-----------+
            | EXECUTING |<--------------------------+
            +-----------+                           |
                   |                                |
          VLM says DONE                    VLM outputs non-observe
                   |                       action while monitoring
                   v                                |
            +------------+                          |
            | MONITORING |>-------------------------+
            +------------+        OR: VLM says NEW GOAL
                   |
            observe() only
            slow cadence (3s)
```

No explicit mode label from the VLM is required. The code infers transitions:
- `DONE` in output while executing → switch to monitoring
- `NEW GOAL` in output → switch to executing with new goal
- Any non-observe action emitted while monitoring → switch to executing

---

## Deployment

### Requirements

- **OS**: Windows (Win32 GDI, `user32.dll`, `gdi32.dll`, `shcore.dll`)
- **Python**: 3.10+ (type unions, `slots=True`, walrus operator)
- **VLM Endpoint**: OpenAI-compatible `/v1/chat/completions` with vision support
- **No pip dependencies**: Standard library + ctypes only

### Configuration

Edit constants at the top of `main.py`:

```python
API = "http://localhost:1234/v1/chat/completions"
MODEL = "qwen3-vl-2b-instruct-1m"
TARGET_WIDTH = 1536
TARGET_HEIGHT = 864
```

### Running

```
python main.py
```

Screenshots, `state.json`, and `story.txt` are saved to `dump/run_YYYYMMDD_HHMMSS/`.

---

## Design Philosophy

FRANZ treats the VLM not as a chatbot but as a **perceptual cortex** — a stateless
function that maps `(visual_input, narrative_context, last_actions)` to
`(next_actions, updated_narrative, control_signals)`. The narrative is not a log; it
is a **lossy compression of episodic memory** that the agent rewrites on every cycle
to maintain relevance within a fixed context window.

The multi-action batching is an expression of trust in the model's visual judgment.
Rather than imposing a rigid single-step policy from the code side, the system
teaches the model a physical intuition (the walking metaphor) and lets it self-regulate.
When the model is wrong, the perceptual grounding (next screenshot) provides immediate
corrective feedback. This is the same principle that makes human computer usage
resilient: you act, you look, you adjust.

The monitoring/executing duality with screen-as-communication-channel transforms FRANZ
from a one-shot task executor into a **persistent desktop presence**. The user
communicates by making things visible on screen — a note in Notepad, an open dialog,
an error message — and FRANZ responds by perceiving and acting. No API, no IPC, no
file watching. The screen is the interface.

---

## Continuation Prompt

Use the prompt below to start a new conversation about this codebase. Paste it as
your first message, then provide the code files in your second message.

```
I am working on FRANZ, an autonomous desktop automation agent for Windows.

Architecture summary:
- Stateless VLM loop: each iteration sends system prompt + user prompt + screenshot
  to a local OpenAI-compatible /v1/chat/completions endpoint with vision support.
- Narrative memory: a short story (not conversation history) is rewritten each cycle
  and sent in full. This keeps context size constant across unlimited iterations.
- Behavior-inferred mode switching: two modes (executing, monitoring). Mode is
  inferred from the VLM's chosen actions, not from structured labels. DONE and
  NEW GOAL keywords control transitions.
- Adaptive multi-action batching: the VLM decides how many actions per cycle based
  on visual certainty (taught via a "walking" metaphor). No code-level cap.
- Human-mimetic execution: smooth cursor movement (smoothstep), realistic typing,
  configurable inter-action delays.
- Visual batch feedback: all actions from the previous batch are drawn as numbered,
  color-coded overlays on the next screenshot.
- Screen-as-communication: the user communicates with Franz by making things visible
  on screen (notes in Notepad, open dialogs, etc.). Franz detects them visually.

The system has 3 files:
- main.py: agent loop, VLM inference, response parsing, action execution,
  mode/goal management, multi-action batching, visualisation orchestration.
- screenshot.py: Windows GDI screen capture with DPI awareness, downsampling,
  BGRA-to-RGBA conversion, manual PNG encoding, draw_func callback support.
- drawing.py: pure-Python pixel-level primitives (crosshair, circle, line, arrow,
  rectangle, normalize_coord) operating on RGBA byte buffers.

Target VLM: 2B parameter model (Qwen3-VL-2B). All prompt engineering must account
for small model limitations: simple language, concrete examples over abstract rules,
no special/unicode characters in prompts, minimal structured output requirements.

In my next message I will share the current code files. Please read them fully,
understand the complete workflow, then ask me what I want to change or fix. If I
describe a feature or bug, reason through it carefully (including mental simulation
of how the 2B model will behave), then produce the full corrected file(s).
```
```