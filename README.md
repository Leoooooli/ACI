<p align="center">
  <h1 align="center">ACI — Agent-Computer Interface</h1>
  <p align="center">
    <strong>APIs let developers talk to software.<br>ACI lets AI agents talk to software.</strong>
  </p>
  <p align="center">
    The open standard that turns any application — web or desktop — into a structured, agent-operable interface.<br>
    One protocol. Any software. No screenshots.
  </p>
</p>

<p align="center">
  <a href="#the-aci-protocol">Protocol</a> &bull;
  <a href="#quick-start">Quick Start</a> &bull;
  <a href="#the-three-innovations">Innovations</a> &bull;
  <a href="#skill-definition">Skill Definition</a> &bull;
  <a href="#knowledge-base">Knowledge Base</a> &bull;
  <a href="#api-reference">API Reference</a> &bull;
  <a href="#contributing">Contributing</a>
</p>

<p align="center">
  <!-- TODO: Add badges after publishing -->
  <!-- ![Python](https://img.shields.io/badge/python-3.8+-blue) -->
  <!-- ![License](https://img.shields.io/badge/license-MIT-green) -->
  <!-- ![Tests](https://img.shields.io/badge/tests-95%20passing-brightgreen) -->
</p>

---

## Why ACI, Why Now

In 2024-2025, AI agents went from research demos to production systems. Anthropic shipped Computer Use. OpenAI launched Operator. Microsoft built UFO. The demand is proven — **agents need to use software**.

But every solution so far solves only half the problem:

- **Screenshot-based** (Computer Use, Operator) — works on any app, but slow (3-10s per action), expensive (vision API calls), and fragile (pixel-guessing).
- **Web-only** (Browser Use, Playwright MCP) — fast and structured, but agents are trapped in the browser. Can't touch desktop apps.
- **Desktop-only** (UFO, pywinauto) — can control native apps, but can't operate web pages. No unified protocol.

The missing piece is obvious: **a single standard interface that works for both web and desktop, structured and fast, that any agent can use.**

That's ACI. The **Agent-Computer Interface**.

Just as APIs became the universal interface between developers and software, ACI is the universal interface between AI agents and software. One protocol, any application, no screenshots.

```
 Developers use APIs to build on software.
 ┌──────────┐    API     ┌──────────────┐
 │Developer │ ────────── │  Software    │
 └──────────┘            └──────────────┘

 Agents use ACI to operate software.
 ┌──────────┐    ACI     ┌──────────────┐
 │ AI Agent │ ────────── │  Software    │
 └──────────┘            └──────────────┘
```

---

## The ACI Protocol

ACI defines a two-operation protocol that works for **any** application:

### `perceive` — See what's on screen

Returns a **structured, UID-referenced element tree** — not a screenshot, not raw HTML.

```
Page: Hacker News
URL: https://news.ycombinator.com
Elements: 42 interactive, 6 landmarks

Interactive:
  link "Hacker News" [@e1]
  link "new" [@e2]
  link "past" [@e3]
  textbox "Search" (type=text) [@e4]
  link "Show HN: I built a real-time..." (href=...) [@e5]
  link "198 comments" [@e6]

Content:
  1. Show HN: I built a real-time collaborative editor (198 points)
  2. Why Rust is winning in systems programming (142 points)
  ...
```

Same protocol, desktop application:

```
Window: WeChat
Elements: 15 interactive

Interactive:
  button "Search" [oc_0]
  button "Contacts" [oc_1]
  textbox "Message input" [vc_7]
  button "Send" [vc_8]

Shortcuts: enter=send message, alt+s=focus input, ctrl+alt+w=toggle window
```

The agent doesn't know or care whether it's talking to Chrome or WeChat. The interface is identical.

### `act` — Do something

```json
{"action_type": "click", "target_uid": "@e5"}          // Click by UID
{"action_type": "type", "target_uid": "@e4", "value": "ACI framework"}  // Type text
{"action_type": "press_key", "value": "Enter"}          // Press keyboard key
{"action_type": "scroll", "value": "down"}              // Scroll
{"action_type": "sequence", "value": "[{\"action_type\":\"type\",...},{\"action_type\":\"press_key\",...}]"}  // Atomic multi-step
```

Every action returns a structured result: `{success, elapsed_ms, ui_change_detected}`.

**That's the entire protocol.** `perceive → act → perceive → act`. Any agent that can make HTTP calls can operate any software.

---

## The Three Innovations

### Innovation 1: Tiered Structured Extraction

The core technical contribution. Instead of screenshots, ACI uses **fast structured methods first** and falls back to vision only when necessary:

```
Web:                              Desktop:
┌─────────────────────┐           ┌─────────────────────┐
│ T0: CDP A11y Tree   │ 5-50ms   │ T1: UIA Control Tree│ 50-300ms
│ (buttons, links,    │           │ (native controls,   │
│  inputs, headings)  │           │  menus, toolbars)   │
├─────────────────────┤           ├─────────────────────┤
│ T1: DOM Supplement  │ 20-100ms  │ T2: Cursor Probing  │ ~350ms
│ (shadow DOM, custom │           │ (grid scan cursor   │
│  elements, iframes) │           │  shapes: HAND/IBEAM)│
├─────────────────────┤           ├─────────────────────┤
│ T2: Vision Fallback │ 1-2s     │ T3: OCR Engine      │ ~200ms
│ (only if <5 elements│           │ (text on unlabeled  │
│  found above)       │           │  visual elements)   │
└─────────────────────┘           ├─────────────────────┤
                                  │ T4: VLM Fallback    │ 1-5s
                                  │ (vision model, last │
                                  │  resort only)       │
                                  └─────────────────────┘
```

**Why this matters:** Most interactions complete in **50-100ms** instead of 3-10 seconds. Vision models are expensive — ACI only calls them when structured extraction genuinely fails. Results are merged via **IoU deduplication** (>0.7 overlap = same element).

### Innovation 2: Community Knowledge Base (YAML App Profiles)

Every application can have a YAML file that teaches agents how to use it — shortcuts, UI patterns, icon meanings. **No code changes required. Community-contributable.**

```yaml
# data/knowledge_base/vscode.yaml
app: vscode
aliases: ["visual studio code", "code"]
process_name: "Code.exe"

shortcuts:
  ctrl+shift+p: "Command Palette"
  ctrl+p: "Quick Open file"
  ctrl+`: "Toggle terminal"
  f5: "Start debugging"

common_icons:
  - pattern: "play_triangle"
    label: "Run / Debug"
    action: click
```

When an agent perceives VS Code, it **automatically receives these shortcuts** — so it presses `Ctrl+Shift+P` instead of hunting for pixels. Think of it as **crowdsourced app drivers for AI agents**.

The knowledge base also supports:
- **Pseudo-UIA persistence** — cached element layouts from previous scans, so apps with poor accessibility (WeChat, custom Electron apps) load instantly on repeat visits
- **Muscle memory** — OpenCV template matching remembers what actions worked on which elements
- **Spatial inference** — icon-only buttons get automatic hints: *"top-right corner, small size → probably window controls"*

### Innovation 3: Interrupt-Aware Execution

Real software throws popups, cookie banners, auth dialogs, and error modals. Most agent frameworks crash or freeze.

ACI has a **MutationShield** system:

| Layer | What It Catches | How |
|-------|----------------|-----|
| DOM Observer | High z-index overlays (modals, banners) | Injected `MutationObserver`, fires on elements with `z-index > 1000` |
| Dialog Handler | Native `alert()`, `confirm()`, `prompt()` | Playwright dialog event → auto-dismiss + notify agent |
| URL Monitor | Unexpected redirects | Polls `page.url` every 500ms |
| Desktop Shield | New windows blocking target app | Win32 foreground window tracking |

Interrupts are reported as structured `UIInterruptEvent` objects — the agent knows *what* happened and can decide how to handle it, instead of failing silently.

---

## Skill Definition: Agent-Ready Packaging

ACI ships with a **`SKILL.md`** — a structured skill definition that any LLM can consume as a system prompt. Drop it into your agent's context and it immediately knows how to use ACI:

```markdown
# SKILL.md — what agents receive
- Mandatory rules (always perceive before acting, use UIDs, verify critical actions)
- Complete CLI reference with examples
- 4-tier detection explanation
- App knowledge shortcut guide
- Recovery patterns (element not found, wrong window, action failed)
- Desktop workflow template (launch → wait → perceive → act)
```

**This is plug-and-play.** Give any LLM the SKILL.md + ACI daemon access, and it can operate software. No fine-tuning, no custom training, no prompt engineering. Tested with Claude (Haiku, Sonnet, Opus) — a basic one-line human instruction is sufficient.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│              AI Agent (Any LLM)                  │
│       Reads SKILL.md → knows the protocol        │
└───────────────────┬─────────────────────────────┘
                    │ REST / JSON
                    ▼
┌─────────────────────────────────────────────────┐
│              ACI Daemon (FastAPI)                 │
│              Port 11434                          │
│                                                  │
│  SessionManager    StateMachine    ProtocolRouter │
│  (I/O mutex per    (IDLE→EXEC→    (request_id    │
│   session)          BLOCKED→DONE)  correlation)  │
└────────┬──────────────────────────┬──────────────┘
         │ WebSocket                │ WebSocket
    ┌────▼──────────┐         ┌────▼──────────┐
    │  Web Bridge   │         │Desktop Bridge │
    │  (Playwright) │         │ (UIA+Vision)  │
    │               │         │               │
    │ T0: CDP A11y  │         │ T1: UIA Tree  │
    │ T1: DOM Parse │         │ T2: Cursor    │
    │ T2: Vision    │         │ T3: OCR       │
    │ MutationShield│         │ T4: VLM       │
    │ HoverProber   │         │ DesktopShield │
    │ FrameManager  │         │ PhysicalInput │
    └───────────────┘         └───────────────┘
```

### Design Principles

1. **Daemon-Worker Separation** — Central daemon manages sessions and state; bridge workers run as isolated subprocesses. Crash isolation: a browser crash doesn't take down the desktop bridge.

2. **Request-Response Correlation** — Every action gets a unique `request_id`. The daemon creates an `asyncio.Future`, sends via WebSocket, and awaits the exact matching response. No queue desync.

3. **Per-Session I/O Mutex** — Two agents can't physically click at the same time in the same session. The lock ensures serialized physical input.

4. **State Machine with Interrupt Stack** — When a popup appears mid-action, state transitions to `BLOCKED_BY_UI` and the original context is pushed onto a call stack. After dismissing the popup, the original task resumes.

---

## Quick Start

### Installation

```bash
git clone https://github.com/Leoooooli/ACI.git
cd ACI
pip install -r requirements.txt
playwright install chromium
```

### Start

```bash
# Option 1: PowerShell launcher (Windows)
powershell -ExecutionPolicy Bypass -File scripts/start_aci.ps1

# Option 2: Manual start (3 terminals)
# Terminal 1: Daemon
set PYTHONPATH=. && python -m core.server

# Terminal 2: Web bridge
set PYTHONPATH=. && python -m bridges.web_bridge.worker

# Terminal 3: Desktop bridge (Windows only)
set PYTHONPATH=. && python -m bridges.desktop_bridge.worker
```

### Use

```bash
# Create session + perceive a website
python scripts/aci_client.py --action start --session demo --env web --url "https://news.ycombinator.com"
python scripts/aci_client.py --action perceive --session demo --env web

# Click an element by UID
python scripts/aci_client.py --action act --session demo --env web --type click --uid "@e5"

# Desktop: launch an app and interact
python scripts/aci_client.py --action start --session desk --env desktop
python scripts/aci_client.py --action act --session desk --env desktop --type launch_app --value "notepad"
python scripts/aci_client.py --action act --session desk --env desktop --type wait --value "2"
python scripts/aci_client.py --action perceive --session desk --env desktop
```

---

## The ACI Specification

ACI defines these contracts as its core specification. Any implementation that conforms to these schemas is ACI-compatible.

### `ContextPerception` — What agents see

```
state: TaskState              # idle | executing | blocked_by_ui | failed | completed
session_id: str               # Session identifier
context_env: "web"|"desktop"  # Environment type
elements: [UIDNode]           # Structured UI elements
spatial_context: str           # Human-readable text snapshot
app_knowledge: dict            # Shortcuts & patterns from knowledge base
current_url: str               # Web: current page URL
active_window_title: str       # Window/tab title
visual_reference_image: str    # Path to annotated screenshot (when vision tier fires)
```

### `UIDNode` — A single UI element

```
uid: str                       # Stable reference: "@e1" (web), "oc_5" / "vc_3" (desktop)
tag: str                       # "button", "textbox", "link", "heading", ...
role: str                      # ARIA role or UIA control type
text: str                      # Visible label (max 200 chars)
attributes: dict               # href, placeholder, aria-label, spatial_hint, thumbnail, ...
bbox: (x, y, w, h)            # Bounding box in screen pixels
interactable: bool             # Can the agent interact with this?
tier: str                      # Which detection tier found it: "a11y" | "dom" | "uia" | "cursor_probe" | "ocr" | "vlm"
```

### `ActionRequest` — What agents send

```
session_id: str
action_type: "click" | "type" | "press_key" | "scroll" | "wait" | "hover" | "launch_app" | "execute_js" | "sequence"
target_uid: str                # UID from perceive (required for click/type/hover)
value: str                     # Text to type, key to press, app to launch, etc.
context_env: "web"|"desktop"
```

### `ActionResult` — What agents get back

```
success: bool
action_type: str
elapsed_ms: float              # How long the action took
ui_change_detected: bool       # Did the screen change after the action?
verification_screenshot: str   # Post-action screenshot path (when available)
error: str                     # Error description on failure
```

### `UIInterruptEvent` — Anomaly notifications

```
interrupt_type: "modal" | "dialog" | "overlay" | "redirect" | "error"
description: str               # Human-readable description
blocking_element_uid: str      # UID of the blocking element, if identifiable
```

---

## Knowledge Base

### How It Works

```
data/knowledge_base/
├── _common.yaml          # Universal: Ctrl+C, Ctrl+V, Alt+F4, common icons
├── chrome.yaml           # Chrome
├── vscode.yaml           # VS Code
├── discord.yaml          # Discord
├── slack.yaml            # Slack
├── notion.yaml           # Notion
├── telegram.yaml         # Telegram Desktop
├── wechat.yaml           # WeChat Desktop (微信)
├── dingtalk.yaml         # DingTalk (钉钉)
├── feishu.yaml           # Feishu / Lark (飞书)
├── wps.yaml              # WPS Office
├── explorer.yaml         # Windows Explorer
└── <your_app>.yaml       # ← Add yours here
```

### Contributing an App Profile

```yaml
# data/knowledge_base/slack.yaml
app: slack
aliases: ["slack desktop"]
process_name: "slack.exe"

shortcuts:
  ctrl+k: "Quick switcher"
  ctrl+shift+k: "Browse channels"
  ctrl+n: "New message"
  alt+up: "Previous channel"
  alt+down: "Next channel"

common_icons:
  - pattern: "compose_square"
    label: "New message"
    action: click
```

Commit. Push. Every agent using ACI now knows Slack's shortcuts. **Zero code changes.**

### Resolution Logic

App matching is fuzzy and multi-path:
- Exact filename → alias lookup → `difflib` fuzzy match (cutoff=0.6)
- Supports: Chinese names (`微信`), process names (`WeChat.exe`), window classes (`WeChatMainWndForPC`)
- `_common.yaml` is always merged — universal shortcuts available everywhere

---

## Comparison

| | **ACI** | Computer Use | Browser Use | UFO | Playwright MCP |
|---|---|---|---|---|---|
| **Web** | Structured (A11y + DOM) | Screenshot | Structured | No | Structured |
| **Desktop** | Structured (UIA + Vision) | Screenshot | No | UIA only | No |
| **Unified protocol** | Yes | No | No | No | No |
| **Typical latency** | 50-300ms | 3-10s | 100-500ms | 200-500ms | 50-200ms |
| **Element references** | Stable UIDs | Pixel coords | CSS selectors | UIA IDs | ARIA refs |
| **Community knowledge** | YAML profiles | None | None | None | None |
| **Interrupt handling** | MutationShield | None | Basic | None | None |
| **Persistent learning** | Pseudo-UIA cache | None | None | None | None |
| **Agent-ready skill** | SKILL.md | Prompt needed | Prompt needed | Prompt needed | Prompt needed |
| **Vision usage** | Last resort | Primary | None | None | None |

---

## API Reference

### REST Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/session` | Create session |
| `POST` | `/v1/perceive` | Get perception snapshot |
| `POST` | `/v1/action` | Execute action |
| `POST` | `/v1/screenshot` | Capture screenshot |
| `GET` | `/v1/sessions` | List active sessions |
| `DELETE` | `/v1/session/{id}` | Close session |
| `POST` | `/api/v1/knowledge/query` | Query knowledge base |
| `GET` | `/health` | Health check |

### Action Types

| Action | Fields | Description |
|--------|--------|-------------|
| `click` | `target_uid` | Physical mouse click at element center |
| `type` | `target_uid`, `value` | Clear field + type text with realistic delay |
| `press_key` | `value` | Keyboard press (`"Enter"`, `"Ctrl+S"`, `"Alt+F4"`) |
| `scroll` | `value` | Mouse wheel (`"up"` / `"down"`) |
| `wait` | `value` | Sleep N seconds |
| `hover` | `target_uid` | Mouse hover (desktop) |
| `launch_app` | `value` | Launch desktop app by name |
| `sequence` | `value` (JSON array) | Atomic multi-step in one round-trip |

---

## Safety & Privacy

- **PII Masking** — Regex-based detection of credit cards, SSNs, Chinese IDs, AWS keys, emails, phone numbers. Automatically masked in perception outputs.
- **Navigation Blocking** — `execute_js` blocks `window.location`, `window.open`, `location.href` patterns. Agents must use perceive→click, preventing JS-based navigation hijacking.
- **Kill Switch** — Emergency session termination handler.
- **I/O Mutex** — Per-session locking prevents race conditions from concurrent agent actions.

---

## Project Structure

```
aci/
├── core/                          # Central daemon
│   ├── server.py                  # FastAPI (port 11434)
│   ├── session_manager.py         # Session lifecycle + I/O mutex
│   ├── state_machine.py           # Task state machine + interrupt stack
│   ├── protocol_router.py         # WebSocket request-response routing
│   └── models/schemas.py          # Pydantic V2 contracts (the ACI spec)
│
├── bridges/                       # Execution workers (subprocesses)
│   ├── base_bridge.py             # IOpenClawBridge abstract interface
│   ├── web_bridge/                # Playwright-based
│   │   ├── executor.py            # Tiered extraction orchestrator
│   │   ├── a11y_extractor.py      # T0: CDP accessibility tree
│   │   ├── mutation_shield.py     # Interrupt detection
│   │   ├── hover_prober.py        # Hover-triggered discovery
│   │   ├── frame_manager.py       # iframe isolation
│   │   └── snapshot_formatter.py  # LLM-readable text output
│   ├── desktop_bridge/            # Windows UIA + Vision
│   │   ├── perception_fusion.py   # UIA + vision merge (IoU dedup)
│   │   ├── physical_input.py      # Hardware mouse/keyboard + DPI
│   │   ├── cursor_probe.py        # Cursor shape grid analysis
│   │   ├── fast_ocr.py            # Windows Media OCR
│   │   ├── vlm_identifier.py      # VLM bounding box generation
│   │   └── desktop_shield.py      # Desktop interrupt detection
│   └── cli_bridge/                # Terminal execution
│
├── memory_core/                   # Persistent learning
│   ├── knowledge_base.py          # YAML app knowledge loader
│   ├── muscle_memory.py           # OpenCV template cache
│   └── shortcut_graph.py          # Shortcut resolution
│
├── safety/                        # Privacy & failsafe
│   ├── data_masking.py            # PII masking
│   └── kill_switch.py             # Emergency abort
│
├── data/knowledge_base/           # YAML app profiles (community-extensible)
├── tests/                         # 95+ tests
├── scripts/                       # CLI client & launchers
├── SKILL.md                       # LLM skill definition (agent-ready)
└── requirements.txt
```

---

## Roadmap

- [ ] Linux desktop bridge (AT-SPI / DBus accessibility)
- [ ] macOS desktop bridge (NSAccessibility)
- [ ] Multi-tab web session isolation
- [ ] Mobile bridge (Android ADB + UIAutomator)
- [ ] Plugin system for custom bridge types
- [ ] ACI SDK (Python client library)
- [ ] Cloud-hosted daemon mode
- [ ] WebArena / OSWorld benchmark integration
- [ ] **100+ YAML app profiles** — PRs welcome

---

## Contributing

### No Code Required
- **Add a YAML app profile** — Teach ACI a new application. [See how](#contributing-an-app-profile)
- **Test with your apps** — Run ACI on software you use daily. Report what breaks.
- **Translate docs** — Help make ACI accessible globally

### Code Contributions
- **New bridge types** — Implement `IOpenClawBridge` for Linux, macOS, mobile
- **Better detection heuristics** — Improve tier triggers and fusion algorithms
- **New action types** — Extend `_ACTION_MAP` in executor classes

### Research
- **Benchmark ACI** — Run against WebArena, OSWorld, or your own evaluation suite
- **Write papers** — ACI's tiered extraction and perception fusion are publishable ideas

---

## Citation

If you use ACI in academic work:

```bibtex
@software{aci2025,
  title={ACI: Agent-Computer Interface},
  author={[Your Name]},
  year={2025},
  url={https://github.com/Leoooooli/ACI}
}
```

---

## License

[Apache License 2.0](LICENSE)

---

<p align="center">
  <strong>APIs connected developers to software and built the modern internet.<br>
  ACI connects AI agents to software. What will they build?</strong>
</p>
<p align="center">
  <sub>Star this repo if you believe agents deserve better than pixel-guessing.</sub>
</p>
