# OpenClaw 详细设计文档

## 目录
1. [概述](#概述)
2. [总体架构](#总体架构)
3. [核心组件设计](#核心组件设计)
4. [技术栈与基础设施](#技术栈与基础设施)
5. [模块详细设计](#模块详细设计)
6. [数据流与交互](#数据流与交互)
7. [关键设计模式](#关键设计模式)
8. [安全与治理](#安全与治理)

---

## 概述

### 项目定位

**OpenClaw** 是一个**多通道 AI 网关平台**，旨在将 AI 助手能力统一接入到 25+ 主流消息平台（Telegram、Discord、Slack、WhatsApp、Signal、iMessage 等）。项目采用 **Monorepo + 插件化架构**，通过 Gateway 作为控制平面，Channel Plugins 作为数据平面，实现统一的 AI Agent 调度、会话管理和消息路由。

### 核心价值

- **统一消息接入**：一套代码同时支持多个消息平台
- **插件化扩展**：通过 Plugin SDK 实现通道、工具、服务的灵活扩展
- **AI Agent 集成**：基于 Pi Agent Core 实现多模态 AI 对话
- **企业级治理**：安全策略、权限控制、审计日志、密钥管理

### 版本信息

- **当前版本**: 2026.3.3
- **运行时**: Node.js 22+ / Bun
- **语言**: TypeScript (ESM, target ES2023)
- **协议**: MIT

---

## 总体架构

### 架构模式

**控制平面 + 数据平面分离架构**

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Plane (Gateway)                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Session │  │  Agent   │  │   Cron   │  │  Plugin  │   │
│  │  Manager │  │ Runtime  │  │ Scheduler│  │ Registry │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         WebSocket Server (ws + Express 5)            │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           ▲
                           │ WebSocket / HTTP
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                       Data Plane (Channels)                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │Telegram │ │ Discord │ │  Slack  │ │WhatsApp │   ...    │
│  │ Plugin  │ │ Plugin  │ │ Plugin  │ │ Plugin  │          │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │
└─────────────────────────────────────────────────────────────┘
                           ▲
                           │ Webhook / Polling
                           ▼
        [Telegram] [Discord] [Slack] [WhatsApp] [Signal] ...
```

### 分层架构

**四层架构模型**：

1. **表示层 (Presentation Layer)**
   - CLI 命令行界面 (src/cli/)
   - Web Provider (WebChat UI)
   - Control UI (监控面板)

2. **应用层 (Application Layer)**
   - Gateway 服务器 (src/gateway/)
   - Agent 执行引擎 (src/agents/, src/acp/)
   - Session 会话管理 (src/sessions/)
   - Routing 路由决策 (src/routing/)

3. **通道层 (Channel Layer)**
   - Core Channels (src/telegram, src/discord, ...)
   - Extension Channels (extensions/, 40+ 插件)

4. **基础设施层 (Infrastructure Layer)**
   - 配置管理 (src/config/)
   - 密钥存储 (src/secrets/)
   - 日志系统 (src/logging/)
   - 媒体处理 (src/media/)
   - 网络工具 (src/infra/net/)

---

## 核心组件设计

### 1. CLI (Command Line Interface)

**位置**: `src/cli/`, `src/commands/`

**技术选型**: Commander.js

**设计要点**:

- **依赖注入模式**: 所有命令通过 `createDefaultDeps()` 注入依赖，便于测试
- **层级命令结构**: 支持 `openclaw gateway run`, `openclaw channels status` 等多级命令
- **Profile 机制**: 开发/生产环境配置隔离 (OPENCLAW_PROFILE=dev)

**关键文件**:
- `src/cli/program.js`: 命令注册入口
- `src/cli/deps.ts`: 依赖注入容器
- `src/commands/gateway/run.ts`: Gateway 启动命令
- `src/commands/channels/status.ts`: 通道状态查询

**命令分类**:
```
核心命令:
├── gateway        # Gateway 管理
│   ├── run        # 启动 Gateway
│   ├── stop       # 停止 Gateway
│   └── logs       # 查看日志
├── agent          # Agent 执行
│   └── run        # 执行 Agent
├── channels       # 通道管理
│   ├── status     # 通道状态
│   ├── setup      # 通道配置向导
│   └── doctor     # 诊断修复
├── onboard        # 新用户引导
├── doctor         # 系统诊断
└── sessions       # 会话管理
```

---

### 2. Gateway Server

**位置**: `src/gateway/`

**核心职责**:
- WebSocket 服务器 (实时通信)
- Session 生命周期管理
- 通道健康监控
- Cron 任务调度
- 配置热重载
- Plugin HTTP 路由

**实现细节**:

#### 服务器架构 (`server.impl.ts`)

```typescript
type GatewayServer = {
  close: (opts?: { reason?: string; restartExpectedMs?: number | null }) => Promise<void>;
};
```

**核心模块**:
- **Server Methods** (`server-methods/`): RPC 风格的方法处理器
  - `agent.ts`: Agent 执行
  - `chat.ts`: 聊天会话
  - `sessions.ts`: 会话持久化
  - `cron.ts`: 定时任务
  - `send.ts`: 出站消息
  - `nodes.ts`: 移动/桌面节点命令

- **Channel Manager** (`server-channels.ts`):
  - 通道插件生命周期管理
  - 健康检查与监控
  - 入站消息分发

- **Node Registry** (`node-registry.ts`):
  - 移动设备注册
  - 节点能力发现 (Capabilities)

- **Auth & Security**:
  - Token 认证
  - 设备配对 (src/pairing/)
  - OAuth 流程
  - 速率限制 (`auth-rate-limit.ts`)

#### WebSocket 协议

**消息格式**:
```typescript
{
  method: string;      // RPC 方法名
  params: unknown;     // 参数
  id?: string;         // 请求 ID
}
```

**事件系统**:
```typescript
// Gateway 事件
GATEWAY_EVENT_UPDATE_AVAILABLE  // 版本更新通知
GATEWAY_EVENT_CHANNEL_HEALTH    // 通道健康状态
GATEWAY_EVENT_PRESENCE_UPDATE   // 在线状态更新
```

---

### 3. Channel Plugin System

**位置**: `src/channels/plugins/`, `extensions/`

**设计哲学**: **适配器模式 + 约定优于配置**

#### 插件契约 (`types.plugin.ts`)

```typescript
type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  // 元数据
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;

  // 核心适配器 (必须实现)
  config: ChannelConfigAdapter<ResolvedAccount>;  // 账号解析

  // 可选适配器
  setup?: ChannelSetupAdapter;           // 初始化配置
  pairing?: ChannelPairingAdapter;       // 设备配对
  security?: ChannelSecurityAdapter;     // 安全策略
  outbound?: ChannelOutboundAdapter;     // 出站消息
  status?: ChannelStatusAdapter;         // 健康检查
  gateway?: ChannelGatewayAdapter;       // Gateway 方法
  streaming?: ChannelStreamingAdapter;   // 流式响应
  threading?: ChannelThreadingAdapter;   // 线程支持
  mentions?: ChannelMentionAdapter;      // @提及
  agentTools?: ChannelAgentToolFactory;  // Agent 工具
  // ... 更多适配器
};
```

#### 适配器详解

**1. Config Adapter**: 账号解析
```typescript
type ChannelConfigAdapter<ResolvedAccount> = {
  resolveAccount: (config: unknown, accountId: string) => ResolvedAccount;
  listAccounts: (config: unknown) => AccountInfo[];
};
```

**2. Outbound Adapter**: 消息发送
```typescript
type ChannelOutboundAdapter = {
  sendMessage: (ctx: OutboundContext) => Promise<SendResult>;
  sendReaction?: (ctx: ReactionContext) => Promise<void>;
  sendTyping?: (ctx: TypingContext) => Promise<void>;
};
```

**3. Security Adapter**: 安全治理
```typescript
type ChannelSecurityAdapter = {
  resolveDmPolicy: (config: unknown, accountId: string) => DmPolicy;
  resolveAllowlist: (config: unknown, accountId: string) => Allowlist;
  checkMessagePolicy: (ctx: SecurityContext) => PolicyResult;
};
```

#### 插件注册流程

```typescript
// 1. 定义插件
const telegramPlugin: ChannelPlugin = {
  id: "telegram",
  meta: { name: "Telegram", icon: "telegram" },
  capabilities: {
    supportsThreads: false,
    supportsReactions: true
  },
  config: { /* ... */ },
  outbound: { /* ... */ }
};

// 2. 在扩展入口注册
const plugin = {
  id: "telegram",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    api.registerChannel({ plugin: telegramPlugin });
  },
};
```

#### 插件分类

**核心通道** (在 `src/` 中):
- `telegram/` - Telegram (grammY)
- `discord/` - Discord (discord.js)
- `slack/` - Slack (Bolt)
- `signal/` - Signal (signal-cli)
- `web/` - WhatsApp (Baileys)
- `imessage/` - iMessage
- `line/` - LINE

**扩展通道** (在 `extensions/` 中):
- `bluebubbles/`, `msteams/`, `matrix/`, `googlechat/`
- `feishu/`, `irc/`, `nostr/`, `twitch/`, `zalo/`
- `mattermost/`, `nextcloud-talk/`, `synology-chat/`
- `voice-call/` - 语音通话集成
- 总计 40+ 扩展插件

---

### 4. Agent Runtime

**位置**: `src/agents/`, `src/acp/`

**核心引擎**: `@mariozechner/pi-agent-core` (Pi Agent Core)

#### Agent 执行模式

**1. Session Mode**: 会话式对话
```typescript
// 持久化会话，支持上下文记忆
acpRuntime.startSession({
  agentId: "default",
  sessionKey: "telegram:user123"
});
```

**2. Run Mode**: 单次执行
```typescript
// 无状态执行
acpRuntime.run({
  agentId: "default",
  message: "What time is it?"
});
```

#### Agent Identity System (`identity.ts`)

**身份配置层级**:
```
L1: Channel Account Level (优先级最高)
  └── account.config.ackReaction

L2: Channel Level
  └── channel.config.ackReaction

L3: Global Level
  └── messages.ackReaction

L4: Agent Identity (默认)
  └── agent.identity.emoji
```

**身份属性**:
```typescript
type IdentityConfig = {
  name?: string;           // Agent 名称
  emoji?: string;          // 标识 Emoji
  bio?: string;            // 自我介绍
  messagePrefix?: string;  // 消息前缀
  responsePrefix?: string; // 响应前缀
};
```

#### Tool Registry

**内置工具** (`src/agents/tools/`):
- **Browser**: 浏览器自动化 (Playwright)
- **Canvas**: 画布渲染
- **Nodes**: 节点控制
- **Skills**: 技能系统
- **Memory**: 记忆管理

**工具注册**:
```typescript
api.registerTool({
  name: "browser_navigate",
  description: "Navigate to a URL",
  inputSchema: { /* Zod/TypeBox schema */ },
  execute: async (params, context) => { /* ... */ }
});
```

---

### 5. Message Routing

**位置**: `src/routing/`

**设计目标**: 通道无关的消息路由

#### 路由决策 (`resolve-route.ts`)

```typescript
type ResolvedAgentRoute = {
  agentId: string;           // 目标 Agent
  channel: string;           // 通道 ID
  accountId: string;         // 账号 ID
  sessionKey: string;        // 会话键
  matchedBy: RouteMatchType; // 匹配方式
};

type RouteMatchType =
  | "binding.peer"      // 点对点绑定
  | "binding.guild"     // 群组绑定
  | "binding.role"      // 角色绑定
  | "binding.thread"    // 线程绑定
  | "default";          // 默认路由
```

#### 路由维度

**三维路由矩阵**:
```
Channel × Account × Peer → Agent
```

**路由规则优先级**:
```
1. Thread Binding (线程级)
   └── 绑定子 Agent 到特定线程

2. Peer Binding (用户级)
   └── 特定用户 → 特定 Agent

3. Guild/Team Binding (群组级)
   └── 特定群组 → 特定 Agent

4. Role Binding (角色级)
   └── 特定角色 → 特定 Agent

5. Default (默认)
   └── 默认 Agent
```

#### Session Key System

**会话键编码**:
```typescript
buildAgentPeerSessionKey({
  agentId: "default",
  channel: "telegram",
  accountId: "bot1",
  peerKind: "user",
  peerId: "12345"
})
// => "default:telegram:bot1:user:12345"
```

**会话隔离**:
- 每个 Session Key 对应独立的会话状态
- 支持跨通道持久化
- 支持会话恢复与迁移

---

### 6. Media Processing Pipeline

**位置**: `src/media/`

**支持的媒体类型**:
- 图片 (JPEG, PNG, GIF, WebP)
- 音频 (MP3, WAV, OGG, M4A)
- 视频 (MP4, WebM)
- 文档 (PDF, Office)

#### 媒体处理流程

```
Inbound Media Flow:
┌──────────────────┐
│  Channel Plugin  │ (接收媒体)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Inbound Policy   │ (policy check: size, type, source)
│  - Size limits   │
│  - Type check    │
│  - SSRF guard    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Temp Storage    │ (temp-files.ts)
│  - Secure path   │
│  - Lifecycle mgmt│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Processing      │
│  - Transcription │ (Whisper API)
│  - OCR           │ (Tesseract)
│  - Resize        │ (Sharp)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Agent Context   │ (传递给 AI)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Cleanup         │ (自动清理)
└──────────────────┘
```

#### 关键模块

**1. Media Store (`store.ts`)**:
```typescript
type MediaStore = {
  store: (media: InboundMedia) => Promise<MediaRef>;
  retrieve: (ref: MediaRef) => Promise<Buffer>;
  cleanup: (ref: MediaRef) => Promise<void>;
};
```

**2. Image Operations (`image-ops.ts`)**:
- 图片缩放
- 格式转换
- 压缩优化

**3. Audio Processing (`audio.ts`)**:
- 音频转录 (Whisper)
- 格式转换 (ffmpeg)
- 静音检测

**4. PDF Extraction (`pdf-extract.ts`)**:
- 文本提取 (pdfjs-dist)
- 表格识别
- 图片抽取

**5. Media Fetch (`fetch.ts`)**:
- SSRF 防护
- 大小限制
- 超时控制

---

### 7. Plugin System

**位置**: `src/plugins/`, `src/plugin-sdk/`

#### Plugin SDK 架构

**公共 API** (`src/plugin-sdk/index.ts`):
```typescript
export type OpenClawPluginApi = {
  // 通道注册
  registerChannel: (opts: { plugin: ChannelPlugin }) => void;

  // 工具注册
  registerTool: (tool: ToolDefinition) => void;

  // 服务注册
  registerService: (service: OpenClawPluginService) => void;

  // Gateway 方法
  registerGatewayMethod: (method: OpenClawPluginGatewayMethod) => void;

  // HTTP 路由
  registerHttpRoute: (route: HttpRouteDefinition) => void;
};
```

**通道特定 SDK**:
```typescript
// openclaw/plugin-sdk/telegram
export { TelegramPlugin, GrammyContext } from '../telegram/plugin-sdk.js';

// openclaw/plugin-sdk/discord
export { DiscordPlugin, DiscordClient } from '../discord/plugin-sdk.js';
```

#### 插件生命周期

```
Plugin Lifecycle:
┌─────────────────────┐
│ 1. Discovery        │ (扫描 extensions/, 加载 package.json)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 2. Registration     │ (调用 register(api))
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 3. Service Binding  │ (api.registerChannel(), ...)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 4. Runtime          │ (处理事件, 执行逻辑)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 5. Shutdown         │ (清理钩子, 释放资源)
└─────────────────────┘
```

#### 插件类型

**1. Channel Plugin**:
```typescript
type ChannelPluginDefinition = {
  id: string;
  configSchema?: PluginConfigSchema;
  register: (api: OpenClawPluginApi) => void;
};
```

**2. Provider Plugin**: AI 模型提供商认证
```typescript
type ProviderPlugin = {
  id: string;
  authenticate: (context: ProviderAuthContext) => Promise<AuthResult>;
};
```

**3. Service Plugin**: 自定义服务
```typescript
type OpenClawPluginService = {
  id: string;
  start: () => Promise<void>;
  stop: () => Promise<void>;
};
```

---

### 8. Configuration Management

**位置**: `src/config/`

#### 配置结构 (`types.ts`)

```typescript
type OpenClawConfig = {
  // Agent 配置
  agents?: Record<string, AgentConfig>;

  // 通道配置
  channels?: Record<string, ChannelConfig>;

  // Gateway 配置
  gateway?: GatewayConfig;

  // 模型配置
  models?: Record<string, ModelConfig>;

  // 消息配置
  messages?: MessagesConfig;

  // Cron 配置
  cron?: Record<string, CronConfig>;

  // 密钥引用
  secrets?: SecretsConfig;

  // Hooks
  hooks?: HooksConfig;

  // 内存配置
  memory?: MemoryConfig;
};
```

#### 配置层级

```
Configuration Resolution:
┌────────────────────┐
│ 1. Config File     │ (~/.openclaw/config.json)
│   - JSON5 format   │
│   - Schema validated│
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ 2. Environment Vars│ (OPENCLAW_*)
│   - Overrides      │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ 3. Secrets Runtime │ (密钥注入)
│   - 1Password      │
│   - AWS Secrets    │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ 4. Runtime Config  │ (最终合并)
└────────────────────┘
```

#### 热重载机制

**Gateway Config Reloader** (`src/gateway/config-reload.ts`):
```typescript
// 监听配置文件变化
chokidar.watch(CONFIG_PATH).on('change', async () => {
  const newConfig = await loadConfig();
  await applyConfigDiff(oldConfig, newConfig);
  notifyConnectedClients();
});
```

---

### 9. Security & Governance

**位置**: `src/security/`, `src/secrets/`, `src/channels/security*.ts`

#### 安全层次模型

```
Security Layers:
┌─────────────────────────────────────────┐
│ Layer 1: Network Security               │
│  - SSRF protection (net/ssrf-guard.ts)  │
│  - Rate limiting                        │
│  - IP allowlist                         │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ Layer 2: Authentication                 │
│  - Token auth                           │
│  - Device pairing                       │
│  - OAuth flows                          │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ Layer 3: Channel Security               │
│  - DM policy (who can DM the bot)       │
│  - Allowlist (who is allowed)           │
│  - Command gating (who can run commands)│
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ Layer 4: Execution Security             │
│  - Tool approval system                 │
│  - Sandbox isolation                    │
│  - Command approval queue               │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ Layer 5: Data Security                  │
│  - Secret management                    │
│  - Encryption at rest                   │
│  - Audit logging                        │
└─────────────────────────────────────────┘
```

#### DM Policy System (`src/channels/security-dm-policy.ts`)

```typescript
type DmPolicy =
  | { mode: "allowlist"; allowlist: string[] }
  | { mode: "paired" }  // 仅已配对用户
  | { mode: "public" }  // 所有人
  | { mode: "disabled" }; // 禁用 DM
```

#### Command Gating (`src/channels/security-commands.ts`)

```typescript
type CommandPolicy = {
  allowFrom?: string[];      // 允许的用户/角色
  denyFrom?: string[];       // 拒绝的用户/角色
  requireRole?: string[];    // 需要的角色
  requirePermission?: string[];
};
```

#### Secret Management (`src/secrets/`)

**支持的后端**:
- 1Password Connect
- AWS Secrets Manager
- Environment Variables
- Local File (开发环境)

**密钥引用语法**:
```json
{
  "channels": {
    "telegram": {
      "token": { "op": "op://Private/Telegram/token" }
    }
  }
}
```

---

### 10. Infrastructure Components

**位置**: `src/infra/` (289 文件)

#### 核心基础设施

**1. Port Management (`ports.ts`)**:
```typescript
// 自动端口发现与分配
type PortManager = {
  findAvailablePort: (preferred?: number) => Promise<number>;
  isPortAvailable: (port: number) => Promise<boolean>;
};
```

**2. Process Lifecycle (`restart.ts`)**:
```typescript
type RestartManager = {
  gracefulRestart: () => Promise<void>;
  scheduleRestart: (delayMs: number) => void;
};
```

**3. Bonjour Discovery (`bonjour-discovery.ts`)**:
```typescript
// mDNS 服务发现
type BonjourService = {
  advertise: (service: ServiceInfo) => void;
  discover: (type: string) => Promise<ServiceInfo[]>;
};
```

**4. Tailscale Integration (`tailscale.ts`)**:
```typescript
// Tailscale 网络集成
type TailscaleClient = {
  getStatus: () => Promise<TailscaleStatus>;
  exposeGateway: (port: number) => Promise<void>;
};
```

**5. Heartbeat Runner (`heartbeat-runner.ts`)**:
```typescript
// 定期心跳消息
type HeartbeatConfig = {
  message: string;
  channel: string;
  accountId: string;
  schedule: CronExpression;
};
```

**6. Exec Approval System (`exec-approvals.ts`)**:
```typescript
// 命令执行审批
type ExecApprovalRequest = {
  command: string;
  reason: string;
  risk: "low" | "medium" | "high";
  requestedBy: string;
};

type ExecApprovalResult = {
  approved: boolean;
  approvedBy?: string;
  reason?: string;
};
```

---

## 技术栈与基础设施

### 语言与运行时

| 类别 | 技术 | 版本 | 用途 |
|-----|------|------|------|
| **语言** | TypeScript | 5.9.3 | 主要开发语言 |
| | Swift | - | iOS/macOS 应用 |
| | Kotlin | - | Android 应用 |
| **运行时** | Node.js | ≥22.12.0 | 主运行时 |
| | Bun | - | 替代运行时 |
| **编译** | tsdown | 0.21.0-beta.2 | TypeScript 编译 |
| **模块** | ESM | ES2023 | 模块系统 |

### 核心框架与库

| 类别 | 库名 | 版本 | 用途 |
|-----|------|------|------|
| **CLI** | Commander.js | 14.0.3 | 命令行框架 |
| **AI Runtime** | @mariozechner/pi-agent-core | 0.55.3 | Agent 核心引擎 |
| **Telegram** | grammY | 1.41.0 | Telegram Bot |
| **Discord** | discord.js | - | Discord Bot |
| **Slack** | @slack/bolt | 4.6.0 | Slack App |
| **WhatsApp** | @whiskeysockets/baileys | 7.0.0-rc.9 | WhatsApp Web |
| **WebSocket** | ws | 8.19.0 | WebSocket 服务器 |
| **HTTP** | Express | 5.2.1 | HTTP 服务器 |
| **Schema** | Zod | 4.3.6 | Schema 验证 |
| | TypeBox | 0.34.48 | Schema 定义 |
| | AJV | 8.18.0 | Schema 编译 |

### 开发工具

| 类别 | 工具 | 版本 | 用途 |
|-----|------|------|------|
| **测试** | Vitest | 4.0.18 | 单元测试 |
| | V8 Coverage | 4.0.18 | 代码覆盖率 |
| **Lint** | Oxlint | 1.50.0 | 代码检查 |
| | Oxlint TSGo | 0.15.0 | 类型感知 Lint |
| **Format** | Oxfmt | 0.35.0 | 代码格式化 |
| **包管理** | pnpm | 10.23.0 | 包管理器 |

### 构建流程

```bash
# 完整构建流程
pnpm build
  ├─ pnpm canvas:a2ui:bundle          # Canvas UI 打包
  ├─ tsdown                            # TypeScript 编译
  ├─ pnpm build:plugin-sdk:dts        # 生成类型声明
  ├─ scripts/write-plugin-sdk-entry-dts.ts
  ├─ scripts/canvas-a2ui-copy.ts
  ├─ scripts/copy-hook-metadata.ts
  ├─ scripts/copy-export-html-templates.ts
  ├─ scripts/write-build-info.ts
  └─ scripts/write-cli-startup-metadata.ts
```

---

## 模块详细设计

### Source Code Organization (`src/`)

```
src/
├── acp/              # Agent Control Protocol 运行时
│   ├── runtime.ts    # ACP 运行时实现
│   └── session.ts    # 会话管理
│
├── agents/           # Agent 相关
│   ├── identity.ts   # 身份配置
│   ├── agent-scope.ts # Agent 配置解析
│   ├── tools/        # Agent 工具
│   │   ├── browser/  # 浏览器工具
│   │   ├── canvas/   # 画布工具
│   │   └── nodes/    # 节点工具
│   ├── skills/       # 技能系统
│   └── subagent-registry.ts # 子 Agent 注册
│
├── auto-reply/       # 自动回复
│   ├── reply/        # 回复分发
│   └── queue/        # 消息队列
│
├── browser/          # 浏览器自动化
│   ├── playwright.ts # Playwright 集成
│   └── stealth.ts    # 反检测
│
├── canvas-host/      # Canvas 渲染主机
│   ├── server.ts     # Canvas 服务器
│   └── a2ui/         # A2UI 打包
│
├── channels/         # 通道基础设施
│   ├── plugins/      # 通道插件系统
│   ├── security*.ts  # 安全策略
│   └── onboarding*.ts # 配置向导
│
├── cli/              # CLI 核心
│   ├── program.js    # 命令注册
│   ├── deps.ts       # 依赖注入
│   └── command-format.ts # 命令格式化
│
├── commands/         # CLI 命令实现
│   ├── gateway/      # Gateway 命令
│   ├── channels/     # Channel 命令
│   ├── agent/        # Agent 命令
│   └── onboard*.ts   # 引导命令
│
├── config/           # 配置管理
│   ├── config.ts     # 配置加载
│   ├── types.ts      # 类型定义
│   └── sessions.ts   # 会话配置
│
├── cron/             # Cron 任务
│   ├── store.ts      # 任务存储
│   └── runner.ts     # 任务执行器
│
├── daemon/           # 守护进程
│   └── launchd*.ts   # macOS LaunchAgent
│
├── discord/          # Discord 通道
├── telegram/         # Telegram 通道
├── slack/            # Slack 通道
├── signal/           # Signal 通道
├── imessage/         # iMessage 通道
├── web/              # WhatsApp Web
├── line/             # LINE 通道
│
├── gateway/          # Gateway 服务器
│   ├── server.impl.ts # 服务器实现
│   ├── server-methods/ # RPC 方法
│   ├── auth*.ts      # 认证
│   └── control-ui.ts # Control UI
│
├── hooks/            # Hook 系统
│   ├── runner.ts     # Hook 执行器
│   └── types.ts      # Hook 类型
│
├── infra/            # 基础设施 (289 文件)
│   ├── ports.ts      # 端口管理
│   ├── restart.ts    # 重启管理
│   ├── bonjour*.ts   # mDNS 发现
│   ├── tailscale.ts  # Tailscale 集成
│   ├── heartbeat*.ts # 心跳任务
│   ├── exec-approvals.ts # 命令审批
│   ├── net/          # 网络工具
│   └── outbound/     # 出站工具
│
├── logging/          # 日志系统
│   ├── subsystem.ts  # 子系统日志
│   └── diagnostic.ts # 诊断日志
│
├── media/            # 媒体处理
│   ├── store.ts      # 媒体存储
│   ├── image-ops.ts  # 图片操作
│   ├── audio.ts      # 音频处理
│   ├── fetch.ts      # 媒体获取
│   └── pdf-extract.ts # PDF 提取
│
├── memory/           # 记忆管理
│   ├── context.ts    # 上下文管理
│   └── store.ts      # 记忆存储
│
├── pairing/          # 设备配对
│   ├── protocol.ts   # 配对协议
│   └── store.ts      # 配对存储
│
├── plugin-sdk/       # Plugin SDK (公共 API)
│   ├── index.ts      # 主入口
│   ├── core.ts       # 核心类型
│   ├── telegram.ts   # Telegram SDK
│   ├── discord.ts    # Discord SDK
│   └── ...           # 其他通道 SDK
│
├── plugins/          # 插件运行时
│   ├── runtime/      # 运行时实现
│   ├── registry.ts   # 插件注册表
│   ├── services.ts   # 插件服务
│   └── hook-runner*.ts # Hook 执行
│
├── providers/        # AI 提供商
│   ├── openai.ts     # OpenAI
│   ├── anthropic.ts  # Anthropic
│   └── bedrock.ts    # AWS Bedrock
│
├── routing/          # 消息路由
│   ├── resolve-route.ts # 路由解析
│   └── bindings.ts   # 绑定管理
│
├── secrets/          # 密钥管理
│   ├── runtime.ts    # 运行时
│   ├── command-config.ts # 命令配置
│   └── sources/      # 密钥源
│       ├── onepassword.ts
│       ├── aws.ts
│       └── env.ts
│
├── security/         # 安全工具
│   ├── crypto.ts     # 加密
│   └── validation.ts # 验证
│
├── sessions/         # 会话持久化
│   ├── store.ts      # 会话存储
│   └── serializer.ts # 序列化
│
├── shared/           # 共享工具
│   ├── utils.ts      # 通用工具
│   └── constants.ts  # 常量
│
├── terminal/         # 终端 UI
│   ├── table.ts      # 表格输出
│   ├── palette.ts    # 颜色调色板
│   └── progress.ts   # 进度条
│
├── tts/              # 文本转语音
│   ├── edge-tts.ts   # Edge TTS
│   └── provider.ts   # TTS 提供者
│
├── tui/              # 终端 UI (TUI)
│   └── app.ts        # TUI 应用
│
├── types/            # 类型定义
│   ├── types.ts      # 导出汇总
│   ├── types.agent*.ts
│   ├── types.channels.ts
│   ├── types.gateway.ts
│   └── ...
│
├── utils/            # 工具函数
│   ├── async.ts      # 异步工具
│   ├── string.ts     # 字符串工具
│   └── time.ts       # 时间工具
│
├── wizard/           # 引导向导
│   ├── onboarding.ts # 新用户引导
│   └── steps/        # 引导步骤
│
├── entry.ts          # 入口点
├── runtime.ts        # 运行时环境
└── index.ts          # 主导出
```

---

## 数据流与交互

### Inbound Message Flow (入站消息流)

```
┌─────────────────────────────────────────────────────────────────┐
│ External Platform (Telegram/Discord/Slack/...)                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Webhook / Long Polling
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Channel Plugin (src/telegram/ or extensions/telegram/)          │
│  ┌──────────────┐                                               │
│  │ 1. Parse     │ - 解析消息格式                                 │
│  │ 2. Normalize │ - 转换为统一消息格式                           │
│  │ 3. Extract   │ - 提取媒体、元数据                             │
│  └──────────────┘                                               │
└───────────────────────────┬─────────────────────────────────────┘
                            │ NormalizedMessage
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Security Layer (src/channels/security*.ts)                      │
│  ┌──────────────────┐                                           │
│  │ DM Policy Check  │ - 是否允许 DM                             │
│  │ Allowlist Check  │ - 用户是否在白名单                        │
│  │ Command Gate     │ - 是否有权限执行命令                      │
│  │ Rate Limit       │ - 频率限制检查                            │
│  └──────────────────┘                                           │
└───────────────────────────┬─────────────────────────────────────┘
                            │ PolicyResult (allow/deny)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Routing Layer (src/routing/resolve-route.ts)                    │
│  ┌──────────────────────┐                                       │
│  │ Resolve Agent        │ - 根据 Channel + Account + Peer       │
│  │ Resolve Session Key  │ - 生成唯一会话键                      │
│  │ Apply Bindings       │ - 应用绑定规则                        │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ ResolvedAgentRoute
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Gateway Server (src/gateway/server.impl.ts)                     │
│  ┌──────────────────────┐                                       │
│  │ Queue Message        │ - 加入消息队列                        │
│  │ Load Session         │ - 加载或创建会话                      │
│  │ Check Debounce       │ - 防抖检查                            │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ QueuedMessage
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Agent Runtime (src/agents/, src/acp/)                           │
│  ┌──────────────────────┐                                       │
│  │ Load Agent Config    │ - 加载 Agent 配置                     │
│  │ Build Prompt         │ - 构建系统提示                        │
│  │ Inject Tools         │ - 注入可用工具                        │
│  │ Execute LLM Call     │ - 调用 AI 模型                        │
│  │ Handle Tool Calls    │ - 处理工具调用                        │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ AgentResponse
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Reply Dispatcher (src/auto-reply/reply/)                        │
│  ┌──────────────────────┐                                       │
│  │ Chunk Response       │ - 分块长消息                          │
│  │ Apply Prefix         │ - 添加前缀 [BotName]                  │
│  │ Send Ack Reaction    │ - 发送确认反应                        │
│  │ Stream Chunks        │ - 流式发送消息块                      │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ OutboundMessage[]
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Channel Plugin - Outbound Adapter                               │
│  ┌──────────────────────┐                                       │
│  │ Send Messages        │ - 调用平台 API 发送消息               │
│  │ Handle Errors        │ - 处理发送失败                        │
│  │ Retry Logic          │ - 重试机制                            │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
                    [Message Delivered]
```

### Outbound Message Flow (出站消息流)

```
┌─────────────────────────────────────────────────────────────────┐
│ Trigger Sources                                                 │
│  ┌────────────────┐ ┌────────────┐ ┌───────────────┐          │
│  │ Agent Response │ │ Cron Job   │ │ Heartbeat     │          │
│  └────────────────┘ └────────────┘ └───────────────┘          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Outbound Context Builder                                        │
│  ┌──────────────────────┐                                       │
│  │ Resolve Target       │ - 目标通道 + 账号 + 用户              │
│  │ Apply Policy         │ - 出站策略检查                        │
│  │ Format Message       │ - 格式化消息                          │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ OutboundContext
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Media Pipeline (src/media/)                                     │
│  ┌──────────────────────┐                                       │
│  │ Fetch Media          │ - 获取媒体文件                        │
│  │ Process/Convert      │ - 处理/转换格式                       │
│  │ Upload to Platform   │ - 上传到平台                          │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Channel Plugin - Outbound Adapter                               │
│  ┌──────────────────────┐                                       │
│  │ Platform API Call    │ - 调用平台 API                        │
│  │ Handle Rate Limits   │ - 处理速率限制                        │
│  │ Retry on Failure     │ - 失败重试                            │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
                    [Message Sent to Platform]
```

### Agent Execution Flow (Agent 执行流)

```
┌─────────────────────────────────────────────────────────────────┐
│ Agent Execution Request                                         │
│  - agentId: "default"                                           │
│  - sessionKey: "telegram:bot1:user:123"                         │
│  - message: "What's the weather?"                               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ ACP Runtime (src/acp/runtime.ts)                                │
│  ┌──────────────────────┐                                       │
│  │ Load Session         │ - 加载会话状态                        │
│  │ Load Agent Config    │ - 加载 Agent 配置                     │
│  │ Initialize Context   │ - 初始化上下文                        │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Prompt Builder                                                   │
│  ┌──────────────────────┐                                       │
│  │ System Prompt        │ - 身份、能力、约束                    │
│  │ Conversation History │ - 历史对话                            │
│  │ User Message         │ - 当前用户消息                        │
│  │ Memory Context       │ - 记忆上下文                          │
│  │ Tool Definitions     │ - 可用工具定义                        │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Prompt
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ LLM Provider Call (Pi Agent Core)                               │
│  ┌──────────────────────┐                                       │
│  │ Select Model         │ - 选择模型 (GPT-4, Claude, etc.)      │
│  │ API Call             │ - 调用 LLM API                        │
│  │ Stream Response      │ - 流式接收响应                        │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ StreamedResponse
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Response Processing                                             │
│  ┌──────────────────────┐                                       │
│  │ Text Content         │ - 文本内容                            │
│  │ Tool Calls           │ - 工具调用请求                        │
│  └──────────────────────┘                                       │
└───────────┬─────────────────────────────────────┬───────────────┘
            │                                     │
            │ Text                                │ Tool Calls
            ▼                                     ▼
┌───────────────────────┐           ┌─────────────────────────────┐
│ Return Text Response  │           │ Tool Execution Loop         │
└───────────────────────┘           │  ┌─────────────────────┐    │
                                    │  │ Select Tool         │    │
                                    │  │ Validate Params     │    │
                                    │  │ Check Approval      │    │
                                    │  │ Execute Tool        │    │
                                    │  │ Return Result       │    │
                                    │  └─────────────────────┘    │
                                    │  ┌─────────────────────┐    │
                                    │  │ Re-prompt LLM       │    │
                                    │  │ with Tool Results   │    │
                                    │  └─────────────────────┘    │
                                    └───────────┬─────────────────┘
                                                │
                                                ▼
                                    ┌─────────────────────────────┐
                                    │ Final Response              │
                                    └─────────────────────────────┘
```

### Plugin Lifecycle (插件生命周期)

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Discovery Phase                                              │
│  ┌──────────────────────┐                                       │
│  │ Scan extensions/     │ - 扫描 extensions/ 目录               │
│  │ Load package.json    │ - 加载 package.json                   │
│  │ Validate Manifest    │ - 验证 openclaw.plugin.json           │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Loading Phase                                                │
│  ┌──────────────────────┐                                       │
│  │ Dynamic Import       │ - 动态导入插件模块                    │
│  │ Create API Instance  │ - 创建 OpenClawPluginApi 实例        │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Registration Phase                                           │
│  ┌──────────────────────┐                                       │
│  │ plugin.register(api) │ - 调用插件的 register 方法            │
│  │  ├─ registerChannel()│   - 注册通道                         │
│  │  ├─ registerTool()   │   - 注册工具                         │
│  │  ├─ registerService()│   - 注册服务                         │
│  │  └─ registerHttpRoute│   - 注册 HTTP 路由                   │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Runtime Phase                                                │
│  ┌──────────────────────┐                                       │
│  │ Channel Handlers     │ - 通道事件处理                        │
│  │ Tool Execution       │ - 工具执行                            │
│  │ HTTP Route Handling  │ - HTTP 路由处理                       │
│  │ Service Operations   │ - 服务操作                            │
│  └──────────────────────┘                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Shutdown Phase                                               │
│  ┌──────────────────────┐                                       │
│  │ Cleanup Resources    │ - 清理资源                            │
│  │ Stop Services        │ - 停止服务                            │
│  │ Unregister Handlers  │ - 注销处理器                          │
│  └──────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键设计模式

### 1. Dependency Injection (依赖注入)

**应用场景**: CLI 命令、测试

**实现方式**:
```typescript
// src/cli/deps.ts
export type CliDeps = {
  loadConfig: typeof loadConfig;
  saveConfig: typeof saveConfig;
  // ... 所有外部依赖
};

export function createDefaultDeps(): CliDeps {
  return {
    loadConfig,
    saveConfig,
    // ... 实际实现
  };
}

// src/commands/gateway/run.ts
export async function runCommand(deps: CliDeps, opts: RunOptions) {
  const config = await deps.loadConfig();
  // ... 使用注入的依赖
}

// 测试时注入 mock
const mockDeps = {
  loadConfig: vi.fn().mockResolvedValue(mockConfig),
  saveConfig: vi.fn()
};
await runCommand(mockDeps, opts);
```

**优势**:
- 易于单元测试
- 松耦合
- 灵活替换实现

---

### 2. Adapter Pattern (适配器模式)

**应用场景**: Channel Plugin System

**实现方式**:
```typescript
// 定义适配器接口
type ChannelOutboundAdapter = {
  sendMessage: (ctx: OutboundContext) => Promise<SendResult>;
  sendReaction: (ctx: ReactionContext) => Promise<void>;
};

// Telegram 实现
const telegramOutbound: ChannelOutboundAdapter = {
  async sendMessage(ctx) {
    await ctx.api.sendMessage(ctx.chatId, ctx.text);
  },
  async sendReaction(ctx) {
    await ctx.api.setMessageReaction(ctx.chatId, ctx.messageId, ctx.emoji);
  }
};

// Discord 实现
const discordOutbound: ChannelOutboundAdapter = {
  async sendMessage(ctx) {
    await ctx.channel.send(ctx.text);
  },
  async sendReaction(ctx) {
    await ctx.message.react(ctx.emoji);
  }
};
```

**优势**:
- 统一接口，隐藏平台差异
- 易于扩展新通道
- 关注点分离

---

### 3. Plugin Architecture (插件架构)

**应用场景**: Extensions, Tools, Services

**核心机制**:
```typescript
// 1. 插件定义
const myPlugin = {
  id: "my-plugin",
  configSchema: { /* ... */ },
  register(api: OpenClawPluginApi) {
    // 注册功能
    api.registerChannel({ plugin: myChannelPlugin });
    api.registerTool(myTool);
  }
};

// 2. 插件运行时
class PluginRuntime {
  private registry = new Map<string, PluginHandle>();

  async loadPlugin(pluginPath: string) {
    const plugin = await import(pluginPath);
    const api = this.createApi(plugin.id);
    await plugin.register(api);
    this.registry.set(plugin.id, { plugin, api });
  }

  private createApi(pluginId: string): OpenClawPluginApi {
    return {
      registerChannel: (opts) => this.registerChannel(pluginId, opts),
      registerTool: (tool) => this.registerTool(pluginId, tool),
      // ...
    };
  }
}
```

**优势**:
- 高度可扩展
- 模块化
- 运行时加载

---

### 4. Strategy Pattern (策略模式)

**应用场景**: Routing, Security Policies

**实现方式**:
```typescript
// 路由策略
type RoutingStrategy = {
  match: (context: RoutingContext) => ResolvedAgentRoute | null;
};

const peerBindingStrategy: RoutingStrategy = {
  match(context) {
    const binding = context.bindings.peer?.[context.peerId];
    if (binding) {
      return { agentId: binding.agentId, matchedBy: "binding.peer" };
    }
    return null;
  }
};

const defaultStrategy: RoutingStrategy = {
  match(context) {
    return { agentId: "default", matchedBy: "default" };
  }
};

// 策略链
const strategies = [
  threadBindingStrategy,
  peerBindingStrategy,
  guildBindingStrategy,
  defaultStrategy
];

for (const strategy of strategies) {
  const result = strategy.match(context);
  if (result) return result;
}
```

**优势**:
- 算法可互换
- 易于添加新策略
- 符合开闭原则

---

### 5. Observer Pattern (观察者模式)

**应用场景**: Event System, Hooks

**实现方式**:
```typescript
// 事件总线
type GatewayEventMap = {
  "channel:health": ChannelHealthEvent;
  "agent:complete": AgentCompleteEvent;
  "config:reload": ConfigReloadEvent;
};

class EventEmitter<T extends Record<string, any>> {
  private listeners = new Map<keyof T, Set<Function>>();

  on<K extends keyof T>(event: K, listener: (data: T[K]) => void) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener);
  }

  emit<K extends keyof T>(event: K, data: T[K]) {
    const listeners = this.listeners.get(event);
    if (listeners) {
      listeners.forEach(listener => listener(data));
    }
  }
}

// 使用
eventEmitter.on("channel:health", (event) => {
  console.log(`Channel ${event.channelId} is ${event.status}`);
});
```

**优势**:
- 松耦合的事件处理
- 易于扩展监听器
- 支持多个订阅者

---

### 6. Factory Pattern (工厂模式)

**应用场景**: Tool Creation, Agent Instantiation

**实现方式**:
```typescript
// 工具工厂
type ChannelAgentToolFactory = (
  context: ToolFactoryContext
) => ChannelAgentTool[];

const telegramToolFactory: ChannelAgentToolFactory = (context) => {
  return [
    {
      name: "telegram_pin_message",
      description: "Pin a message in a Telegram chat",
      inputSchema: { /* ... */ },
      execute: async (params) => {
        await context.api.pinChatMessage(params.chatId, params.messageId);
      }
    }
  ];
};

// 注册工厂
plugin.agentTools = telegramToolFactory;

// 运行时调用
const tools = plugin.agentTools?.(context) ?? [];
```

**优势**:
- 延迟创建
- 依赖注入
- 上下文感知

---

### 7. Session Key Pattern (会话键模式)

**应用场景**: Session Isolation

**设计**:
```typescript
// 会话键编码
type SessionKeyComponents = {
  agentId: string;
  channel: string;
  accountId: string;
  peerKind: "user" | "chat" | "channel";
  peerId: string;
};

function buildSessionKey(components: SessionKeyComponents): string {
  return `${components.agentId}:${components.channel}:${components.accountId}:${components.peerKind}:${components.peerId}`;
}

// 示例
// "default:telegram:bot1:user:12345"
// "assistant:discord:main:chat:67890"
```

**优势**:
- 全局唯一标识
- 可解析的组件
- 支持多维度隔离

---

## 安全与治理

### 1. 多层安全模型

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: Network Security                                    │
│  ├─ SSRF Protection (禁止访问内网地址)                        │
│  ├─ Rate Limiting (请求频率限制)                              │
│  ├─ IP Allowlist (IP 白名单)                                  │
│  └─ CORS Policy (跨域策略)                                    │
└─────────────────────┬────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 2: Authentication                                      │
│  ├─ Token Authentication (令牌认证)                          │
│  ├─ Device Pairing (设备配对)                                │
│  ├─ OAuth 2.0 Flows (OAuth 流程)                             │
│  └─ Session Management (会话管理)                            │
└─────────────────────┬────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 3: Authorization (Channel Security)                    │
│  ├─ DM Policy (谁可以私聊 Bot)                               │
│  │   ├─ allowlist: 仅白名单用户                              │
│  │   ├─ paired: 仅已配对用户                                 │
│  │   ├─ public: 所有人                                      │
│  │   └─ disabled: 禁用私聊                                   │
│  ├─ Allowlist (用户白名单)                                   │
│  ├─ Command Gating (命令权限控制)                            │
│  │   ├─ allowFrom: 允许的用户列表                            │
│  │   ├─ denyFrom: 拒绝的用户列表                             │
│  │   ├─ requireRole: 需要的角色                              │
│  │   └─ requirePermission: 需要的权限                        │
│  └─ Role-based Access Control (角色访问控制)                 │
└─────────────────────┬────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 4: Execution Security                                  │
│  ├─ Tool Approval System (工具审批系统)                      │
│  │   ├─ Low Risk: 自动批准                                   │
│  │   ├─ Medium Risk: 需要确认                                │
│  │   └─ High Risk: 需要明确批准                              │
│  ├─ Sandbox Isolation (沙箱隔离)                             │
│  │   ├─ File System Sandbox                                  │
│  │   ├─ Network Sandbox                                      │
│  │   └─ Process Sandbox                                      │
│  └─ Command Approval Queue (命令审批队列)                    │
│      ├─ Interactive Approval (交互式审批)                    │
│      └─ Auto-approval Rules (自动审批规则)                   │
└─────────────────────┬────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 5: Data Security                                       │
│  ├─ Secret Management (密钥管理)                             │
│  │   ├─ 1Password Connect                                   │
│  │   ├─ AWS Secrets Manager                                 │
│  │   ├─ Environment Variables                               │
│  │   └─ Encrypted Local Storage                             │
│  ├─ Encryption at Rest (静态加密)                            │
│  │   ├─ AES-256 for sensitive data                          │
│  │   └─ Secure key derivation                               │
│  ├─ Audit Logging (审计日志)                                 │
│  │   ├─ Authentication events                               │
│  │   ├─ Authorization decisions                             │
│  │   ├─ Tool executions                                     │
│  │   └─ Configuration changes                               │
│  └─ Data Retention Policy (数据保留策略)                     │
└──────────────────────────────────────────────────────────────┘
```

### 2. SSRF 防护

**位置**: `src/infra/net/ssrf-guard.ts`

**实现**:
```typescript
type SSRFGuardConfig = {
  blockPrivateIPs: boolean;     // 阻止私有 IP
  blockLoopback: boolean;       // 阻止回环地址
  blockLinkLocal: boolean;      // 阻止链路本地地址
  allowedHosts?: string[];      // 允许的主机
};

async function fetchWithSSRFGuard(
  url: string,
  config: SSRFGuardConfig
): Promise<Response> {
  const parsedUrl = new URL(url);
  const resolved = await dnsLookup(parsedUrl.hostname);

  if (isPrivateIP(resolved) && config.blockPrivateIPs) {
    throw new Error("SSRF attempt blocked: private IP");
  }

  if (isLoopback(resolved) && config.blockLoopback) {
    throw new Error("SSRF attempt blocked: loopback address");
  }

  return fetch(url);
}
```

### 3. 命令审批系统

**位置**: `src/infra/exec-approvals.ts`

**工作流程**:
```
Agent Request to Execute Command
           │
           ▼
   ┌───────────────┐
   │ Risk Analysis │
   └───────┬───────┘
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
 Low Risk      High Risk
    │             │
    ▼             ▼
 Auto-approve  ┌──────────────┐
    │          │ Approval Queue│
    │          └───────┬──────┘
    │                  │
    │          ┌───────┴────────┐
    │          │                │
    │          ▼                ▼
    │       Approve          Deny
    │          │                │
    └──────────┴────────────────┘
               │
               ▼
         Execute or Abort
```

### 4. 审计日志

**记录内容**:
```typescript
type AuditLogEntry = {
  timestamp: Date;
  eventType: AuditEventType;
  actor: {
    type: "user" | "agent" | "system";
    id: string;
    channel?: string;
    accountId?: string;
  };
  action: string;
  resource: {
    type: string;
    id: string;
  };
  result: "success" | "failure" | "denied";
  metadata?: Record<string, unknown>;
};

type AuditEventType =
  | "auth.login"
  | "auth.logout"
  | "auth.pairing"
  | "channel.message.receive"
  | "channel.message.send"
  | "tool.execute"
  | "config.change"
  | "security.policy_violation";
```

---

## 总结

### 架构优势

1. **高可扩展性**
   - 插件化架构支持 40+ 通道扩展
   - Plugin SDK 提供稳定的公共 API
   - 工具系统支持动态注册

2. **高可用性**
   - Gateway 作为控制平面，支持热重载
   - Session 持久化支持故障恢复
   - 健康监控与自动重启

3. **安全性**
   - 多层安全模型
   - 细粒度权限控制
   - 审计日志与合规性

4. **开发者友好**
   - TypeScript 强类型
   - 依赖注入便于测试
   - 完善的 CLI 工具

### 关键技术亮点

- **通道无关路由**: 统一的路由层，支持多维绑定
- **Session Key 系统**: 全局唯一会话标识
- **媒体处理管道**: 统一的媒体处理流程
- **实时通信**: WebSocket + HTTP 混合架构
- **配置热重载**: 无缝配置更新

### 适用场景

- **企业级 AI 助手**: 多通道统一接入
- **DevOps 机器人**: 跨平台运维自动化
- **客户服务机器人**: 多渠道客服支持
- **团队协作工具**: Slack/Discord/Teams 集成

---

**文档版本**: 1.0
**生成日期**: 2026-03-07
**项目版本**: 2026.3.3
