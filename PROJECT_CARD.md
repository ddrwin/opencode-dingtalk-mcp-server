# DingTalk MCP Server · 项目卡

> 项目特定常识库，用于 OpenCode 会话加载。
>
> **维护者**：ddrwin · Sisyphus
> **仓库**：https://github.com/ddrwin/opencode-dingtalk-mcp-server
> **工作目录**：`E:\CodeBase\opencode-dingtalk-mcp-server\`
> **备份目录**：`D:\同步文件夹\软件\3 - AI 生产力\5. 项目Projects\opencode-dingtalk-mcp-server\`

---

## 一、项目定位

让 OpenCode（Sisyphus）与钉钉双向实时通信。你在外头钉钉发消息，家里的 Sisyphus 收到并处理，结果回你手机。

**核心约束**：国内环境，不需要翻墙，不需要公网 IP。

---

## 二、系统架构

```
钉钉 App (手机)
    ↕ WebSocket (Stream 模式)
钉钉 Stream 云端
    ↕ WebSocket
┌─────────────────────────────────┐
│  DingTalk MCP Server             │
│  (node index.mjs)                │
│  ├─ 连接钉钉 Stream SDK          │
│  ├─ 接收消息 → 存 pending/*.json │
│  └─ MCP Stdio Server (待完善)    │
└─────────────────────────────────┘
         ↕ Sisyphus 读取文件
┌─────────────────────────────────┐
│  Sisyphus (OpenCode)             │
│  ├─ 检查 pending/ 目录           │
│  ├─ 处理消息内容                 │
│  └─ 通过 webhook POST 回钉钉     │
└─────────────────────────────────┘
```

### 数据流

```
钉钉发消息 → Stream WebSocket → MCP 服务器 → pending/*.json
                                                    ↓
                                            Sisyphus 读取
                                                    ↓
                                            处理 & 回复
                                                    ↓
                                    HTTP POST sessionWebhook → 钉钉
```

---

## 三、部署方式

### 3.1 MCP 服务器（消息接收端）

**启动方式**：开机自启（启动文件夹）

```
C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\dingtalk-mcp.bat
```

内容：
```bat
@echo off
start /B node "E:\CodeBase\opencode-dingtalk-mcp-server\index.mjs"
```

**验证运行**：`Get-Process -Name "node"` 应该能看到进程。

### 3.2 OpenCode 配置（可选，待修复）

`C:\Users\Administrator\.config\opencode\opencode.json`：

```json
{
  "mcp": {
    "dingtalk": {
      "type": "local",
      "command": [
        "node",
        "E:\\CodeBase\\opencode-dingtalk-mcp-server\\index.mjs"
      ],
      "enabled": true
    }
  }
}
```

> ⚠️ 当前 OpenCode 桌面版可能不自启 MCP 子进程，以启动文件夹方式为准。

---

## 四、配置文件

`.env`（位于项目根目录，已 gitignore）：

```env
DINGTALK_CLIENT_ID=dingfz7oia3mic7i9fhq
DINGTALK_CLIENT_SECRET=qW-HCYqhSvF-suKC460TB9aJtzmQMf211qw_1jPgqKo43clKHL8Z1WphaQYO6Zhp
OPENCODE_SERVER_URL=http://127.0.0.1:4096
```

---

## 五、项目结构

```
opencode-dingtalk-mcp-server/
├── index.mjs              ← 唯一源码（~600 行，单文件）
├── package.json           ← 依赖配置
├── .env                   ← 钉钉凭证（本地，不提交）
├── .env.example           ← 环境变量模板
├── .gitignore
├── README.md              ← 项目文档
├── PROJECT_CARD.md        ← 本文件（项目卡）
├── CONTRIBUTING.md
├── LICENSE                ← MIT
├── pending/               ← 待处理消息目录（运行时生成，不提交）
├── node_modules/
└── package-lock.json
```

### 关键依赖

| 包 | 版本 | 用途 |
|---|---|---|
| `dingtalk-stream-sdk-nodejs` | ^2.0.4 | 钉钉官方 Stream SDK（WebSocket 双向通信） |
| `@modelcontextprotocol/sdk` | ^1.0.0 | MCP 协议 SDK（暴露工具接口） |
| `@opencode-ai/sdk` | ^1.2.4 | OpenCode SDK（未使用，API 服务未启动） |
| `got` | ^14.6.6 | HTTP 客户端 |
| `lru-cache` | ^11.2.6 | LRU 缓存 |
| `p-queue` | ^9.1.0 | 异步队列 |

---

## 六、运行说明

### 6.1 启动流程

```bash
cd E:\CodeBase\opencode-dingtalk-mcp-server
node index.mjs
```

启动后日志：
```
📊 性能配置: 并发=3, 缓存=1000
🚀 DingTalk MCP Server 已启动 (v2.1.0)
🔌 连接钉钉 Stream 服务...
✅ 钉钉 Stream 连接成功
📱 现在可以在钉钉中给机器人发送消息了
```

### 6.2 消息处理流程（Sisyphus）

每次 Sisyphus 上线时，应执行：

1. 检查 `pending/` 目录是否有 `*.json` 文件
2. 按时间顺序读取所有文件
3. 解析 `content` 和 `sessionWebhook`
4. 处理消息内容
5. 通过 HTTP POST 回复 sessionWebhook
6. 删除已处理的文件

> ⚠️ 重要：回复后必须删除 pending 文件，否则下次会重复处理。
> ⚠️ 重要：读文件后用 `[System.Text.Encoding]::UTF8.GetString()` 解码。

### 6.3 回复钉钉消息

```powershell
$body = '{"msgtype":"text","text":{"content":"消息内容"}}'
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)
Invoke-RestMethod -Uri $webhookUrl -Method Post -ContentType "application/json; charset=utf-8" -Body $utf8
```

> ⚠️ 必须用 `[System.Text.Encoding]::UTF8.GetBytes()` 转 UTF-8 字节流，
> 不能直接用 `-Body $body`（PowerShell 默认用 GB2312）。

---

## 七、已知问题

### 7.1 OpenCode 桌面版不自启 MCP

`opencode.json` 的 `mcp` 配置在桌面版 OpenCode 中可能导致崩溃或不被加载。
当前绕过：通过启动文件夹自启 node 进程。

### 7.2 不是真正的实时推送

当前机制是 Sisyphus 主动轮询 `pending/` 目录，不是钉钉消息实时推送到对话。
真正的实时需要 OpenCode 的 HTTP API 服务运行（`@opencode-ai/sdk` 需要连接 `opencode serve`）。

### 7.3 启动文件夹进程可能被误杀

如果手工运行其他 node 进程管理命令（如 `Stop-Process -Name "node"`），可能误杀 MCP 服务器。
每次重启后检查 `Get-Process -Name "node"` 确认存活。

---

## 八、后续可扩展

- [ ] 合并钉钉官方 MCP 工具能力（发卡片、查日程、待办）
- [ ] OpenCode MCP 集成修复（解决 `mcp` 配置崩溃问题）
- [ ] 真正的实时推送（研究 OpenCode API server 方案）
- [ ] WebUI 管理界面
- [ ] 多会话支持（允许多人同时对话）

---

## 九、决策记录

| 日期 | 决策 |
|---|---|
| 2026-05-09 | Fork `yewh/opencode-dingtalk-mcp-server` 到 ddrwin 账号 |
| 2026-05-09 | 编码问题：PowerShell 默认 GB2312，所有 I/O 必须显式 UTF-8 |
| 2026-05-09 | 部署方案：启动文件夹（bat）而非任务计划程序 |
| 2026-05-09 | 消息机制：文件轮询而非 SDK 推送（因 OpenCode 桌面版无 API） |
