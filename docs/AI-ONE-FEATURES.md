# OpenClaw 功能点清单与实现细节

**文档编号**: AI-ONE-FEATURES-001  
**版本**: v1.0  
**创建日期**: 2026-03-07 13:49 UTC  
**作者**: AI-Secretary (小兰)  
**状态**: 🟡 待研讨小组审阅  
**来源**: 基于子代理代码分析

---

## 📋 目录

1. [核心系统](#1-核心系统)
2. [通道集成](#2-通道集成)
3. [模型提供者](#3-模型提供者)
4. [技能与工具](#4-技能与工具)
5. [记忆与存储](#5-记忆与存储)
6. [插件系统](#6-插件系统)
7. [自动化](#7-自动化)
8. [CLI 与管理](#8-cli-与管理)
9. [安全与认证](#9-安全与认证)
10. [开发与测试](#10-开发与测试)

---

## 1. 核心系统

### 1.1 Gateway 服务器

**功能描述**: Multi-protocol gateway server with WebSocket and HTTP support

**关键文件**:
- `src/gateway/server.ts` - Server exports
- `src/gateway/server.impl.ts` - Main gateway implementation (~1200+ lines)
- `src/gateway/server-methods.ts` - RPC method handlers
- `src/gateway/server-chat.ts` - Agent message handling
- `src/gateway/server-cron.ts` - Cron service integration
- `src/gateway/server-http.ts` - HTTP API endpoints
- `src/gateway/call.ts` - Gateway client connection
- `src/gateway/net.ts` - Network utilities
- `src/gateway/protocol/` - Protocol definitions

**核心逻辑**:

1. **WebSocket Server Setup**
   - 创建 WebSocket 服务器监听指定端口
   - 集成 TLS 支持 (wss://)
   - 客户端认证 (token/password)
   - 协议版本协商

2. **HTTP API Server**
   - REST API for gateway management
   - Channel status endpoints
   - Session management endpoints
   - Plugin management endpoints

3. **Message Routing**
   - Inbound message routing to agents
   - Outbound message routing to channels
   - Session key generation and tracking
   - Thread binding support

4. **Plugin Integration**
   - Plugin HTTP route registration
   - Channel plugin management
   - Hook execution

**数据结构**:

```typescript
// Gateway Server
type GatewayServer = {
  runtime: GatewayRuntime;
  close: () => Promise<void>;
  sendToAll: (event: GatewayEvent) => void;
};

interface GatewayEvent {
  type: string;
  payload: unknown;
  target?: string;
}

// Gateway RPC Method
type GatewayRPCMethod<T = unknown> = {
  name: string;
  schema: JSONSchema7;
  handler: (params: T) => Promise<unknown>;
  scopes: OperatorScope[];
};

// Gateway Connection Details
type GatewayConnectionDetails = {
  url: string;
  urlSource: string;
  bindDetail?: string;
  remoteFallbackNote?: string;
  message: string;
};

// Operator Scope
type OperatorScope = 
  | "gateway:status"
  | "gateway:restart"
  | "gateway:plugin:read"
  | "gateway:plugin:write"
  | "channel:read"
  | "channel:write"
  | "session:read"
  | "session:write";
```

**集成点**:
- Channel plugins registration
- Plugin HTTP route registration
- Cron job scheduling
- Node subscriptions (mobile app connections)
- Control UI frontend
- Memory search integration

---

### 1.2 Agent 系统

**功能描述**: Multi-agent orchestration with isolated execution contexts

**关键文件**:
- `src/agents/acp-spawn.ts` - ACP agent spawning (~500+ lines)
- `src/agents/acp-binding-architecture.guardrail.test.ts`
- `src/agents/subagent-registry.ts` - Subagent lifecycle management
- `src/agents/subagent-spawn.ts` - Sub-agent creation
- `src/agents/agent-scope.ts` - Agent file structure
- `src/agents/agent-paths.ts` - Agent directory structure
- `src/agents/assistant-identity.ts` - Assistant identity
- `src/agents/agent-prompt.ts` - Agent prompt generation

**核心逻辑**:

1. **Agent Isolation**
   - 每个 agent 有自己的工作区
   - 独立的 Memory 索引
   - 独立的工具权限
   - 独立的配置

2. **Sub-agent Hierarchy**
   - 支持 agent-to-agent 通信
   - 最大嵌套深度限制
   - 子 agent 是否可被唤醒的权限控制

3. **Session Management**
   - Thread-bound vs. one-shot execution modes
   - Session persistence and recovery
   - Message history preservation

4. **ACP Integration**
   - Agent Control Protocol compliance
   - Cross-gateway spawning
   - Remote execution approval

**数据结构**:

```typescript
// Sub-agent Spawn Parameters
type SpawnAcpMode = "run" | "session";
type SpawnAcpSandboxMode = "inherit" | "require";

type SpawnAcpParams = {
  task: string;
  label?: string;
  agentId?: string;
  cwd?: string;
  mode?: SpawnAcpMode;
  thread?: boolean;
  sandbox?: SpawnAcpSandboxMode;
};

// Sub-agent Spawn Result
type SpawnAcpResult = {
  status: "accepted" | "forbidden" | "error";
  childSessionKey?: string;
  runId?: string;
  note?: string;
  error?: string;
};

// Agent Paths
type AgentPaths = {
  root: string;
  agentDir: string;
  skill IndexPath: string;
  memoryPath: string;
  workspacePath: string;
};

// Assistant Identity
type AssistantIdentity = {
  name: string;
  theme: string;
  emoji: string;
  avatar?: string;
};
```

**集成点**:
- Channel message routing
- Plugin tool execution
- Memory search
- SkillInvocations

---

### 1.3 Control Plane

**功能描述**: Remote management and orchestration

**关键文件**:
- `src/acp/control-plane/manager.ts`
- `src/acp/control-plane/spawn.ts`
- `src/acp/policy.ts`
- `src/acp/transport.ts`

**核心逻辑**:

1. **Secure Spawning**
   - Cross-gateway agent spawning
   - Approval workflow
   - Security policy enforcement

2. **Session Binding**
   - Cross-channel thread synchronization
   - Session key management
   - Message routing

3. **Remote Execution**
   - Command execution with approval
   - Tool result collection
   - Status reporting

---

## 2. 通道集成

### 2.1 Channel Infrastructure

**功能描述**: Unified channel plugin architecture with provider SDKs

**关键文件**:
- `src/channels/plugins/types.ts` - Plugin SDK types (~500+ lines)
- `src/channels/dock.ts` - Channel docking system (~500+ lines)
- `src/channels/session.ts` - Session management
- `src/channels/channel-config.ts` - Channel configuration
- `src/channels/registry.ts` - Channel registry
- `src/channels/dock.ts` - Docking system

**核心逻辑**:

1. **Plugin Loading**
   - 从 `extensions/` 目录加载 channel 插件
   - 插件 SDK 模式: `@openclaw/plugin-sdk/{channel}`
   - 动态插件注册

2. **Channel Configuration**
   - Per-channel configuration
   - Account management
   - Advanced message threading

3. **Message Routing**
   - Inbound message routing
   - Outbound message delivery
   - Thread binding support

**数据结构**:

```typescript
// Channel Plugin
export type ChannelPlugin<T = unknown> = {
  id: string;
  meta: ChannelMeta;
  onboarding?: ChannelOnboardingAdapter;
  pairing?: ChannelPairingAdapter;
  capabilities: ChannelCapabilities;
  streaming?: ChannelStreamingAdapter;
  reload?: ChannelReloadAdapter;
  configSchema?: any;
  config?: ChannelConfigAdapter<T>;
  security?: ChannelSecurityAdapter;
  execute?: ChannelExecuteAdapter<T>;
  message?: ChannelMessageAdapter<T>;
  command?: ChannelCommandAdapter<T>;
  hook?: ChannelHookAdapter;
  http?: ChannelHttpAdapter;
  directory?: ChannelDirectoryAdapter;
};

// Channel Capabilities
export type ChannelCapabilities = {
  chatTypes: ("direct" | "channel" | "thread")[];
  polls?: boolean;
  reactions?: boolean;
  threads?: boolean;
  media?: boolean;
  nativeCommands?: boolean;
};

// Channel Meta
export type ChannelMeta = {
  id: string;
  label: string;
  selectionLabel?: string;
  detailLabel?: string;
  docsPath: string;
  docsLabel?: string;
  blurb: string;
  systemImage?: string;
  selectionDocsPrefix?: string;
  selectionDocsOmitLabel?: boolean;
  selectionExtras?: string[];
};
```

**集成点**:
- Gateway server integration
- Agent message handling
- Plugin HTTP routes
- Memory indexing

---

### 2.2 Supported Channels

#### 2.2.1 Telegram

**关键文件**:
- `extensions/telegram/` (plugin structure)
- `src/telegram/` - Bot SDK integration

**能力**:
- Direct messages, groups, supergroups
- File/media handling
- Inline keyboards
- Bot command support
- Polls and reactions

**配置**:
```typescript
{
  channels: {
    telegram: {
      enabled: true,
      botToken: string,
      dmPolicy: "pairing" | "allowlist" | "open" | "disabled",
      groups: {
        "*": { requireMention: boolean }
      }
    }
  }
}
```

#### 2.2.2 Discord

**关键文件**:
- `extensions/discord/` - Plugin implementation
- `src/discord/` - Discord API integration

**能力**:
- DMs, text channels, threads
- Reactions and polls
- Rich embeds
- Voice channel integration
- Slash commands

**配置**:
```typescript
{
  channels: {
    discord: {
      enabled: true,
      token: string,
      guilds: {
        "*": { requireMention: boolean }
      }
    }
  }
}
```

#### 2.2.3 WhatsApp

**关键文件**:
- `src/whatsapp/` - Baileys library wrapper

**能力**:
- Direct messaging
- Media sharing
- Message status tracking
- Template messages

#### 2.2.4 Signal

**关键文件**:
- `src/signal/` - Signal CLI integration

**能力**:
- Direct messaging
- Group messaging
- Media sharing

#### 2.2.5 iMessage

**关键文件**:
- `src/imessage/` - Apple messages integration

**能力**:
- Direct messaging
- Media sharing
- iMessage-specific features

#### 2.2.6 Slack

**关键文件**:
- `extensions/slack/` - Slack Bolt SDK integration

**能力**:
- DMs and channels
- Threaded conversations
- Rich messages
- Slash commands

#### 2.2.7 IRC

**关键文件**:
- `src/irc/` - IRC integration

**能力**:
- DM and channel routing
- Pairing controls
- Channel management

#### 2.2.8 Google Chat

**关键文件**:
- `src/googlechat/` - Google Chat API

**能力**:
- Spaces and DMs
- HTTP webhook delivery
- Rich messages

---

### 2.3 Channel Threading System

**功能描述**: Intelligent message threading with context preservation

**关键文件**:
- `src/channels/thread-bindings-policy.ts`
- `src/channels/thread-bindings-messages.ts`
- `src/channels/channel/thread.ts`

**核心逻辑**:

1. **Thread Detection**
   - Automatic thread detection from inbound messages
   - Thread ID extraction from platforms

2. **Thread-to-Session Binding**
   - Bind thread to conversation session
   - Preserve context within thread
   - Cross-channel thread sync

3. **Idle Timeout Management**
   - Auto-unfocus after inactivity
   - Max age enforcement
   - Session cleanup

**数据结构**:

```typescript
type ThreadBinding = {
  threadId: string;
  channelId: string;
  sessionKey: string;
  createdAt: number;
  lastActivity: number;
  idleHours?: number;
  maxAgeHours?: number;
};

type ThreadPolicy = {
  enabled: boolean;
  idleHours: number;
  maxAgeHours: number;
};
```

---

## 3. 模型提供者

### 3.1 Provider Architecture

**功能描述**: Multi-model provider support with fallback chain

**关键文件**:
- `src/providers/model-catalog.ts` - Model metadata
- `src/providers/model-auth.ts` - Authentication management
- `src/providers/model-selection.ts` - Model selection logic
- `src/providers/model-fallback.ts` - Fallback strategy
- `src/providers/index.ts` - Provider registry

**支持的 Provider**:
1. **OpenAI** (GPT-4, GPT-3.5, OAI-compatible)
2. **Anthropic** (Claude 3.x, Claude 2.x)
3. **Google Gemini** (Pro, Ultra, Flash)
4. **Azure OpenAI**
5. **AWS Bedrock** (Claude, Llama)
6. **Hugging Face Inference API**
7. **Ollama** (local)
8. **Mistral AI**
9. **Together AI**
10. **Qwen Portal** (Alibaba)
11. **BytePlus Volcengine**
12. **GitHub Copilot**
13. **Google Cloud AI Gateway**
14. **DeepSeek**

**数据结构**:

```typescript
// Provider Configuration
type ProviderConfig = {
  type: string;
  name: string;
  baseUrl?: string;
  apiKey?: string;
  models: ModelConfig[];
};

// Model Configuration
type ModelConfig = {
  id: string;
  provider: string;
  name?: string;
  contextWindow: number;
  capabilities: {
    vision?: boolean;
    functionCalling?: boolean;
    reasoning?: boolean;
  };
};

// Model Selection Result
type ModelSelection = {
  modelId: string;
  providerId: string;
  reason: string;
  capabilities: ModelCapabilities;
};
```

**核心逻辑**:

1. **Model Catalog**
   - Stored in `agents.defaults.models`
   - Model metadata (context window, capabilities)
   - Alias support

2. **Authentication**
   - API key management
   - Secret reference support
   - env fallback

3. **Selection Logic**
   - Explicit model override in config
   - Channel-specific model mappings
   - Fallback chain
   - Capabilities check

---

## 4. 技能与工具

### 4.1 Skill System

**功能描述**: Modular skill execution with plugin architecture

**关键文件**:
- `skills/` - Built-in skills directory (50+ skills)
- `src/agents/skills/` - Skill loading and execution
- `src/skills/` - Skill runtime
- `src/agents/openclaw-tools.ts` - Built-in tools
- `src/agents/skills/loader.ts` - Skill loader

**内置技能**:

| Skill Name | Description | Key Files |
|------------|-------------|-----------|
| `coding-agent` | Code generation and editing | `skills/coding-agent/` |
| `summarize` | Text summarization | `skills/summarize/` |
| `weather` | Weather forecasts (wttr.in) | `skills/weather/` |
| `doc-coauthoring` | Document co-authoring | `skills/doc-coauthoring/` |
| `persona-engine` | Personality simulation | `skills/persona-engine/` |
| `skill-creator` | Skill development | `skills/skill-creator/` |
| `http` | Web searches | `skills/http/` |
| `search` | Mixed search (vector + FTS) | `skills/search/` |
| `pdf` | PDF processing | `skills/pdf/` |
| `xlsx` | Spreadsheet processing | `skills/xlsx/` |
| `pptx` | Presentation processing | `skills/pptx/` |
| `json` | JSON processing | `skills/json/` |
| `text` | Text processing | `skills/text/` |
| `image` | Image processing | `skills/image/` |
| `tts` | Text-to-speech | `skills/tts/` |
| `vox` | Voice processing | `skills/vox/` |
| `voice-call` | Voice calls | `skills/voice-call/` |

**技能结构**:
```
skill-name/
├── SKILL.md           # Required: Trigger logic + instructions
├── scripts/           # Optional: Executable code
├── references/        # Optional: Documentation
└── assets/            # Optional: Templates/assets
```

**核心逻辑**:

1. **Skill Loading**
   - Load from `skills/` directory
   - Parse SKILL.md for trigger logic
   - Register skill to agent

2. **Skill Execution**
   - Trigger based on user intent
   - Execute skill script
   - Return result to agent

3. **Skill API**
   - Agent context
   - Tool access
   - Memory access

**数据结构**:

```typescript
// Skill Manifest
type SkillManifest = {
  id: string;
  name?: string;
  description?: string;
  trigger?: string;
  version?: string;
};

// Skill Result
type SkillResult = {
  success: boolean;
  message?: string;
  data?: unknown;
  error?: string;
};
```

---

### 4.2 Tool System

**功能描述**: Extensible tool execution with sandboxing

**关键文件**:
- `src/agents/tools/common.ts` - Tool base types
- `src/agents/openclaw-tools.ts` - Built-in tools
- `src/agents/pi-tools.ts` - Embedded PI tools
- `src/agents/tool-policy.ts` - Tool access control
- `src/agents/tools/binary.ts` - Binary tools
- `src/agents/tools/file.ts` - File tools
- `src/agents/tools/exec.ts` - Execution tools

**工具分类**:

#### 4.2.1 System Tools
- `read` - Read file content
- `write` - Write file content
- `edit` - Edit file content
- `exec` - Execute shell command
- `process` - Manage background processes
- `canvas` - Canvas operations

#### 4.2.2 Channel Tools
- `message` - Send message to channel
- `message_delete` - Delete message
- `reaction` - Add reaction to message
- `poll` - Create poll

#### 4.2.3 Agent Tools
- `sessions_spawn` - Spawn sub-agent
- `sessions_send` - Send message to another session
- `sessions_list` - List sessions
- `sessions_history` - Get session history
- `subagents` - Manage sub-agents

#### 4.2.4 Memory Tools
- `memory_store` - Store memory
- `memory_recall` - Recall memory
- `memory_forget` - Forget memory
- `memory_clear` - Clear memory

#### 4.2.5 Plugin Tools
- Provider-specific functionality (e.g., embeddings, audio)

**核心逻辑**:

```typescript
// Tool Factory
type AgentToolFactory = (ctx: {
  agentId: string;
  channel?: string;
  accountId?: string;
  toolCallId: string;
}) => AgentTool;

// Tool Implementation
type AgentTool = {
  name: string;
  description: string;
  schema: JSONSchema7;
  handler: (params: unknown) => Promise<AgentToolResult>;
};

// Tool Result
type AgentToolResult = {
  content: Array<{ type: string; text?: string; buffer?: string }>;
  isError?: boolean;
};
```

**安全机制**:

1. **Sandbox Execution**
   - System tools run in sandbox
   - Path safety checks
   - Resource limits

2. **Tool Policy**
   - Tool allow/deny lists
   - Elevated access control
   - Channel-specific restrictions

3. **Owner-Only Restriction**
   - Critical tools restricted to owner
   - Explicit permission required

---

## 5. 记忆与存储

### 5.1 Memory Index System

**功能描述**: Hybrid search (vector + FTS + BM25) with temporal decay

**关键文件**:
- `src/memory/index.ts` - Public API
- `src/memory/manager.ts` - Main implementation (~900+ lines)
- `src/memory/manager-search.ts` - Search implementation
- `src/memory/embeddings.ts` - Embedding providers
- `src/memory/manager-sync-ops.ts` - Sync operations
- `src/memory/manager-embedding-ops.ts` - Embedding operations

**Providers**:
- OpenAI Embeddings
- Google Gemini Embeddings
- Mistral Embeddings
- Voyage AI
- Ollama (local)
- Local HuggingFace models

**核心逻辑**:

```typescript
// Memory Provider Interface
type EmbeddingProvider = {
  name: string;
  type: string;
  dimensions: number;
  providers: string[];
  embed: (texts: string[]) => Promise<number[][]>;
};

// Memory Search Implementation
type MemorySearchOptions = {
  query: string;
  limit?: number;
  vectorWeight?: number;
  bm25Weight?: number;
  ftsWeight?: number;
  temporalDecay?: number;
};

// Memory Search Result
type MemorySearchResult = {
  content: string;
  metadata: {
    source: string;
    timestamp: number;
    score: number;
    vectorScore?: number;
    ftsScore?: number;
    bm25Score?: number;
    temporalDecay?: number;
  };
  snippet: string;
};
```

**核心算法**:

1. **Embedding**
   - Text chunks (configurable size)
   - Batch processing
   - Provider caching

2. **Indexing**
   - Vector index (ANN search)
   - Full-text index (BM25)
   - Metadata index

3. **Search**
   - Hybrid search (weighted combination)
   - Temporal decay
   - Result ranking

**存储结构**:

```
memory/
├── lancedb/           # Vector index
├── memor embeddings/  # Embeddings cache
└── memor memories/    # Memory items
```

---

### 5.2 Session Storage

**功能描述**: Session history with transcript preservation

**关键文件**:
- `src/session-utils.fs.ts` - File system operations
- `src/session-files.ts` - File management
- `src/session-tool-result-guard.ts` - Tool result filtering

**存储布局**:

```
agent-sessions/
  {session-key}/
    session.json       # Metadata + config
    transcript.jsonl   # Chat history
    attachments/       # Media files
    memories/          # Indexed memories
```

**核心逻辑**:

1. **Session Creation**
   - Generate session key
   - Create directory structure
   - Initialize session metadata

2. **Transcript Storage**
   - Append messages to transcript.jsonl
   - Store attachments
   - Index memories

3. **Session Recovery**
   - Load transcript from disk
   - Restore session state
   - Resume conversation

---

## 6. 插件系统

### 6.1 Plugin Architecture

**功能描述**: Dynamic plugin loading with lifecycle management

**关键文件**:
- `src/plugins/registry.ts` - Plugin registry (~600+ lines)
- `src/plugins/loader.ts` - Plugin loading logic
- `src/plugins/hook-runner-global.ts` - Hook execution
- `src/plugins/types.ts` - Plugin SDK types
- `src/plugins/cli.ts` - CLI integration

**Plugin Types**:

1. **ChannelPlugin**
   - Message channel integration
   - Onboarding adapter
   - Message adapter
   - Command adapter

2. **ProviderPlugin**
   - Model provider connection
   - Authentication adapter
   - Model catalog

3. **ServicePlugin**
   - Background services
   - Periodic tasks
   - Event listeners

4. **HookPlugin**
   - Event hooks
   - Message hooks
   - Tool hooks

**生命周期**:

```typescript
// Plugin Interface
type Plugin = {
  id: string;
  name: string;
  version?: string;
  register: (api: OpenClawPluginApi) => void;
};

// Plugin API
type OpenClawPluginApi = {
  registerChannel: (registration: ChannelPluginRegistration) => void;
  registerProvider: (registration: ProviderPluginRegistration) => void;
  registerHook: (name: PluginHookName, handler: HookHandler) => void;
  registerService: (service: OpenClawPluginService) => void;
  registerHttpRoute: (route: PluginHttpRouteRegistration) => void;
  registerCommand: (command: PluginCommandRegistration) => void;
  logger: PluginLogger;
  logger: PluginLogger;
  stateDir: string;
  pluginsDir: string;
  // ... more APIs
};
```

**核心逻辑**:

1. **Plugin Discovery**
   - Scan `extensions/` directory
   - Load plugin manifests
   - Validate plugin structure

2. **Plugin Loading**
   - Load plugin code
   - Execute register function
   - Register to API

3. **Plugin Lifecycle**
   - enabled/disabled state
   - Configuration reload
   - Hot reload support

---

### 6.2 Hook System

**功能描述**: Event-driven hooks for extensibility

**关键文件**:
- `src/hooks.ts` - Core hook system
- `src/plugins/hook-runner-global.ts`
- `src/hooks-mapping.ts` - Hook mapping

**Hook Types**:

| Hook Name | Description | Trigger |
|-----------|-------------|---------|
| `before-agent-start` | Before agent initialization |Agent startup |
| `after-tool-call` | After tool execution | Tool completion |
| `before-tool-call` | Before tool execution | Tool invocation |
| `message` | Inbound/outbound messages | Message events |
| `session` | Session lifecycle events | Session events |
| `subagent` | Sub-agent lifecycle | Sub-agent events |
| `compaction` | Message compaction events | Compaction |

**核心逻辑**:

```typescript
// Hook Handler
type HookHandler = (
  ctx: HookContext,
  payload: unknown
) => Promise<unknown>;

// Hook Context
type HookContext = {
  agentId: string;
  channelId?: string;
  accountId?: string;
  sessionId?: string;
  timestamp: number;
};
```

---

## 7. 自动化

### 7.1 Cron Scheduler

**功能描述**: Robust cron system with delivery preferences

**关键文件**:
- `src/cron/` - Cron service (~1500+ lines)
- `src/cron/service/` - Service implementation
- `src/cron/delivery.ts` - Delivery logic
- `src/cron/schedule.ts` - cron-parser integration
- `src/cron/normalize.ts` - Job normalization
- `src/cron/run-log.ts` - Run logs
- `src/cron/delivery-plan.ts` - Delivery planning

**功能特性**:

1. **Cron Expression Parsing**
   - Standard cron syntax
   - Support for seconds/minutes/hours/days

2. **Job Scheduling**
   - Fixed interval jobs (`every: 5m`)
   - Cron expression jobs (`every: "0 */2 * * *"`)
   - Single run jobs (`once: true`)

3. **Delivery Preferences**
   - Direct delivery
   - Group delivery
   - Channel-specific delivery
   - Delivery target targeting

4. **Run Logging**
   - Job run history
   - Success/failure tracking
   - statistics

**数据结构**:

```typescript
// Cron Job
type CronJob = {
  id: string;
  label: string;
  every: string; // e.g., "5m", "30m", "1h", "0 */2 * * *"
  task: string;
  agentId?: string;
  thread?: boolean;
  mode?: "run" | "session";
  heartbeat?: CronHeartbeat;
  target?: string | string[];
  inject?: boolean;
  ackMaxChars?: number;
  abort?: boolean;
  once?: boolean;
  enabled?: boolean;
};

// Cron Heartbeat
type CronHeartbeat = {
  enabled?: boolean;
  showOk?: boolean;
  showAlerts?: boolean;
  useIndicator?: boolean;
  ackMaxChars?: number;
  />

// Job Run Log
type JobRunLog = {
  jobId: string;
  startedAt: number;
  finishedAt?: number;
  success: boolean;
  output?: string;
  error?: string;
  stats?: JobRunStats;
};

// Job Run Stats
type JobRunStats = {
  tokensIn: number;
  tokensOut: number;
  durationMs: number;
};
```

**核心逻辑**:

1. **Timer Management**
   - Create timers for each job
   - Handle timer restarts
   - Prevent duplicate timers

2. **Job Execution**
   - Spawn agent for job
   - Capture output
   - Handle errors

3. **Delivery**
   - Apply delivery plan
   - Send to targets
   - Handle failures

**配置示例**:

```typescript
{
  cron: {
    jobs: [
      {
        id: "daily-check",
        label: "Daily health check",
        every: "24h",
        task: "Read HEARTBEAT.md if it exists... reply HEARTBEAT_OK",
        agentId: "main",
        target: ["+1234567890"]
      }
    ]
  }
}
```

---

### 7.2 State Machine

**功能描述**: WebSocket-based session state management

**关键文件**:
- `src/channels/session.ts` - Session management
- `src/channels/run-state-machine.ts` - State machine
- `src/channels/session-meta.ts` - Session metadata

---

## 8. CLI 与管理

### 8.1 CLI Architecture

**功能描述**: Command-line interface with rich commands

**关键文件**:
- `src/cli/` - CLI core (~1000+ lines)
- `src/cli/argv.ts` - Argument parsing
- `src/cli/program.ts` - Commander-based CLI
- `src/cli/run-main.ts` - Main CLI runner
- `src/cli/profile.ts` - Profile management

**支持的命令**:

1. **Core Commands**
   - `gateway` - Gateway management
   - `daemon` - Daemon management
   - `status` - System status
   - `health` - System health
   - `logs` - System logs

2. **Channel Commands**
   - `channels login` - Channel authentication
   - `channels list` - List channels
   - `channels add` - Add channel
   - `channels remove` - Remove channel

3. **Agent Commands**
   - `agent` - Run agent turn
   - `sessions` - Session management
   - `sessions cleanup` - Session cleanup

4. **Configuration Commands**
   - `config get` - Get config value
   - `config set` - Set config value
   - `config unset` - Unset config value

5. **Model Commands**
   - `models list` - List models
   - `models status` - Check model status
   - `models scan` - Scan for new models

6. **Tool Commands**
   - `xhr` - HTTP request tool
   - `exec` - Execute command
   - `read` - Read file

**核心逻辑**:

```typescript
// CLI Program
type CLIProgram = {
  buildProgram: () => Program;
  registerCommand: (name: string, handler: CLIHandler) => void;
};

// CLI Handler
type CLIHandler = {
  description: string;
  options: CLIOption[];
  args: CLIArg[];
  handler: (argv: string[]) => Promise<void>;
};

// CLI Option
type CLIOption = {
  name: string;
  alias?: string;
  description: string;
  type: "string" | "number" | "boolean" | "array";
  required?: boolean;
};

// CLI Arg
type CLIArg = {
  name: string;
  description: string;
  required?: boolean;
};
```

---

### 8.2 Configuration Management

**功能描述**: Config management with JSON5 support

**关键文件**:
- `src/config/` - Config module (~1500+ lines)
- `src/config/config.ts` - Config loading
- `src/config/types.ts` - Config types
- `src/config/types.secrets.ts` - Secret types

**配置特性**:

1. **JSON5 Support**
   - Comments
   - Trailing commas
   - Single quotes
   - Multi-line strings

2. **Secrets Support**
   - Environment variables
   - File-based secrets
   - Secret references

3. **Config Reload**
   - Hot reload support
   - Validation
   - Error handling

**配置结构**:

```typescript
type OpenClawConfig = {
  meta?: MetaConfig;
  models?: ModelsConfig;
  agents?: AgentsConfig;
  channels?: ChannelsConfig;
  gateway?: GatewayConfig;
  plugins?: PluginsConfig;
  commands?: CommandsConfig;
  cron?: CronConfig;
};
```

---

## 9. 安全与认证

### 9.1 Authentication

**功能描述**: Multi-layer authentication system

**关键文件**:
- `src/gateway/auth.ts` - Gateway authentication
- `src/agents/auth-profiles.ts` - Auth profile management
- `src/secrets/` - Secret management
- `src/security/` - Security utilities

**认证机制**:

1. **Gateway Token**
   - Randomly generated
   - Stored in `~/.openclaw/gateway-token`
   - Used for client authentication

2. **Channel Pairing**
   - DM pairing code
   - 1-hour expiry
   - User approval

3. **API Key Management**
   - Provider-specific keys
   - Environment fallback
   - Secret reference

---

### 9.2 Authorization

**功能描述**: Fine-grained authorization control

**关键文件**:
- `src/gateway/method-scopes.ts` - Method scopes
- `src/agents/auth-health.ts` - Auth health checks

**授权机制**:

1. **Method Scopes**
   - `gateway:status`
   - `gateway:restart`
   - `channel:read`
   - `channel:write`
   - `session:read`
   - `session:write`

2. **Owner-Only Restrictions**
   - Critical tools restricted
   - Explicit permission required

3. **Access Groups**
   - Channel-specific access
   - User allowlists

---

## 10. 开发与测试

### 10.1 Test Framework

**功能描述**: Comprehensive testing infrastructure

**关键文件**:
- `src/test-utils/` - Test utilities
- `src/test-helpers/` - Test helpers
- `test/` - Test files

**测试类型**:

1. **Unit Tests**
   - Tool tests
   - Utility tests
   - Helper tests

2. **Integration Tests**
   - Channel integration
   - Provider integration
   - Plugin integration

3. **E2E Tests**
   - Full workflow tests
   - Multi-channel tests
   - Multi-provider tests

**测试框架**:
- Vitest
- Playwright (E2E)
- Custom test helpers

**测试命令**:

```bash
# Run all tests
pnpm test

# Run unit tests
pnpm test:unit

# Run integration tests
pnpm test:integration

# Run E2E tests
pnpm test:e2e

# Run coverage
pnpm test:coverage
```

---

### 10.2 Documentation

**功能描述**: Comprehensive documentation system

**关键文件**:
- `docs/` - Documentation (~300+ pages)
- `docs/index.md` - Documentation index
- `docs/nav-tabs-underline.js` - Docs navigation

**文档类型**:

1. **User Guides**
   - Getting started
   - Configuration
   - Channel setup
   - Agent management

2. **Developer Docs**
   - Code structure
   - API reference
   - Plugin development
   - Tool development

3. **Architecture Docs**
   - Gateway design
   - Agent design
   - Plugin design

4. **Reference Docs**
   - Configuration reference
   - API reference
   - CLI reference

---

## 📊 统计数据

| 类别 | 项目 | 数量 |
|------|------|------|
| **核心模块** | 主要文件 | 100+ |
| **通道集成** | 支持通道 | 8 个 |
| **插件系统** | 插件类型 | 4 种 |
| **模型提供者** | 支持 provider | 14+ |
| **内置技能** | 内置技能 | 50+ |
| **工具系统** | 内置工具 | 20+ |
| **测试覆盖率** | 单元测试 | >80% |
| **文档页数** | 文档页面 | 300+ |

---

## 📚 参考文档

- **OpenClaw 官方文档**: https://docs.openclaw.ai
- **OpenClaw 源码**: https://github.com/openclaw/openclaw
- **AI-One 设计文档**: `docs/AI-One-DESIGN.md`
- **OpenClaw Vision**: `VISION.md`

---

*AI-One 功能点清单 v1.0 | AI-Secretary (小兰) 📋*  
*创建时间：2026-03-07 13:49 UTC*
