# 🚀 Drive-Agent (通用型网盘智能整理管家)

<p align="center">
  <strong>面向 AI Coding Agent 的通用型网盘智能整理、资源生命周期调度与影视库分类治理框架</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Agent%20Ready-AGENTS.md-blue?style=flat-square" alt="AGENTS.md Ready">
  <img src="https://img.shields.io/badge/Python-3.8+-green?style=flat-square" alt="Python Version">
  <img src="https://img.shields.io/badge/Protocol-WebDAV%20%7C%20REST%20%7C%20gRPC%20%7C%20FUSE-orange?style=flat-square" alt="Protocols">
  <img src="https://img.shields.io/badge/License-MIT-purple?style=flat-square" alt="License">
</p>

---

## 📖 项目简介 (Overview)

**Drive-Agent** 是一个专为 AI Coding Agent（如 Codex、ChatGPT、Claude Code、Cursor、Google Antigravity 等）量身打造的**存储中立型网盘自主整理与资源管理框架**。

面对杂乱无章的网盘文件、重复下载的资源、层级冗余的目录以及不符合流媒体刮削标准的视频，Drive-Agent 通过**规范化指令约束（AGENTS.md）**与**分级流水线架构**，让 AI 能够安全、高效、零误删地完成海量云端资产的自动化治理。

---

## ✨ 核心特性 (Key Features)

- 🌐 **存储中立与通用抽象 (Vendor-Agnostic)**：
  - 摆脱对单一网盘或特定客户端的绑定，支持 **WebDAV、REST API、gRPC、S3、FUSE 虚拟挂载** 及各类聚合挂载服务（OpenList、AList、CloudDrive、Rclone）。
- 🤖 **Agent 原生规范支持 (Agent-Native)**：
  - 遵循开放 [AGENTS.md](https://agents.md) 标准规范，无需复杂配置，Agent 启动即自动加载操作准则与安全边界。
- ⚖️ **分级双模执行流水线 (Dual-Tier Pipeline)**：
  - **模式 A（交互即时响应）**：针对 1~20 个文件的检索、移动、重命名及配额检查，优先调用原生 MCP / CLI 工具，零冗余开销。
  - **模式 B（高通量批处理）**：针对千百级海量文件，采用 **“本地快照缓存 `.cache/` ➔ 95% 本地轻量正则匹配 ➔ 5% 模型语义决策 ➔ 受控并发与指数退避”** 架构，彻底解决上下文溢出与 API 限频难题。
- 🎬 **标准化媒体库刮削整理 (Plex/Emby/Jellyfin)**：
  - 严格遵循现代影视库规范（电影、剧集、季/集、动漫剧场版/OVA 分类）。
  - **关键文件强制保护**：多语言字幕（`.srt`, `.ass`）、NFO 元数据及专辑封面无论体积大小，受白名单强制保护，严禁误删。
- 🛡️ **严格安全底线与 100% 回滚保障 (Zero-Risk & 100% Rollback)**：
  - **敏感凭证零泄露**：Token、密码、环境变量严格脱敏，禁止输出至任何日志或控制台。
  - **沙箱脚本即用即删**：临时脚本严格限定于系统 `/tmp/` 目录，执行完毕强制销毁。
  - **强制 Dry-Run 预检**：批处理超 20 项文件强制展示预览表格并校验。
  - **可逆执行清单**：所有批处理全量记录至 `manifests/manifest_<timestamp>.json`，支持一键无损反向回滚。
- 🩺 **通用排障与自愈体系 (Universal Troubleshooting)**：
  - 配套完善的 [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)，覆盖 HTTP/gRPC/FUSE 通用错误码、挂载点冲突、反代 Buffering 流控阻断与指数退避重试算法。

---

## 🏗️ 架构概览 (Architecture)

```
                       [用户指令 / 整理任务]
                                 │
                 任务规模是否 > 20 文件 或 需全盘递归？
                                / \
                               /   \
                       [否]   /     \   [是]
                             ▼       ▼
                    ┌──────────────────┐    ┌───────────────────────────────────┐
                    │ 模式 A: 交互执行 │    │ 模式 B: Python 批处理流水线       │
                    │ MCP / 原生 CLI   │    │ 1. 分页检索与本地索引快照 (.cache)│
                    │ (1 ~ 20 个文件)  │    │ 2. 规则前置分类 (95% 本地 / 5% AI)│
                    │ 毫秒级反馈       │    │ 3. 线程池并发 + 清单日志写入      │
                    └──────────────────┘    └───────────────────────────────────┘
                                                              │
                                                              ▼
                                                    ┌──────────────────┐
                                                    │ 审计清单与回滚保障│
                                                    │ manifests/*.json │
                                                    │ (支持一键逆向对调)│
                                                    └──────────────────┘
```

---

## 📁 目录结构 (Directory Structure)

```text
.
├── AGENTS.md               # 根目录：通用型 Agent 行为准则、存储抽象原语与执行策略规范
├── TROUBLESHOOTING.md      # 根目录：通用排障与跨协议错误速查手册 (WebDAV/REST/gRPC/FUSE)
├── README.md               # 项目主说明文档
├── LICENSE                 # MIT 开源许可证
└── PikPak/                 # 📦 PikPak 专属套件（遵循 AGENTS.md 分层覆盖机制）
    ├── AGENTS.md           # PikPak 专属规范（PAT 权限、根目录 "" 约定、25% 配额熔断等）
    └── TROUBLESHOOTING.md  # PikPak 专属排障手册（7000 文件种子限制、Cloudflare 1010 防护等）
```

---

## 🎬 影视与资料分类体系 (Taxonomy Standards)

Drive-Agent 默认按照业界主流媒体服务器标准自动规范目录与文件名：

```text
/
├── _Inbox/                         # 待分类收集箱 / 临时下载缓冲池
├── Movies/                         # 电影资料库
│   └── Inception (2010)/
│       ├── Inception (2010) [1080p][H265].mkv
│       └── Inception (2010) [1080p][H265].zh-CN.srt
├── TV Shows/                       # 电视剧与连载番剧
│   └── Breaking Bad/
│       └── Season 01/
│           └── Breaking Bad - S01E01 - Pilot.mp4
├── Documents/                      # 文档与电子书 (按分类/年份归档)
│   └── Finance/2026/
├── Software/                       # 软件安装包 (按系统隔离: Windows/macOS/Linux/Android)
├── Archives/                       # 归档压缩包 (zip, rar, 7z, tar.gz)
└── Torrents/                       # 种子与元数据暂存
```

---

## 🔄 批处理清单与一键回滚示例 (Rollback Workflow)

每次批处理均会生成标准审计清单文件：

```json
{
  "manifest_id": "manifest_20260928_220000",
  "created_at": "2026-09-28T22:00:00+08:00",
  "total_items": 42,
  "successful": 42,
  "failed": 0,
  "operations": [
    {
      "item_id": "file_123456",
      "action": "move",
      "original_path": "/_Inbox/[SubGroup] Show - 01 [1080p].mp4",
      "target_path": "/TV Shows/Show/Season 01/Show - S01E01 [1080p].mp4",
      "status": "success"
    }
  ]
}
```

如需撤销变更，只需提示 Agent：
> *“请根据 `manifests/manifest_20260928_220000.json` 执行一键回滚。”*

Agent 将自动反向调换 `original_path` 与 `target_path`，执行前置安全碰撞探测后逐一恢复。

---

## 🚀 使用方式 (How to Use)

### 1. 复制到工作区
将本仓库的 `AGENTS.md`（以及 `TROUBLESHOOTING.md`）直接复制到你管理网盘的项目根目录下即可。

- 如果你的项目涉及 **PikPak**，可直接将 `PikPak/` 目录一同复制或单独使用其中的规则。
- 如果你的工具支持全局规则（如 Codex 的 `~/.codex/AGENTS.md`），也可直接作为全局默认规范。

### 2. 直接对 AI 下达指令
AI Agent（Codex、Cursor、Claude Code、Antigravity 等）启动时会自动读取并加载该规范。直接用自然语言吩咐它干活：
- *“帮我扫描 `/_Inbox` 目录，按照 Emby 规范整理里面的电影和美剧，先给我看 Dry-Run 预览。”*
- *“清理网盘中残留的广告文本和空文件夹，注意保留所有字幕文件。”*
- *“查询当前各个存储挂载点的空间占用与容量状态。”*

---

## 📄 开源许可证 (License)

本项目采用 [MIT License](./LICENSE) 开源许可协议。欢迎提交 Issue 与 Pull Request 共同完善！
