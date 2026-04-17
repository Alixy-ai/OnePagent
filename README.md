<div align="center">

<img src="logo.svg" alt="OnePagent" width="128" height="128" />

# OnePagent

### *One Page, Omnipotent Agent.*

**The browser-native workbench that lives in One Page.**

[![HTML](https://img.shields.io/badge/single--file-HTML-ff6b35?style=flat-square)](onepagent.html)
[![Zero Build](https://img.shields.io/badge/build-none-00ff88?style=flat-square)](#getting-started)
[![BYOK](https://img.shields.io/badge/keys-BYOK-5b8af5?style=flat-square)](#configuration)
[![License](https://img.shields.io/badge/license-MIT-aa66ff?style=flat-square)](#license)

**English** &nbsp;|&nbsp; [简体中文](README.zh.md)

</div>

---

OnePagent is a **single-file, zero-build, browser-native** AI agent workbench. Open one HTML and you get a fully-featured, internet-aware, programmable, extensible agent: multi-turn chat, tool calls, Python sandbox, web search, skills, memory compaction, file operations — all running on a single page.

> No backend. No `npm install`. No Docker. Just one `.html` that carries an entire universe.

---

## Highlights

- **Single-file deployment** — Drop `onepagent.html` on any static host or open it locally. No build, no dependency chain.
- **Multi-provider LLM** — Built-in support for Anthropic, OpenAI, DeepSeek, and any custom endpoint. Keys are injected by a Service Worker, never leak into URLs or logs.
- **Long context with auto-compaction** — Per-model context-window configuration; when approaching the limit, an LLM summarizer compresses history while preserving key decisions and code changes.
- **Tools & MCP** — File I/O, shell, code-gen, and form rendering are exposed as tools. Add your own MCP tools via URL endpoint or in-browser JS handler.
- **Python sandbox** — Execute Python snippets right in the browser via Pyodide, no server needed.
- **Web Search** — Tavily integration with `basic` / `advanced` depth modes; results are fed directly to the agent.
- **Skills system** — Install `.skill` / `.zip` packs, pull from GitHub, or create in-page. Each skill carries its own prompt + tools.
- **Conversation management** — Multiple sessions, auto-persistence, one-click export, token-based memory bar visualization.
- **Theming** — Dark / light themes driven by CSS variables; code highlighting swaps in sync.


---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    onepagent.html                           │
│  ┌───────────┐  ┌──────────┐  ┌────────────┐  ┌──────────┐  │
│  │  UI Shell │  │  Chat /  │  │  Tools /   │  │  Skills  │  │
│  │ (3-column)│  │  Streams │  │  MCP Bus   │  │ Registry │  │
│  └─────┬─────┘  └────┬─────┘  └─────┬──────┘  └────┬─────┘  │
│        │             │              │              │        │
│  ┌─────┴─────────────┴──────────────┴──────────────┴─────┐  │
│  │          Pretext Layout Engine (inlined)             │  │
│  │  markdown → blocks → lines → flowed DOM              │  │
│  └──────────────────────────────────────────────────────┘  │
│        │                                                    │
│  ┌─────┴───────┐   ┌──────────────┐   ┌─────────────────┐   │
│  │ Service     │   │ LocalStorage │   │ Pyodide         │   │
│  │ Worker      │   │ (settings,   │   │ (Python sandbox)│   │
│  │ (key inject)│   │  conv, skill)│   │                 │   │
│  └─────┬───────┘   └──────────────┘   └─────────────────┘   │
└────────┼─────────────────────────────────────────────────────┘
         │
         ▼
  ┌──────────────┐   ┌────────────┐   ┌──────────────┐
  │  Anthropic   │   │  OpenAI    │   │   Tavily     │
  │  DeepSeek    │   │  …         │   │  Web Search  │
  └──────────────┘   └────────────┘   └──────────────┘
```

---

## Getting Started

### 1. Clone

```bash
git clone <your-fork-url> onepagent
cd onepagent
```

### 2. Open

Double-click `onepagent.html`, or serve it from any static server:

```bash
# Python
python -m http.server 8000

# Node
npx serve .

# Any editor's Live Preview works too
```

Then visit `http://localhost:8000/onepagent.html`.

> **Service Worker note**: SWs register only over `https://` or `localhost`. When opened via `file://`, OnePagent falls back to direct `fetch` — fully functional, but keys appear in request headers (still sent only to your configured API endpoint).

### 3. Configure

Click **Settings** in the top bar:

| Field | Description |
|---|---|
| Provider | `anthropic` / `openai` / `deepseek` |
| API Endpoint | Auto-filled when provider changes; override for proxies |
| API Key | LLM key, stored in localStorage only |
| Models | **Fetch from API** pulls the full model list (unfiltered); multi-select to add. You can also type custom model ids. |
| Default Model | Model used for new conversations |
| Model Context Lengths | Per-model context window (`model=tokens` per line) |
| Tavily API Key | Required for Web Search |

Takes effect immediately — no reload needed.

---

## Configuration

### Where is my state?

All configuration and conversation data lives in the browser's `localStorage`:

| Key | Contents |
|---|---|
| `ba_settings` | Provider / Endpoint / Keys / Models / Context Lengths |
| `ba_selected_model` | Currently-selected model |
| `ba_ws_config` | Tavily search depth & result count |
| `ba_theme` | `dark` / `light` |
| `ba_conv_*` | Conversation history |

Data is **never** uploaded to any third-party server. Clearing browser data = resetting OnePagent.

### Privacy Model

- LLM requests: browser → (Service Worker injects key) → your configured endpoint
- Tavily requests: same path
- Other resources: static CDN files only (marked / highlight.js / pyodide / fonts)

**OnePagent itself has no server and no relay of any kind.**

---

## Extending

### Add a Skill

Left panel **SKILLS → + Create**: fill in name, trigger description, SKILL.md body, and optional custom tools. Or install a `.skill` / `.zip` pack, or pull one from a GitHub URL.

### Add a custom tool (MCP)

Right panel **TOOLS → + MCP**:

- **MCP URL mode** — tool calls POST to your MCP endpoint
- **Browser JS mode** — `handler` is a JS snippet executed in-browser; `input` is the tool's argument object

```js
// Example: in-browser handler
return 'Echo: ' + input.text;
```

### Pretext layout engine

Message rendering uses the inlined **Pretext engine** — canvas-based precise text measurement + bidi / CJK line breaking / soft hyphens / geometric layout for lists and quotes. Compared to `innerHTML = marked.parse(...)`, it guarantees code blocks don't overflow, long URLs wrap correctly, and CJK punctuation never orphans.

---

## Roadmap

- [ ] Skill marketplace (community skill aggregator)
- [ ] IndexedDB conversation archive (bypass localStorage limits)
- [ ] WebRTC multi-device sync
- [ ] Local models (via `window.ai` / WebGPU)

---

## Contributing

This is a single-file project — fork it, edit it, send a PR. Guidelines:

- Keep `onepagent.html` runnable on its own
- Avoid dependencies that require a build step

---

## License

MIT © OnePagent contributors

---

<div align="center">

**One Page. Omnipotent Agent.**

Built for people who believe a single HTML can still do everything.

</div>
