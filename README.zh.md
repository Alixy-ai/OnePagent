<div align="center">

<img src="logo.svg" alt="OnePagent" width="128" height="128" />

# OnePagent

### *One Page, Omnipotent Agent.*

**The browser-native workbench that lives in One Page.**

[![HTML](https://img.shields.io/badge/single--file-HTML-ff6b35?style=flat-square)](onepagent.html)
[![Zero Build](https://img.shields.io/badge/build-none-00ff88?style=flat-square)](#getting-started)
[![BYOK](https://img.shields.io/badge/keys-BYOK-5b8af5?style=flat-square)](#configuration)
[![License](https://img.shields.io/badge/license-MIT-aa66ff?style=flat-square)](#license)

[English](README.md) &nbsp;|&nbsp; **简体中文**

</div>

---

OnePagent 是一个**单文件、零构建、浏览器原生**的 AI Agent 工作台。打开一个 HTML，你就拥有了一整个可联网、可编程、可扩展的智能体：多轮对话、工具调用、Python 沙箱、网页检索、技能系统、记忆压缩、文件操作——全部运行在一张页面里。

> 没有后端，没有 npm install，没有 Docker。一张 `.html`，自带整个宇宙。

---

## Highlights

- **单文件部署** — 将 `onepagent.html` 丢进任意静态主机 / 本地打开即可运行，无构建、无依赖链
- **多 LLM 供应商** — 内置 Anthropic、OpenAI、DeepSeek，可自定义 Endpoint；API 层通过 Service Worker 安全注入，密钥不进 URL、不进日志
- **长上下文与自动压紧** — 每个模型独立的 context window 配置，接近上限时自动 LLM 摘要压缩，保留关键决策与代码改动
- **工具与 MCP** — 文件读写、命令执行、代码生成、表单渲染均以工具形态暴露；支持自定义 MCP 工具（URL 端点或浏览器内 JS handler）
- **Python 沙箱** — 通过 Pyodide 在浏览器内执行 Python 片段，无需服务端
- **Web Search** — Tavily 集成，`basic` / `advanced` 两档搜索深度，结果直接喂给 Agent
- **Skills 系统** — 可安装 `.skill` / `.zip` 包、从 GitHub 拉取、或直接在界面创建；每个 Skill 自带 prompt + 工具
- **会话管理** — 多会话、自动持久化、一键导出、按 token 的记忆条带可视化
- **主题切换** — 深 / 浅双主题，CSS 变量驱动，代码高亮同步切换


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

双击 `onepagent.html`，或用任意静态服务器托管：

```bash
# Python
python -m http.server 8000

# Node
npx serve .

# 任意编辑器的 Live Preview 亦可
```

然后访问 `http://localhost:8000/onepagent.html`。

> **Service Worker 限制**：SW 仅在 `https://` 或 `localhost` 下注册。本地 `file://` 打开时会自动降级为直连 fetch，功能完整但密钥会出现在请求头（仍仅发往你配置的 API 端点）。

### 3. Configure

点击顶栏 **Settings**：

| 字段 | 说明 |
|---|---|
| Provider | `anthropic` / `openai` / `deepseek` |
| API Endpoint | 切换 Provider 时自动填入官方地址；可改为代理 |
| API Key | LLM 密钥，仅存 localStorage |
| Models | **Fetch from API** 拉取完整模型列表（不过滤），多选加入；支持手动输入自定义 model id |
| Default Model | 会话默认模型 |
| Model Context Lengths | 每个模型独立的上下文长度（`model=tokens` 逐行） |
| Tavily API Key | Web Search 功能所需 |

保存后即刻生效，无需刷新。

---

## Configuration

### Where is my state?

配置与会话数据全部留在浏览器本地。轻量项存 `localStorage`，完整会话历史（消息、VFS、渲染 HTML、元数据）存在 **IndexedDB**（数据库 `ba_conversations`，对象存储 `data` / `meta`），不受 localStorage 5–10 MB 容量限制。

| localStorage Key | 内容 |
|---|---|
| `ba_settings` | Provider / Endpoint / Keys / Models / Context Lengths |
| `ba_selected_model` | 当前选中模型 |
| `ba_active_conv` | 上次打开的会话 id |
| `ba_ws_config` | Tavily 搜索深度与结果数 |
| `ba_theme` | `dark` / `light` |

数据**不会**上传任何第三方服务器。清理浏览器数据（localStorage + IndexedDB）= 清零 OnePagent。

### Privacy Model

- LLM 请求：浏览器 → （Service Worker 注入密钥）→ 你配置的 Endpoint
- Tavily 请求：同上
- 其它资源：只有 CDN 静态文件（marked / highlight.js / pyodide / 字体）

**OnePagent 本身没有服务端，也不经过任何中转。**

---

## Extending

### 添加 Skill

点击左栏 **SKILLS → + Create**，填入名称、触发描述、SKILL.md 正文与可选的自定义工具。或通过 `.skill` / `.zip` 安装、从 GitHub URL 拉取。

### 添加自定义工具（MCP）

右栏 **TOOLS → + MCP**：

- **MCP URL 模式**：工具调用会 POST 到你的 MCP endpoint
- **Browser JS 模式**：`handler` 是一段在浏览器内执行的 JS，`input` 是工具参数对象

```js
// 示例：浏览器内 handler
return 'Echo: ' + input.text;
```

### Pretext 布局引擎

消息渲染走的是内联的 **Pretext 引擎**——基于 Canvas 的精确文本度量 + 双向文字 / CJK 断行 / 软连字符 / 列表与引用的几何布局。相比 `innerHTML = marked.parse(...)`，它能保证代码块不溢出、长 URL 正确截断、中文标点不孤悬。

---

## Roadmap

- [ ] 插件市场（社区 Skills 聚合）
- [x] IndexedDB 会话归档（替换 localStorage 上限）
- [ ] WebRTC 多端同步
- [ ] 本地模型（via `window.ai` / WebGPU）

---

## Contributing

这是一个单文件项目——欢迎直接 fork、改、PR。建议：

- 保持 `onepagent.html` 可独立运行
- 不引入需构建步骤的依赖

---

## License

MIT © OnePagent contributors

---

<div align="center">

**One Page. Omnipotent Agent.**

Built for people who believe a single HTML can still do everything.

</div>
