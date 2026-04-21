<div align="center">

<img src="logo.svg" alt="OnePagent" width="128" height="128" />

# OnePagent

### *One Page, Omnipotent Agent.*

**浏览器原生的单文件 AI 智能体工作台。**

[![HTML](https://img.shields.io/badge/single--file-HTML-ff6b35?style=flat-square)](onepagent.html)
[![Zero Build](https://img.shields.io/badge/build-none-00ff88?style=flat-square)](#getting-started)
[![BYOK](https://img.shields.io/badge/keys-BYOK-5b8af5?style=flat-square)](#configuration)
[![License](https://img.shields.io/badge/license-MIT-aa66ff?style=flat-square)](#license)

[English](README.md) &nbsp;|&nbsp; **简体中文**

[在线体验](https://onepagent.top) &nbsp;&middot;&nbsp; [一键部署](#deploy) &nbsp;&middot;&nbsp; [使用配置](#configuration)

</div>

---

打开一个 HTML 文件，即可获得一个可联网、可编程、可扩展的完整 AI 智能体——多轮对话、工具调用、Python 沙箱、网页检索、技能系统、上下文压缩、长期记忆、文件操作、云端同步，全部在一张页面中运行。

> 没有后端，没有 npm install，没有 Docker。一张 `.html`，自带整个宇宙。

---

## 预览

![OnePagent Preview](https://jsd.onmicrosoft.cn/gh/mydracula/image@master/20260421/188f31edc79848ff9ed581bc3b5339ff.png)

---

## Highlights

| 能力 | 说明 |
|---|---|
| 单文件部署 | `onepagent.html` 放入任意静态主机或本地打开即可运行 |
| 多 LLM 供应商 | Anthropic / OpenAI / DeepSeek，可自定义 Endpoint；密钥由 Service Worker 安全注入 |
| 推理程度控制 | 顶栏内联选择器，支持 `off / minimal / low / medium / high / xhigh` 六档 |
| 长上下文压缩 | 每模型独立 context window，接近上限时自动 LLM 摘要压缩 |
| 长期记忆 | 可选开启，跨会话持久保留事实 / 偏好 / 事件 / 技能，自动提取 + Agent 工具调用 + 手动增删，标签与关键词检索 |
| MCP 服务器 | 粘贴 `mcpServers` JSON 即可导入（`streamable_http` / `sse`），支持 Bearer 鉴权与 CORS 代理 |
| Plan Mode | Agent 先只读调研，生成 Markdown 计划供审批，批准后才执行写入 |
| TodoWrite | Agent 主动维护可视任务清单（pending / in-progress / completed） |
| Hooks | 用户自定义 JS 处理函数，挂在 Agent 6 个生命周期事件上 |
| Python 沙箱 | 通过 Pyodide 在浏览器内执行 Python |
| Web Search | Tavily 集成，`basic` / `advanced` 两档搜索深度 |
| Skills 系统 | `.skill` / `.zip` 安装，GitHub 拉取，或界面直接创建 |
| 会话管理 | 多会话、文件夹分组、拖拽排序、IndexedDB 持久化、一键导出 |
| Cloud Sync | S3 兼容桶增量同步，可选 AES-256-GCM 端到端加密 |

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
  │  Anthropic   │  │  OpenAI    │  │  Tavily  │  │ 你的 S3 桶  │
  │  DeepSeek    │  │  …         │  │          │  │ AWS/R2/MinIO│
  └──────────────┘  └────────────┘  └──────────┘  └─────────────┘
```

---

## Getting Started

```bash
git clone https://github.com/sligter/OnePagent.git
cd OnePagent
```

双击 `onepagent.html`，或用任意静态服务器托管：

```bash
python -m http.server 8000
# 或
npx serve .
```

访问 `http://localhost:8000/onepagent.html`，点击顶栏 **Settings** 配置 Provider / API Key / 模型，保存后即刻生效。

> **Service Worker 限制**：SW 仅在 `https://` 或 `localhost` 下注册。`file://` 打开时自动降级为直连 fetch。

---

## Deploy

OnePagent 是纯静态站点，任意静态主机均可运行：

<div align="center">

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https%3A%2F%2Fgithub.com%2Fsligter%2FOnePagent&project-name=onepagent&repository-name=onepagent)
&nbsp;
[![Deploy on Zeabur](https://zeabur.com/button.svg)](https://zeabur.com/new)
&nbsp;
[![Deploy to Cloudflare Pages](https://img.shields.io/badge/Deploy-Cloudflare%20Pages-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)](https://dash.cloudflare.com/?to=/:account/pages/new)

</div>

| 平台 | 步骤 |
|---|---|
| **Vercel** | 点击按钮 → 登录 → Deploy，约 10 秒完成 |
| **Zeabur** | 点击按钮 → 选择 fork 仓库 → 自动识别静态站点 → Generate Domain |
| **Cloudflare Pages** | Connect to Git → Framework preset: None, Build command: 留空, Output: `/` → Deploy |
| **GitHub Pages** | Settings → Pages → Source: GitHub Actions → 推送即自动部署 |

> 所有数据存在浏览器本地，不同域名之间不共享。跨域名/跨设备同步请用 [Cloud Sync](#cloud-sync)。

---

## Cloud Sync

通过 S3 兼容桶备份和跨设备同步，**不经过 OnePagent 任何服务器**。

- 支持 AWS S3 / Cloudflare R2 / Backblaze B2 / MinIO
- SHA-256 内容寻址，增量同步，二进制去重
- 可选 AES-256-GCM 端到端加密（PBKDF2-SHA256 / 200k 迭代）
- LLM 密钥不同步，始终留在本机

配置路径：**Settings → Cloud Sync** → 填写 Endpoint / Region / Bucket / 凭据 → Test connection → Push now。

| 后端 | Endpoint | Region | Path-style |
|---|---|---|---|
| AWS S3 | `https://s3.<region>.amazonaws.com` | 真实 region | 推荐 |
| Cloudflare R2 | `https://<account_id>.r2.cloudflarestorage.com` | `auto` | 必须 |
| MinIO | `https://<your-host>` | 自定义 | 必须 |
| Backblaze B2 | `https://s3.<region>.backblazeb2.com` | 如 `us-west-002` | 必须 |

---

## Memory

跨对话持久化的长期记忆系统，与上下文压缩（压的是当前会话窗口）完全不同——此处记录的是**跨会话可复用的事实**。

> **公式**：Memory = 信息 + 时间标签 + 可搜索关系（标签）

- **存储**：独立 IndexedDB（`ba_memories`），参与 Cloud Sync 增量同步
- **默认关闭**：在 **Settings → Memory** 勾选启用；可单独关闭自动提取
- **五种类型**：`fact` / `preference` / `event` / `skill` / `note`
- **三种来源**：`auto`（LLM 自动提取）/ `tool`（Agent 主动调用）/ `manual`（手动添加）
- **自动提取**：每轮助手回答后追加一次轻量级 LLM 调用，只保留真正耐用的事实，可指定专用提取模型
- **召回策略**：按**时效 + 标签重叠 + 关键词匹配**综合排序，取 Top-N（可配置，默认 8）注入系统提示
- **Agent 工具**：`memory_save` / `memory_search` / `memory_update` / `memory_forget`，支持 `supersedesIds` 让新记忆替代过时记忆
- **Memory Viewer**：顶栏按钮进入，支持搜索、按类型/来源过滤、标签云、JSON 导入导出、显示已废弃记录
- **废弃而非删除**：被 supersedes 的记录保留历史但不再召回，可在 Viewer 中查看

---

## Configuration

配置与会话全部存于浏览器本地（`localStorage` + `IndexedDB`），不上传任何第三方服务器。

**Privacy Model**：浏览器 → Service Worker 注入密钥 → 你配置的 API Endpoint。OnePagent 本身没有服务端。

---

## Contributing

单文件项目——欢迎 fork、修改、PR。

- 保持 `onepagent.html` 可独立运行
- 不引入需构建步骤的依赖
- `index.html` 仅做跳转，请勿加逻辑

---

## Star History

<div align="center">

[![Star History Chart](https://api.star-history.com/svg?repos=sligter/OnePagent&type=Date)](https://star-history.com/#sligter/OnePagent&Date)

</div>

---

## 友链

- [Linux Do](https://linux.do/) — 新的理想型社区

---

## License

MIT (c) OnePagent contributors

---

<div align="center">

**One Page. Omnipotent Agent.**

Built for people who believe a single HTML can still do everything.

</div>
