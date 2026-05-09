# DingTalk MCP Server

让钉钉 AI 助手自动回复你的消息 —— 你在外头发钉钉，家里的 AI 收到并处理，结果回你手机。

---

## 📖 项目背景

### 为什么有这个项目

ddrwin 的需求很简单：

> **场景**：人在户外，用手机钉钉给家里的 OpenCode AI agent（Sisyphus）发消息，Sisyphus 收到后执行任务，结果回传到钉钉。
>
> **约束**：人在国内，不能依赖需要翻墙的方案（Telegram、Discord 等排除）。国内可用的平台：钉钉、飞书、企业微信、微信、QQ。
>
> **目标**：双向实时通信，不需要公网 IP，不需要 ngrok。

### 方案评估过程

在 GitHub 上调研了国内 IM 平台的 MCP Server 方案：

| 方案 | Stars | 双向通信 | 维护者 | 结论 |
|---|---|---|---|---|
| `yewh/opencode-dingtalk-mcp-server` | 2 | ✅ Stream 模式 | 个人开发者 | ✅ **Fork 自建** |
| `open-dingtalk/dingtalk-mcp`（钉钉官方） | 17 | ❌ 单向推送 | 钉钉官方 | ❌ 不是聊天方案 |
| `loonghao/wecom-bot-mcp-server`（企微） | 82 | ❌ 单向 | 第三方 | ❌ 需求不对口 |
| 飞书 MCP（`ztxtxwd/feishu-mcp-server`） | 83 | ✅ | 第三方 | ❌ ddrwin 不想用飞书 |

**最终决策：Fork `yewh/opencode-dingtalk-mcp-server` 并自维护。**

理由：
1. 它解决的问题正好是"钉钉 ↔ OpenCode 双向实时通信"
2. 使用钉钉官方 Stream SDK（`dingtalk-stream-sdk-nodejs`）作为通信底座，WebSocket 长连接，**不需要公网 IP**
3. 代码量极小（1 个文件，~18KB），维护成本极低
4. MIT 协议，可自由 fork 和修改
5. 底座全是官方 SDK（钉钉官方、MCP 官方、OpenCode 官方），底座不动它就不会废

### 决策记录

| 日期 | 决策 |
|---|---|
| 2026-05-09 | Fork `yewh/opencode-dingtalk-mcp-server` 到 ddrwin 账号 |
| 2026-05-09 | 项目根目录：`D:\同步文件夹\软件\3 - AI 生产力\5. 项目Projects\opencode-dingtalk-mcp-server\` |
| 2026-05-09 | 首次计划：先跑通双向通信，后续可合并钉钉官方 MCP 工具能力 |

---

## 🏗️ 系统架构

```
┌──────────────┐      WebSocket       ┌──────────────────┐
│  你的手机     │ ◄──────────────────► │  钉钉 Stream 云端 │
│  (钉钉 App)   │                      └────────┬─────────┘
└──────────────┘                               │
                                      WebSocket │ Stream
                                               │
                                     ┌─────────▼──────────┐
                                     │  DingTalk MCP      │
                                     │  Server (本服务)    │
                                     │  index.mjs          │
                                     └─────────┬──────────┘
                                               │
                                    ┌──────────┴──────────┐
                                    ▼                     ▼
                            ┌──────────────┐   ┌──────────────────┐
                            │  DeepSeek AI │   │  OpenCode (记录)  │
                            │  (自动回复)   │   │  (可选同步)       │
                            └──────────────┘   └──────────────────┘
```

### 核心组件

| 组件 | 作用 | 底座 |
|---|---|---|
| DingTalk Stream 客户端 | 与钉钉建立 WebSocket 长连接，收发消息 | `dingtalk-stream-sdk-nodejs`（钉钉官方） |
| 身份文档加载引擎 | 启动时加载共享文档，构建 AI 身份体系（**有记忆的 DeepSeek**） | `fs`（本地文件系统） |
| DeepSeek AI | 自动处理消息并生成回复（**自动回复的核心**） | DeepSeek API |
| 会话记忆持久化 | 每个会话保存对话历史到本地文件，重启不丢 | `sessions/*.json` |
| OpenCode SDK 客户端 | 同步消息到 OpenCode 做记录（可选，不影响自动回复） | `@opencode-ai/sdk`（OpenCode 官方） |
| MCP 服务器 | 桥接以上所有组件 | `@modelcontextprotocol/sdk`（MCP 官方） |

### 消息流程

```
你在钉钉发消息
   ↓
钉钉 Stream 云端（WebSocket）
   ↓
DingTalk MCP Server
  ├─ 消息解析 + 去重检查
  ├─ 保存会话 Webhook（用于回复）
  ├─ ★ 调用 DeepSeek AI 自动生成回复
  ├─ （可选）同步到 OpenCode 做记录
  ↓
DingTalk MCP Server 通过 Webhook 发回钉钉
  ↓
你手机收到回复
```

---

## 🧠 共享身份体系

这个 MCP Server 不是"又一个 AI"，而是 **你已有 AI 身份的延伸**。

### 架构设计

```
                  ┌──────────────────────────────────┐
                  │   工程哲学 / 工程哲学 V3.3.md      │
                  │   (用户维护的总纲 — 唯一源头)       │
                  └──────────┬──────────────────────┘
                             │ 摘取 / 提取
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
   ┌────────────┐    ┌────────────┐    ┌──────────────────┐
   │ Claude Code│    │  OpenCode  │    │ DingTalk MCP      │
   │ (我)       │    │            │    │ Server（小小机器人）│
   │ memory/    │    │ memory/    │    │ DOCUMENTS_PATH    │
   └────────────┘    └────────────┘    └──────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             ▼
                    ┌──────────────────┐
                    │ 同一套身份系统     │
                    │ IDENTITY V0.2    │
                    │ 工程哲学 V3.3     │
                    │ 纪律手册 V3.3     │
                    │ AGENTS V3.3      │
                    │ VISION           │
                    └──────────────────┘
```

**三个 AI 分身，同一套体系。** 文档在工程哲学目录维护，各个 AI 按需加载。

### 记忆机制

| 机制 | 说明 |
|------|------|
| **身份记忆** | 启动时从 `DOCUMENTS_PATH` 加载核心文档，构建 AI 身份体系 |
| **对话记忆** | 每个会话保存最近 20 轮对话到 `sessions/*.json`，重启不丢 |
| **无状态 API** | DeepSeek API 本身无状态，记忆靠本地文件持久化 |

重启 MCP Server 后，之前的对话上下文不会丢失。

---

## 🚀 快速开始

### 前置要求

- Node.js >= 18.0.0
- 一个钉钉企业内部应用（用于获取 Client ID 和 Client Secret）
- OpenCode 已安装并可用

### 第一步：获取钉钉应用凭证

1. 访问 [钉钉开放平台](https://open.dingtalk.com/)
2. 创建一个企业内部应用
3. 在应用详情页获取 **Client ID**（原 AppKey）和 **Client Secret**（原 AppSecret）
4. 在应用权限管理中添加**企业内机器人发送消息**权限
5. 发布应用

### 第二步：安装依赖

```bash
cd DingTalk-MCP-Server（本目录）
npm install
```

### 第三步：配置环境变量

复制 `.env.example` 为 `.env`，填入你的钉钉应用凭证：

```bash
cp .env.example .env
```

编辑 `.env`：

```env
# 钉钉应用凭证
DINGTALK_CLIENT_ID=your_client_id_here
DINGTALK_CLIENT_SECRET=your_client_secret_here

# OpenCode 地址（如果 OpenCode serve 在其他端口则修改）
OPENCODE_SERVER_URL=http://127.0.0.1:4096
```

### 第四步：配置 OpenCode

在 OpenCode 配置文件 `~/.config/opencode/opencode.json` 中添加：

```json
{
  "mcp": {
    "dingtalk": {
      "type": "local",
      "command": [
        "node",
        "D:\\同步文件夹\\软件\\3 - AI 生产力\\5. 项目Projects\\opencode-dingtalk-mcp-server\\index.mjs"
      ],
      "enabled": true
    }
  }
}
```

### 第五步：启动

先确保 OpenCode 处于运行状态，然后启动 MCP Server：

```bash
npm start
```

或者通过 OpenCode 自动管理（配置 MCP 后 OpenCode 会在需要时自动拉起）。

---

## 🛠️ 可用工具

| 工具 | 功能 |
|---|---|
| `dingtalk_send_message` | 发送文本/Markdown 消息到钉钉会话 |
| `dingtalk_get_stats` | 获取服务器运行统计（消息数、处理时间、内存等） |

---

## ⚙️ 配置参考

### 核心配置项

| 配置项 | 说明 | 默认值 |
|---|---|---|
| `DINGTALK_CLIENT_ID` | 钉钉应用 Client ID | — |
| `DINGTALK_CLIENT_SECRET` | 钉钉应用 Client Secret | — |
| `OPENCODE_SERVER_URL` | OpenCode 服务地址 | `http://127.0.0.1:4096` |
| `DOCUMENTS_PATH` | 共享身份文档目录（指向 memory/ 或 工程哲学/） | —（基础身份） |

### 缓存配置（源码中调整）

| 参数 | 说明 | 默认值 |
|---|---|---|
| `CACHE.PROCESSED_MESSAGES_MAX` | 消息去重缓存上限 | 1000 条 |
| `CACHE.SESSIONS_MAX` | 会话缓存上限 | 100 个 |
| `CACHE.WEBHOOKS_MAX` | Webhook 缓存上限 | 100 个 |

### Stream 模式说明

本服务使用钉钉 **Stream 模式**（WebSocket 长连接），而非传统的 Webhook 模式：

- ✅ **不需要公网 IP** — 连接是客户端主动发起的
- ✅ **不需要配置防火墙** — 不监听外部端口
- ✅ **自动重连** — 断线后指数退避重连（最多 10 次）
- ✅ **实时双向** — WebSocket 全双工通信

---

## 🔧 维护指南

### 项目结构

```
opencode-dingtalk-mcp-server/
├── index.mjs          ← 唯一源码文件（~18KB，核心逻辑 ~250 行）
├── package.json       ← 依赖和脚本配置
├── .env.example       ← 环境变量模板
├── .env               ← 本地环境变量（已 gitignore）
├── sessions/          ← 对话历史持久化（重启不丢）
├── README.md          ← 本文件
├── CONTRIBUTING.md    ← 贡献指南
└── LICENSE            ← MIT 协议
```

### 依赖关系

```
index.mjs
├── @modelcontextprotocol/sdk        ← MCP 官方（维护稳定）
├── @opencode-ai/sdk                 ← OpenCode 官方（维护稳定）
├── dingtalk-stream-sdk-nodejs       ← 钉钉官方（维护稳定）
├── got                              ← HTTP 客户端（成熟）
├── lru-cache                        ← 缓存库（成熟）
└── p-queue                          ← 队列（成熟）
```

所有核心依赖均为**官方维护的产品级 SDK**，上层封装极薄。只要底座不动，本项目的维护量就很小。

### 后续可扩展方向

- [ ] 合并钉钉官方 MCP 工具能力（发卡片、查日程、操作通讯录）
- [ ] 增加更多消息类型支持（图片、文件、语音）
- [ ] WebUI 管理界面
- [ ] 多会话隔离

---

## 📄 许可证

MIT License — 见 `LICENSE` 文件。

## 🔗 相关链接

- [钉钉开放平台](https://open.dingtalk.com/)
- [钉钉 Stream 模式文档](https://open.dingtalk.com/document/orgapp/stream-mode)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [OpenCode](https://opencode.ai/)
- [原项目 yewh/opencode-dingtalk-mcp-server](https://github.com/yewh/opencode-dingtalk-mcp-server)
