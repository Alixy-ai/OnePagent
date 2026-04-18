<div align="center">

<img src="logo.svg" alt="OnePagent" width="128" height="128" />

# OnePagent

### *One Page, Omnipotent Agent.*

**浏览在一页里的浏览器原生智能体工作台。**

[![HTML](https://img.shields.io/badge/single--file-HTML-ff6b35?style=flat-square)](onepagent.html)
[![Zero Build](https://img.shields.io/badge/build-none-00ff88?style=flat-square)](#getting-started)
[![BYOK](https://img.shields.io/badge/keys-BYOK-5b8af5?style=flat-square)](#configuration)
[![License](https://img.shields.io/badge/license-MIT-aa66ff?style=flat-square)](#license)

[English](README.md) &nbsp;|&nbsp; **简体中文**

[在线体验](https://onepagent.top) &nbsp;·&nbsp; [一键部署](#deploy) &nbsp;·&nbsp; [使用配置](#configuration)

</div>

---

OnePagent 是一个**单文件、零构建、浏览器原生**的 AI Agent 工作台。打开一个 HTML，你就拥有了一整个可联网、可编程、可扩展的智能体：多轮对话、工具调用、Python 沙箱、网页检索、技能系统、记忆压缩、文件操作、云端同步——全部运行在一张页面里。

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
- **会话管理** — 多会话、IndexedDB 自动持久化、一键导出、按 token 的记忆条带可视化
- **☁ 云端同步（S3）** — 将会话、技能、设置同步到你自己的 S3 兼容桶（AWS、Cloudflare R2、MinIO、Backblaze B2）。按内容哈希增量同步——仅上传变化的对象，二进制文件按 SHA-256 去重；可选 AES-256-GCM 端到端加密
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

### 1. Clone

```bash
git clone https://github.com/sligter/OnePagent.git
cd OnePagent
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

然后访问 `http://localhost:8000/onepagent.html`（如果有重定向 `index.html`，直接访问 `/` 亦可）。

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
| ☁ Cloud Sync | 可选的 S3 兼容桶，用于跨设备同步——详见 [Cloud Sync](#cloud-sync) |

保存后即刻生效，无需刷新。

---

## Deploy

OnePagent 是纯静态站点——`onepagent.html` + 一个重定向 `index.html`。任意静态主机均可运行。一键部署：

<div align="center">

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https%3A%2F%2Fgithub.com%2Fsligter%2FOnePagent&project-name=onepagent&repository-name=onepagent)
&nbsp;
[![Deploy on Zeabur](https://zeabur.com/button.svg)](https://zeabur.com/new)
&nbsp;
[![Deploy to Cloudflare Pages](https://img.shields.io/badge/Deploy-Cloudflare%20Pages-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)](https://dash.cloudflare.com/?to=/:account/pages/new)

</div>

### Vercel

1. 点击上方 **Deploy with Vercel**。
2. 登录 Vercel（GitHub / GitLab / 邮箱），授权 clone 本仓库，选择所有者。
3. 保留默认项目名或重命名，点击 **Deploy**。
4. 大约 10 秒后你会得到 `https://onepagent-xxx.vercel.app` 形式的地址。打开后 `index.html` 会自动重定向到 `/onepagent.html`。

无需构建命令、无需框架识别、无需环境变量。绑定自定义域名：**Project → Settings → Domains → Add**。

### Zeabur

1. 点击上方 **Deploy on Zeabur** 并登录。
2. **Create Project** → **Deploy New Service** → **Git** → 授权 GitHub → 选择你 fork 的 `OnePagent` 仓库。
3. Zeabur 自动识别为静态站点并直接托管（无 `Dockerfile`、无构建步骤）。
4. 启用公网域名：**Service → Networking → Generate Domain**。
5. 访问 `https://<service>.zeabur.app/`。

### Cloudflare Pages

1. 点击上方 **Deploy to Cloudflare Pages** 并登录（或直接进入 [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages** → **Create application** → **Pages** 标签页）。
2. **Connect to Git** → 授权 GitHub → 选择你 fork 的 `OnePagent` 仓库。
3. 构建设置：
   - **Framework preset**：*None*
   - **Build command**：*（留空）*
   - **Build output directory**：`/`
4. **Save and Deploy**。首次构建约 30 秒。
5. 访问 `https://<project>.pages.dev/`。

### GitHub Pages（已预配置）

本仓库的 [.github/workflows/deploy.yml](.github/workflows/deploy.yml) 已配置自动部署：每次推送到 `main` 会将 `onepagent.html` 复制为 `index.html` 并发布。在你的 fork 上：

1. **Settings → Pages → Source：GitHub Actions**。
2. 推送任意提交，workflow 自动部署。
3. 访问 `https://<你>.github.io/OnePagent/`。

> **关于数据**：OnePagent 是 BYOK 模式，所有数据存在浏览器本地，**不同部署域名之间不共享**——每个域名有独立的 IndexedDB。跨域名 / 跨设备同步请用 [Cloud Sync](#cloud-sync)。

---

## Cloud Sync

通过任意 S3 兼容桶备份和跨设备同步。**不经过 OnePagent 任何服务器**——签名后的 PUT / GET 直接从浏览器发到你的桶。

### 支持的后端

AWS S3 · Cloudflare R2 · Backblaze B2 · MinIO · 任意兼容 S3 API 的服务。

### 工作原理

```
  <prefix>/manifest.json     ← 小型索引：所有对象的哈希
  <prefix>/objects/<sha256>  ← 会话 / 技能 / 设置，每项一个文件
  <prefix>/blobs/<sha256>    ← 内容寻址的二进制文件（图片、附件）
```

- **增量同步** — 每个对象按 SHA-256 内容寻址。Push 时对比本地与远端 manifest，只上传新增或变化的内容；未改动的会话完全跳过
- **二进制去重** — 两个会话里的同一张图片在桶里只存一份
- **端到端加密（可选）** — 设置 passphrase 后所有对象（含二进制）用 AES-256-GCM 加密，密钥由 PBKDF2-SHA256（20 万次迭代）派生；桶里的内容对他人是白噪声
- **进度指示** — Sync 菜单内嵌进度条和阶段标签
- **LLM 密钥不同步** — LLM / Tavily 密钥始终留在本机，每台设备独立填写

### 配置步骤

1. **打开 Settings** → 滚动到 **☁ Cloud Sync**。
2. 填写 **Endpoint**、**Region**、**Bucket**、**Access Key ID**、**Secret Access Key**。
3. （可选）填写 **Encryption Passphrase**——丢失等于丢失备份，请妥善保管。
4. MinIO / R2 必须勾选 **Use path-style URLs**；AWS 若想用 virtual-host 风格可取消。
5. 点击 **Test connection** → ✅ 表示认证 + CORS 均通过。
6. 若 CORS 失败，点击 **Show CORS config** 把 JSON 粘到桶的 CORS 配置里：

```json
[{"AllowedOrigins":["*"],"AllowedMethods":["GET","PUT","HEAD"],"AllowedHeaders":["*"],"ExposeHeaders":["ETag"]}]
```

（生产环境请把 `"*"` 换成你真实部署的 origin。）

7. 保存。回到顶栏点击 **☁ Sync → Push now**。
8. 在第二台设备：填入相同的桶信息 + passphrase，点击 **Pull now** 确认——所有内容即刻到位。

### 各后端快速对照

| 后端 | Endpoint | Region | Path-style |
|---|---|---|---|
| AWS S3 | `https://s3.<region>.amazonaws.com` | 真实 region，如 `us-east-1` | ✓ 推荐 |
| Cloudflare R2 | `https://<account_id>.r2.cloudflarestorage.com` | `auto` | ✓ 必须 |
| MinIO | `https://<你的 MinIO 主机>` | 你配置的 region | ✓ 必须 |
| Backblaze B2 | `https://s3.<region>.backblazeb2.com` | 如 `us-west-002` | ✓ 必须 |

---

## Configuration

### Where is my state?

配置与会话数据全部留在浏览器本地。轻量项存 `localStorage`，完整会话历史（消息、VFS、渲染 HTML、元数据）以及技能文件存在 **IndexedDB**，不受 localStorage 5–10 MB 容量限制。

| localStorage Key | 内容 |
|---|---|
| `ba_settings` | Provider / Endpoint / Keys / Models / Context Lengths |
| `ba_selected_model` | 当前选中模型 |
| `ba_active_conv` | 上次打开的会话 id |
| `ba_ws_config` | Tavily 搜索深度与结果数 |
| `ba_theme` | `dark` / `light` |
| `ba_skills` | 技能元数据（文件体存于 IDB） |
| `ba_mcp_tools` | 自定义 MCP 工具定义 |
| `ba_disabled_tools` | 工具开关状态 |
| `ba_s3_sync` | S3 桶配置（endpoint / 凭据 / passphrase） |
| `ba_s3_last_sync` | 最近一次成功同步的时间戳 |
| `ba_s3_last_pass_hash` | 最近 passphrase 的哈希（用于检测轮换） |

| IndexedDB 数据库 | 对象存储 | 内容 |
|---|---|---|
| `ba_conversations` | `data` / `meta` | 完整会话数据 + 元数据索引 |
| `ba_skill_files` | `files` | 技能文件体（文本 + 二进制） |

数据**不会**上传任何第三方服务器（除非你主动配置 Cloud Sync 到自己的桶）。清理浏览器数据（localStorage + IndexedDB）= 清零 OnePagent。

### Privacy Model

- LLM 请求：浏览器 → （Service Worker 注入密钥）→ 你配置的 Endpoint
- Tavily 请求：同上
- S3 同步请求：浏览器 → （浏览器端 SigV4 签名）→ 你的桶
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

- [x] 插件市场（社区 Skills 聚合）
- [x] IndexedDB 会话归档（替换 localStorage 上限）
- [x] S3 兼容云端同步（增量、内容寻址、可选端到端加密）
- [ ] WebRTC 点对点实时同步
- [ ] 本地模型（via `window.ai` / WebGPU）

---

## Contributing

这是一个单文件项目——欢迎直接 fork、改、PR。建议：

- 保持 `onepagent.html` 可独立运行
- 不引入需构建步骤的依赖
- 仓库根部的 `index.html` 只是给不支持 `/` 重写的平台做跳转——请勿在里面放真实逻辑

---

## License

MIT © OnePagent contributors

---

<div align="center">

**One Page. Omnipotent Agent.**

Built for people who believe a single HTML can still do everything.

</div>
