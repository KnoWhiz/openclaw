# OpenClaw 核心创新技术架构深度解析

## 目录

1. [项目概述](#项目概述)
2. [核心设计理念](#核心设计理念)
3. [Gateway 中心化架构](#gateway-中心化架构)
4. [Agent 运行时系统](#agent-运行时系统)
5. [多渠道消息系统](#多渠道消息系统)
6. [会话管理机制](#会话管理机制)
7. [多智能体路由系统](#多智能体路由系统)
8. [工具与技能系统](#工具与技能系统)
9. [浏览器控制系统](#浏览器控制系统)
10. [Canvas 可视化工作区](#canvas-可视化工作区)
11. [语音唤醒与对话系统](#语音唤醒与对话系统)
12. [插件扩展架构](#插件扩展架构)
13. [安全防护体系](#安全防护体系)
14. [部署与运维](#部署与运维)
15. [技术创新总结](#技术创新总结)

---

## 项目概述

### 基本信息

**OpenClaw** (原名 Clawdbot / Moltbot) 是一个开源的个人 AI 助手系统，由奥地利开发者 Peter Steinberger 于 2025 年 11 月创建。项目采用 MIT 许可证，目前在 GitHub 上已获得超过 200,000 颗星。

**核心定位**: 一个运行在本地设备上的自主 AI 代理网关，能够在多个消息平台上执行真实操作，而不仅仅是生成文本。

**技术栈**:
- 主要语言: TypeScript (ESM)
- 运行环境: Node.js 22+
- 包管理: pnpm / bun
- 测试框架: Vitest
- 构建工具: tsdown / tsx

### 项目演进历程

```
Warelay → Clawdbot → Moltbot → OpenClaw
                    (2025.11)    (2026.02)
```

---

## 核心设计理念

### 1. 本地优先 (Local-First)

OpenClaw 的核心哲学是 **"本地优先"**，所有数据和计算都在用户自己的设备上进行:

- **数据主权**: 用户数据永不离开本地控制
- **隐私保护**: 无需依赖第三方云服务
- **离线能力**: 支持本地模型运行 (Ollama, DeepSeek)

### 2. 模型无关 (Model-Agnostic)

系统支持任意 AI 模型提供商:

- **云端模型**: Claude, GPT, Gemini, DeepSeek
- **本地模型**: Ollama, LM Studio
- **混合模式**: 支持模型故障转移和负载均衡

### 3. 渠道解耦 (Channel Decoupled)

消息渠道与 AI 模型完全解耦，可以独立切换:

```
┌─────────────────────────────────────────────────────────┐
│                    消息渠道层                            │
│  WhatsApp │ Telegram │ Discord │ Slack │ iMessage │ ... │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   Gateway 控制平面                       │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    AI 模型层                             │
│    Claude │ GPT │ Gemini │ Ollama │ DeepSeek │ ...      │
└─────────────────────────────────────────────────────────┘
```

### 4. 单一真相源 (Single Source of Truth)

Gateway 作为系统的唯一控制平面，管理所有:
- 会话状态
- 路由决策
- 渠道连接
- 工具执行

---

## Gateway 中心化架构

### 架构概览

OpenClaw 采用 **单体 Gateway 架构**，而非微服务设计。所有功能集成在一个持久化的 Node.js 进程中。

```
                    ┌─────────────────────────────────────┐
                    │            Gateway                   │
                    │      ws://127.0.0.1:18789           │
                    │                                      │
                    │  ┌─────────────────────────────────┐│
                    │  │     1. 渠道适配器 (Adapters)     ││
                    │  │   标准化多平台消息格式            ││
                    │  └─────────────────────────────────┘│
                    │                │                     │
                    │  ┌─────────────────────────────────┐│
                    │  │     2. 会话管理器 (Sessions)     ││
                    │  │   隔离对话上下文                  ││
                    │  └─────────────────────────────────┘│
                    │                │                     │
                    │  ┌─────────────────────────────────┐│
                    │  │     3. 消息队列 (Queue)          ││
                    │  │   序列化 Agent 执行              ││
                    │  └─────────────────────────────────┘│
                    │                │                     │
                    │  ┌─────────────────────────────────┐│
                    │  │     4. Agent 运行时             ││
                    │  │   模型-工具反馈循环               ││
                    │  └─────────────────────────────────┘│
                    │                │                     │
                    │  ┌─────────────────────────────────┐│
                    │  │     5. 控制平面 (Control)        ││
                    │  │   WebSocket API                  ││
                    │  └─────────────────────────────────┘│
                    │                                      │
                    └─────────────────────────────────────┘
```

### WebSocket 协议

Gateway 使用简洁的 WebSocket 协议进行通信:

| 帧类型 | 方向 | 格式 |
|--------|------|------|
| Request | 客户端 → Gateway | `{type: "req", id, method, params}` |
| Response | Gateway → 客户端 | `{type: "res", id, ok, payload\|error}` |
| Event | Gateway → 客户端 | `{type: "event", event, payload, seq?}` |

**关键约束**: 连接的第一帧必须是 `connect` 帧，否则连接将被强制关闭。

### 热重载机制

Gateway 支持配置热重载，包含四种模式:

| 模式 | 行为 |
|------|------|
| `off` | 不进行配置重载 |
| `hot` | 仅应用热安全更改 |
| `restart` | 需要时重启 |
| `hybrid` (默认) | 安全时热应用，否则重启 |

### Hub-and-Spoke 拓扑

Gateway 作为中心枢纽，连接多个外围组件:

```
                      ┌─────────────────┐
                      │   Pi Agent      │
                      │   (RPC Mode)    │
                      └────────┬────────┘
                               │
┌──────────────┐     ┌─────────┴─────────┐     ┌──────────────┐
│   CLI        │────▶│                   │◀────│  WebChat UI  │
│  (openclaw)  │     │     Gateway       │     │              │
└──────────────┘     │                   │     └──────────────┘
                     │ ws://127.0.0.1:   │
┌──────────────┐     │      18789        │     ┌──────────────┐
│   macOS App  │────▶│                   │◀────│  iOS/Android │
│              │     └───────────────────┘     │    Nodes     │
└──────────────┘                               └──────────────┘
```

---

## Agent 运行时系统

### Pi-Mono 集成

OpenClaw 内嵌了 **pi-mono** 代理运行时，但会话管理、发现和工具接线由 OpenClaw 自行控制。

**关键特性**:
- 无独立的 pi-coding agent 运行时
- 不读取 `~/.pi/agent` 或 `<workspace>/.pi` 设置
- 会话转录存储为 JSONL 格式

### Agent 循环 (Agent Loop)

完整的 Agent 循环包含以下阶段:

```
┌──────────────────────────────────────────────────────────────┐
│                       Agent Loop                              │
│                                                               │
│   1. 消息接收 (Intake)                                        │
│          │                                                    │
│          ▼                                                    │
│   2. 上下文组装 (Context Assembly)                            │
│          │                                                    │
│          ▼                                                    │
│   3. 模型推理 (Model Inference)                               │
│          │                                                    │
│          ▼                                                    │
│   4. 工具执行 (Tool Execution)                                │
│          │                                                    │
│          ▼                                                    │
│   5. 流式回复 (Streaming Replies)                             │
│          │                                                    │
│          ▼                                                    │
│   6. 持久化 (Persistence)                                     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 工作区配置文件

Agent 行为通过 Markdown 文件定义:

| 文件 | 用途 |
|------|------|
| `AGENTS.md` | 操作指令和"记忆" |
| `SOUL.md` | 人格、边界、语气 |
| `TOOLS.md` | 用户维护的工具说明 |
| `BOOTSTRAP.md` | 首次运行仪式 (完成后删除) |
| `IDENTITY.md` | Agent 名称/风格/表情 |
| `USER.md` | 用户档案和首选地址 |

### Heartbeat 守护进程

OpenClaw 的创新特性之一是 **自主任务执行**:

- 每 30 分钟 (可配置) 自动执行
- 读取 `HEARTBEAT.md` 检查列表
- 判断是否需要采取行动
- 无需用户提示即可执行任务

这使得 OpenClaw 能够执行定时监控、收件箱检查和工作流触发。

---

## 多渠道消息系统

### 支持的渠道

OpenClaw 支持广泛的消息渠道:

**核心渠道**:
- WhatsApp (via Baileys)
- Telegram (via grammY)
- Discord (via discord.js)
- Slack (via Bolt)
- Google Chat (via Chat API)
- Signal (via signal-cli)
- iMessage (via imsg / BlueBubbles)

**扩展渠道 (插件)**:
- Microsoft Teams
- Matrix
- Zalo / Zalo Personal
- Mattermost
- WebChat

### 渠道适配器架构

```typescript
interface ChannelPlugin {
  id: string;
  meta: {
    id: string;
    label: string;
    docsPath: string;
    blurb: string;
    aliases: string[];
  };
  capabilities: { chatTypes: ["direct" | "group"] };
  config: {
    listAccountIds: (cfg) => string[];
    resolveAccount: (cfg, accountId) => AccountConfig;
  };
  outbound: {
    deliveryMode: "direct" | "queue";
    sendText: (params) => Promise<{ ok: boolean }>;
  };
}
```

### 多账户支持

单个渠道可配置多个账户:

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        personal: { authDir: "~/.openclaw/credentials/whatsapp/personal" },
        business: { authDir: "~/.openclaw/credentials/whatsapp/business" }
      }
    }
  }
}
```

---

## 会话管理机制

### 会话模型

OpenClaw 的会话系统提供关键的数据隔离:

**会话存储位置**:
```
~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl
```

### 会话模式

| 模式 | 描述 |
|------|------|
| 默认模式 | 每个 Agent 一个共享 DM 会话，群组/渠道独立上下文 |
| 安全 DM 模式 | 每个发送者/渠道隔离直接消息，防止上下文共享 |

### 队列模式

消息队列支持三种模式:

| 模式 | 行为 |
|------|------|
| `steer` | 入站消息注入当前运行，工具调用后检查队列 |
| `followup` | 消息保留直到当前回合结束，然后开始新回合 |
| `collect` | 收集消息，批量处理 |

### 流式输出与分块

- **Block Streaming**: 完成的 assistant 块立即发送
- **Chunk 大小**: 默认 800-1200 字符
- **断点优先级**: 段落 > 换行 > 句子

---

## 多智能体路由系统

### 核心概念

OpenClaw 支持在单个 Gateway 中运行多个隔离的 Agent:

```
┌─────────────────────────────────────────────────────────────┐
│                        Gateway                               │
│                                                              │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │
│  │   Agent A     │  │   Agent B     │  │   Agent C     │   │
│  │  (Workspace)  │  │  (Workspace)  │  │  (Workspace)  │   │
│  │  (Sessions)   │  │  (Sessions)   │  │  (Sessions)   │   │
│  │  (Auth)       │  │  (Auth)       │  │  (Auth)       │   │
│  └───────────────┘  └───────────────┘  └───────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 每个 Agent 的隔离组件

- **工作区** (Workspace): 文件、AGENTS.md、SOUL.md、USER.md
- **状态目录** (agentDir): 认证配置、模型注册、Agent 配置
- **会话存储** (Sessions): `~/.openclaw/agents/<agentId>/sessions`

### 路由绑定规则

绑定是确定性的，**最具体者优先**:

```
1. peer 匹配 (精确 DM/群组/渠道 ID)
2. parentPeer 匹配 (线程继承)
3. guildId + roles (Discord 角色路由)
4. guildId (Discord)
5. teamId (Slack)
6. accountId 匹配
7. 渠道级匹配 (accountId: "*")
8. 回退到默认 Agent
```

### 配置示例

```json5
{
  agents: {
    list: [
      { id: "home", workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" }
    ]
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "business" } }
  ]
}
```

---

## 工具与技能系统

### 内置工具

OpenClaw 提供核心工具集:

| 工具类别 | 包含工具 |
|----------|----------|
| 文件操作 | read, write, edit, apply_patch |
| 系统执行 | exec, bash, process |
| 浏览器 | browser (snapshot, navigate, act) |
| 会话 | sessions_list, sessions_history, sessions_send |
| 节点 | camera, screen, location, notifications |
| 自动化 | cron, webhooks |

### 技能 (Skills) 系统

技能是模块化的 Markdown 包，扩展 Agent 能力:

**加载位置优先级**:
1. 工作区技能: `<workspace>/skills`
2. 托管/本地技能: `~/.openclaw/skills`
3. 捆绑技能: 随安装包提供

**技能目录结构**:
```
skills/
  └── my-skill/
      ├── SKILL.md      # 技能定义
      ├── scripts/      # 可执行脚本
      └── templates/    # 模板文件
```

### ClawHub 技能注册中心

ClawHub 是最小化的技能注册中心，Agent 可以自动搜索和拉取新技能:
- 社区驱动
- 数百个预构建技能
- 支持热加载

---

## 浏览器控制系统

### 架构设计

OpenClaw 可以运行专用的 Chrome/Brave/Edge/Chromium 配置文件，与用户个人浏览器完全隔离。

```
┌──────────────────────────────────────────────────────────────┐
│                    浏览器控制架构                              │
│                                                               │
│  ┌─────────────────┐     ┌─────────────────────────────────┐│
│  │   Agent Tool    │────▶│     Browser Control Server      ││
│  │   (browser)     │     │     (Loopback HTTP API)         ││
│  └─────────────────┘     └──────────────┬──────────────────┘│
│                                          │                   │
│                          ┌───────────────┼───────────────┐   │
│                          │               │               │   │
│                          ▼               ▼               ▼   │
│                  ┌───────────┐   ┌───────────┐   ┌───────────┐
│                  │  openclaw │   │  chrome   │   │  remote   │
│                  │  profile  │   │  profile  │   │  CDP URL  │
│                  │ (managed) │   │(extension)│   │           │
│                  └───────────┘   └───────────┘   └───────────┘
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 多配置文件支持

| 配置文件类型 | 描述 |
|--------------|------|
| `openclaw` | 托管的隔离浏览器实例 |
| `chrome` | Chrome 扩展中继 (使用现有 Chrome) |
| `remote` | 远程 CDP URL (Browserless 等) |

### 快照与引用系统

两种快照样式:

**AI Snapshot (数字引用)**:
```bash
openclaw browser snapshot
# 输出包含数字引用的文本快照
openclaw browser click 12
```

**Role Snapshot (角色引用)**:
```bash
openclaw browser snapshot --interactive
# 输出 [ref=e12] 格式的角色列表
openclaw browser click e12
```

### 控制 API

浏览器暴露本地回环 HTTP API:

| 端点 | 功能 |
|------|------|
| `GET /tabs` | 列出所有标签页 |
| `POST /navigate` | 导航到 URL |
| `POST /act` | 执行操作 (click, type, drag) |
| `GET /snapshot` | 获取页面快照 |
| `POST /screenshot` | 截图 |
| `POST /pdf` | 生成 PDF |

---

## Canvas 可视化工作区

### 概述

Canvas 是 macOS 应用中嵌入的 Agent 控制可视化面板，使用 WKWebView 实现。

### 文件存储

```
~/Library/Application Support/OpenClaw/canvas/<session>/...
```

### 自定义 URL Scheme

```
openclaw-canvas://<session>/<path>
```

示例:
- `openclaw-canvas://main/` → 加载 `index.html`
- `openclaw-canvas://main/assets/app.css` → 加载 CSS 资源

### A2UI 集成

Canvas 支持 A2UI v0.8 协议:

| 消息类型 | 功能 |
|----------|------|
| `beginRendering` | 开始渲染 |
| `surfaceUpdate` | 更新表面 |
| `dataModelUpdate` | 更新数据模型 |
| `deleteSurface` | 删除表面 |

### Agent API

Canvas 通过 Gateway WebSocket 暴露:
- `canvas present` - 显示/隐藏面板
- `canvas navigate` - 导航到路径或 URL
- `canvas eval` - 执行 JavaScript
- `canvas snapshot` - 捕获快照图像

---

## 语音唤醒与对话系统

### Voice Wake 系统

OpenClaw 将唤醒词作为 **Gateway 拥有的全局列表**:

- 无每节点自定义唤醒词
- 任何节点/应用 UI 都可以编辑列表
- 更改持久化并广播给所有客户端

**存储位置**:
```
~/.openclaw/settings/voicewake.json
```

**数据结构**:
```json
{
  "triggers": ["openclaw", "claude", "computer"],
  "updatedAtMs": 1730000000000
}
```

### Talk Mode

Talk Mode 提供持续对话能力:
- macOS/iOS/Android 支持
- ElevenLabs TTS 集成
- 实时语音转文字

### 协议

| 方法 | 描述 |
|------|------|
| `voicewake.get` | 获取触发词列表 |
| `voicewake.set` | 设置触发词列表 |
| `voicewake.changed` (事件) | 触发词变更通知 |

---

## 插件扩展架构

### 插件系统概览

插件是扩展 OpenClaw 的代码模块，可以注册:
- Gateway RPC 方法
- Gateway HTTP 处理器
- Agent 工具
- CLI 命令
- 后台服务
- 技能
- 自动回复命令

### 插件发现优先级

```
1. 配置路径: plugins.load.paths
2. 工作区扩展: <workspace>/.openclaw/extensions/
3. 全局扩展: ~/.openclaw/extensions/
4. 捆绑扩展: <openclaw>/extensions/ (默认禁用)
```

### 插件配置

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-extension"] },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } }
    }
  }
}
```

### 官方插件

| 插件 | 功能 |
|------|------|
| `@openclaw/voice-call` | 语音通话 (Twilio) |
| `@openclaw/msteams` | Microsoft Teams |
| `@openclaw/matrix` | Matrix 协议 |
| `@openclaw/zalo` | Zalo 消息 |
| `memory-lancedb` | 长期记忆 (LanceDB) |

### 插件 API 示例

```typescript
export default function register(api) {
  // 注册 Gateway 方法
  api.registerGatewayMethod("myplugin.status", ({ respond }) => {
    respond(true, { ok: true });
  });

  // 注册 CLI 命令
  api.registerCli(({ program }) => {
    program.command("mycmd").action(() => {
      console.log("Hello");
    });
  });

  // 注册 Hook
  api.registerHook("command:new", async () => {
    // Hook 逻辑
  });
}
```

---

## 安全防护体系

### 纵深防御框架

OpenClaw 实施了 **40+ 安全加固措施**，涵盖六个类别:

### 1. 插件信任边界

- 基于能力的沙箱限制插件权限
- 插件声明所需能力
- 运行时强制限制 API 和文件系统路径访问

### 2. 速率限制

- 令牌桶算法防止滥用
- 可配置的 IP/用户/端点限制
- 多实例部署支持 Redis 分布式限流

### 3. Webhook 保护

- HMAC-SHA256 签名验证
- 时间戳验证 (5 分钟窗口) 防止重放攻击
- 负载大小限制和模式验证

### 4. 运行时容器化

- seccomp 配置文件限制约 50 个允许的系统调用
- 只读根文件系统
- 能力丢弃防止权限提升

### 5. 认证系统

- 设备中心模型，需要显式授权
- 每个设备接收范围限定的访问令牌
- 高风险操作支持生物识别绑定

### 6. TLS 强制

- TLS 1.3 强制
- 已知服务的证书固定

### DM 访问控制

| 策略 | 描述 |
|------|------|
| `pairing` | 未知发送者收到配对码 |
| `allowlist` | 仅允许列表中的发送者 |
| `open` | 允许所有发送者 (需显式启用) |

### 沙箱模式

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",  // 非主会话在 Docker 沙箱中运行
        docker: {
          setupCommand: "apt-get update && apt-get install -y git"
        }
      }
    }
  }
}
```

---

## 部署与运维

### 快速部署

```bash
# 安装
npm install -g openclaw@latest

# 引导配置
openclaw onboard --install-daemon

# 启动 Gateway
openclaw gateway --port 18789 --verbose

# 验证状态
openclaw gateway status
openclaw channels status --probe
```

### 服务管理

**macOS (launchd)**:
```bash
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
```

**Linux (systemd)**:
```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway.service
```

### 健康检查

```bash
openclaw health          # 基本健康检查
openclaw doctor          # 诊断和修复
openclaw logs --follow   # 实时日志
```

### 多 Gateway 部署

每个实例需要:
- 唯一的 `gateway.port`
- 唯一的 `OPENCLAW_CONFIG_PATH`
- 唯一的 `OPENCLAW_STATE_DIR`
- 唯一的工作区

### 高可用架构

```
┌─────────────────────────────────────────────────────────────┐
│                    高可用部署架构                             │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                      │
│  │Gateway 1│  │Gateway 2│  │Gateway N│   (无状态实例)        │
│  └────┬────┘  └────┬────┘  └────┬────┘                      │
│       │            │            │                            │
│       └────────────┼────────────┘                            │
│                    │                                         │
│            ┌───────┴───────┐                                 │
│            │    Redis      │   (会话存储)                    │
│            └───────┬───────┘                                 │
│                    │                                         │
│            ┌───────┴───────┐                                 │
│            │  PostgreSQL   │   (持久化)                      │
│            └───────────────┘                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 可观测性 (OTEL v2)

- 分布式追踪跨组件跟踪请求
- 关联 ID 与结构化日志关联 span
- 开销降低到 2% 以下

---

## 技术创新总结

### 1. Gateway 单体架构

**创新点**: 摒弃微服务，采用单一 Gateway 作为控制平面

**优势**:
- 简化部署和运维
- 减少网络延迟
- 统一状态管理
- 更容易调试

### 2. 文件驱动配置

**创新点**: 使用 Markdown 文件定义 Agent 行为

**优势**:
- 版本控制友好
- 完全可审计
- 人类可读
- 易于修改

### 3. 多智能体隔离

**创新点**: 单 Gateway 支持多个完全隔离的 Agent

**优势**:
- 资源共享高效
- 真正的数据隔离
- 灵活的路由规则
- 支持多用户场景

### 4. 确定性路由

**创新点**: 最具体者优先的绑定规则

**优势**:
- 可预测的消息路由
- 精细的控制粒度
- 支持复杂的路由场景

### 5. 浏览器控制隔离

**创新点**: 专用浏览器配置文件与快照引用系统

**优势**:
- 避免与个人浏览器冲突
- 确定性的标签页控制
- 稳定的 UI 树快照

### 6. Heartbeat 自主执行

**创新点**: Agent 可以主动执行任务而非仅响应用户

**优势**:
- 支持定时任务
- 自动化工作流
- 减少用户干预

### 7. 本地优先隐私

**创新点**: 所有数据和计算在本地设备

**优势**:
- 完全的数据主权
- GDPR 合规 (使用本地模型时)
- 无第三方依赖

### 8. 插件热加载

**创新点**: 开发时支持技能热重载

**优势**:
- 快速迭代
- 降低开发成本
- 灵活扩展

---

## 参考资源

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [GitHub 仓库](https://github.com/openclaw/openclaw)
- [ClawHub 技能中心](https://clawhub.com)
- [架构深度解析 - innFactory](https://innfactory.ai/en/blog/openclaw-architecture-explained/)
- [技术安全分析 - Atal Upadhyay](https://atalupadhyay.wordpress.com/2026/02/21/openclaw-2026-2-19-technical-deep-dive-security-analysis/)
- [开发者指南 - DEV Community](https://dev.to/laracopilot/what-is-openclaw-ai-in-2026-a-practical-guide-for-developers-25hj)

---

*文档版本: 2026.02*
*基于 OpenClaw 2026.2.x 版本分析*
