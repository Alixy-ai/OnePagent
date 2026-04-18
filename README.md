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

[Live Demo](https://onepagent.top) &nbsp;·&nbsp; [Deploy Your Own](#deploy) &nbsp;·&nbsp; [Configuration](#configuration)

</div>

---

OnePagent is a **single-file, zero-build, browser-native** AI agent workbench. Open one HTML and you get a fully-featured, internet-aware, programmable, extensible agent: multi-turn chat, tool calls, Python sandbox, web search, skills, memory compaction, file operations, cloud sync — all running on a single page.

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
- **Conversation management** — Multiple sessions, auto-persistence to IndexedDB, one-click export, token-based memory bar visualization.
- **☁ Cloud Sync (S3)** — Sync conversations, skills, and settings to your own S3-compatible bucket (AWS, Cloudflare R2, MinIO, Backblaze B2). Content-addressed + incremental: only changed objects upload. Optional AES-256-GCM end-to-end encryption. Binary files are deduplicated by SHA-256.
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
│  ┌─────┴───────┐ ┌──────────────┐ ┌───────────┐ ┌────────┐  │
│  │ Service     │ │ LocalStorage │ │ Pyodide   │ │ S3     │  │
│  │ Worker      │ │ + IndexedDB  │ │ (Python)  │ │ SigV4  │  │
│  │ (key inject)│ │ (all state)  │ │           │ │ Client │  │
│  └─────┬───────┘ └──────────────┘ └───────────┘ └───┬────┘  │
└────────┼──────────────────────────────────────────────┼─────┘
         │                                              │
         ▼                                              ▼
  ┌──────────────┐  ┌────────────┐  ┌──────────┐  ┌─────────────┐
  │  Anthropic   │  │  OpenAI    │  │  Tavily  │  │ Your bucket │
  │  DeepSeek    │  │  …         │  │          │  │ (AWS/R2/MinIO│
  └──────────────┘  └────────────┘  └──────────┘  │  /B2/…)     │
                                                   └─────────────┘
```

---

## Getting Started

### 1. Clone

```bash
git clone https://github.com/sligter/OnePagent.git
cd OnePagent
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

Then visit `http://localhost:8000/onepagent.html` (or just `/` if the redirect `index.html` is present).

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
| ☁ Cloud Sync | Optional S3-compatible bucket for cross-device sync — see [Cloud Sync](#cloud-sync) |

Takes effect immediately — no reload needed.

---

## Deploy

OnePagent is a pure static site — `onepagent.html` plus a redirect `index.html`. It runs on any static host. One-click deploys:

<div align="center">

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https%3A%2F%2Fgithub.com%2Fsligter%2FOnePagent&project-name=onepagent&repository-name=onepagent)
&nbsp;
[![Deploy on Zeabur](https://zeabur.com/button.svg)](https://zeabur.com/new)
&nbsp;
[![Deploy to Cloudflare Pages](https://img.shields.io/badge/Deploy-Cloudflare%20Pages-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)](https://dash.cloudflare.com/?to=/:account/pages/new)

</div>

### Vercel

1. Click **Deploy with Vercel** above.
2. Sign in (GitHub / GitLab / email), authorize the clone, pick an owner.
3. Accept the default project name or rename it, click **Deploy**.
4. After ~10 seconds you get a URL like `https://onepagent-xxx.vercel.app`. Open it — the redirect page sends you to `/onepagent.html` automatically.

No build command, no framework detection, no environment variables needed. To use a custom domain: **Project → Settings → Domains → Add**.

### Zeabur

1. Click **Deploy on Zeabur** above and sign in.
2. **Create Project** → **Deploy New Service** → **Git** → authorize GitHub → select your `OnePagent` fork.
3. Zeabur detects the static content and serves it automatically (no `Dockerfile`, no build).
4. Enable a public domain: **Service → Networking → Generate Domain**.
5. Access at `https://<service>.zeabur.app/`.

### Cloudflare Pages

1. Click **Deploy to Cloudflare Pages** above and sign in (or go to [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages** → **Create application** → **Pages** tab).
2. **Connect to Git** → authorize GitHub → pick your `OnePagent` fork.
3. Build settings:
   - **Framework preset**: *None*
   - **Build command**: *(leave empty)*
   - **Build output directory**: `/`
4. **Save and Deploy**. First build takes ~30 seconds.
5. Access at `https://<project>.pages.dev/`.

### GitHub Pages (pre-configured)

This repo ships with a GitHub Actions workflow at [.github/workflows/deploy.yml](.github/workflows/deploy.yml) that publishes the site on every push to `main`. On your fork:

1. **Settings → Pages → Source: GitHub Actions**.
2. Push any commit — the workflow renames `onepagent.html` to `index.html` in the build and deploys.
3. Access at `https://<you>.github.io/OnePagent/`.

> **Note on state**: Because OnePagent is BYOK and all data lives in the browser, different deployed domains do **not** share state — each domain has its own IndexedDB. Use [Cloud Sync](#cloud-sync) to share conversations, skills, and settings between deploys and devices.

---

## Cloud Sync

Back up and synchronize across browsers / devices via any S3-compatible bucket. **No OnePagent server is involved** — the signed PUT / GET goes directly from your browser to your bucket.

### Supported backends

AWS S3 · Cloudflare R2 · Backblaze B2 · MinIO · any server speaking the S3 API.

### How it works

```
  <prefix>/manifest.json     ← small index: hashes of every object
  <prefix>/objects/<sha256>  ← conversations / skills / settings, one file each
  <prefix>/blobs/<sha256>    ← content-addressed binary files (images, attachments)
```

- **Incremental** — each object is content-addressed by SHA-256. Push compares local hashes to the remote manifest and uploads only what's new or changed. Unchanged conversations are skipped entirely.
- **Binary dedup** — the same image attached in two conversations is stored once.
- **Encryption (optional)** — set a passphrase and every object (including binaries) is AES-256-GCM encrypted; the key is derived via PBKDF2-SHA256 (200k iterations). Bucket contents look like noise.
- **Progress UI** — upload / download progress bar + phase labels in the Sync menu.
- **No LLM keys synced** — your LLM / Tavily keys are always device-local (each machine re-enters them).

### Setup

1. **Open Settings** → scroll to **☁ Cloud Sync**.
2. Fill in **Endpoint**, **Region**, **Bucket**, **Access Key ID**, **Secret Access Key**.
3. (Optional) Set an **Encryption Passphrase** — losing it means losing the backup, so remember it.
4. Keep **Use path-style URLs** checked for MinIO / R2 (required); unchecking enables virtual-host style for AWS.
5. Click **Test connection** → ✅ means auth + CORS are good.
6. If CORS fails, click **Show CORS config** and paste the JSON into your bucket settings:

```json
[{"AllowedOrigins":["*"],"AllowedMethods":["GET","PUT","HEAD"],"AllowedHeaders":["*"],"ExposeHeaders":["ETag"]}]
```

(Replace `"*"` with your actual deploy origin in production.)

7. Save. Back in the top bar, click **☁ Sync → Push now**.
8. On a second device: configure the same bucket + passphrase, click **Pull now**, confirm — everything appears.

### Per-backend quick config

| Backend | Endpoint | Region | Path-style |
|---|---|---|---|
| AWS S3 | `https://s3.<region>.amazonaws.com` | real region, e.g. `us-east-1` | ✓ recommended |
| Cloudflare R2 | `https://<account_id>.r2.cloudflarestorage.com` | `auto` | ✓ required |
| MinIO | `https://<your-minio-host>` | whatever you configured | ✓ required |
| Backblaze B2 | `https://s3.<region>.backblazeb2.com` | e.g. `us-west-002` | ✓ required |

---

## Configuration

### Where is my state?

All configuration and conversation data lives in the browser. Lightweight keys go to `localStorage`; full conversation history (messages, VFS, rendered HTML, metadata) and skill files live in **IndexedDB**, so there is no 5–10 MB localStorage cap.

| localStorage Key | Contents |
|---|---|
| `ba_settings` | Provider / Endpoint / Keys / Models / Context Lengths |
| `ba_selected_model` | Currently-selected model |
| `ba_active_conv` | Last-opened conversation id |
| `ba_ws_config` | Tavily search depth & result count |
| `ba_theme` | `dark` / `light` |
| `ba_skills` | Skill metadata (files live in IDB) |
| `ba_mcp_tools` | Custom MCP tool definitions |
| `ba_disabled_tools` | Tool on/off state |
| `ba_s3_sync` | S3 bucket config (endpoint, creds, passphrase) |
| `ba_s3_last_sync` | Last successful sync timestamp |
| `ba_s3_last_pass_hash` | Hash of last-known passphrase (detects rotation) |

| IndexedDB Database | Stores | Contents |
|---|---|---|
| `ba_conversations` | `data` / `meta` | Full conversation data + metadata index |
| `ba_skill_files` | `files` | Skill file payloads (text + binary) |

Data is **never** uploaded to any third-party server (unless you configure Cloud Sync to your own bucket). Clearing browser data (localStorage + IndexedDB) = resetting OnePagent.

### Privacy Model

- LLM requests: browser → (Service Worker injects key) → your configured endpoint
- Tavily requests: same path
- S3 sync requests: browser → (SigV4-signed from browser) → your bucket
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

- [x] Skill marketplace (community skill aggregator)
- [x] IndexedDB conversation archive (bypass localStorage limits)
- [x] Cloud Sync to S3-compatible buckets (incremental, content-addressed, optional E2E encryption)
- [ ] Global memory system (cross-conversation long-term recall)
- [ ] Local models (via `window.ai` / WebGPU)

---

## Contributing

This is a single-file project — fork it, edit it, send a PR. Guidelines:

- Keep `onepagent.html` runnable on its own
- Avoid dependencies that require a build step
- The repo root `index.html` is a redirect shim for platforms that don't rewrite `/` — don't put real logic there

---

## License

MIT © OnePagent contributors

---

<div align="center">

**One Page. Omnipotent Agent.**

Built for people who believe a single HTML can still do everything.

</div>
