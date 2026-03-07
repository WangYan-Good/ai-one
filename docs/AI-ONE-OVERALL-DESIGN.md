# AI-One 整体设计文档

**文档编号**: DESIGN-OVERALL-001  
**版本**: v1.0  
**创建日期**: 2026-03-07 13:30 UTC  
**作者**: AI-Secretary (小兰)  
**状态**: 🟡 待研讨小组审阅  
**审批流程**: 研讨小组审阅 → CEO 批准 → 推送远端仓库

---

## 📋 文档目的

本文档为 AI-One 项目的整体设计文档，供研讨小组审阅批准。文档涵盖:

1. **项目概述** - AI-One 的定位、目标、范围
2. **架构设计** - 系统架构、组件设计、技术选型
3. **核心功能** - 热更新系统、Agent 管理、通信机制
4. **开发流程** - 开发规范、测试策略、发布流程
5. **部署方案** - 部署架构、运维策略、监控告警

---

## 1. 项目概述 (Executive Summary)

### 1.1 项目背景

**AI-One** 是基于 OpenClaw  Fork 的定制化 Agent 平台，专为 CEO (主人) 个人使用习惯深度定制。

**为什么要做 AI-One?**

| 需求 | OpenClaw 现状 | AI-One 解决方案 |
|------|--------------|----------------|
| **专属定制** | 通用设计，无法深度定制 | Fork 独立演进，完全自主控制 |
| **热更新能力** | 手动部署，服务中断 5-10 分钟 | 自动检测 + 热加载，中断<30 秒 |
| **快速响应** | 依赖上游发布节奏 | 独立 CI/CD，自主发布 |
| **使用习惯适配** | 通用配置，无个性化 | 深度适配 CEO 使用习惯 |

### 1.2 项目目标

| 目标 | 说明 | 成功标准 (可量化) | 优先级 |
|------|------|------------------|--------|
| **平滑迁移** | 从 OpenClaw 无缝迁移到 AI-One | 0 数据丢失，0 配置重配 | P0 |
| **热更新能力** | 自动检测并应用新版本 | 检测<5 分钟，更新<2 分钟，中断<30 秒 | P0 |
| **专属定制** | 适配 CEO 使用习惯 | 支持自定义技能/配置/UI | P1 |
| **独立演进** | 建立独立 CI/CD | 版本独立管理，自主发布 | P1 |

### 1.3 项目范围

**Phase 1 (2026-03-05 ~ 2026-03-17)**:

- ✅ Fork OpenClaw 代码库
- ✅ 实现热更新系统 (版本检测 + 下载 + 热加载 + 回滚)
- ✅ 建立独立 CI/CD 流水线
- ✅ 数据迁移工具
- ✅ 兼容性测试

**Phase 2 (2026-03-18 ~ 2026-04-01)**:

- ⏳ 专属技能开发
- ⏳ UI 定制
- ⏳ 性能优化

**Phase 3 (2026-04-02 ~ )**:

- ⏳ 自有更新服务器
- ⏳ 灰度发布
- ⏳ 多节点管理

---

## 2. 系统架构 (System Architecture)

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      AI-One 架构概览                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              新增：热更新系统 (Phase 1)                  │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │ Version      │  │ Update       │  │ Hot          │  │   │
│  │  │ Detector     │─▶│ Manager      │─▶│ Loader       │  │   │
│  │  │ 版本检测器   │  │ 更新管理器   │  │ 热加载器     │  │   │
│  │  └──────────────┘  └──────┬───────┘  └──────────────┘  │   │
│  │                           │                             │   │
│  │                    ┌──────┴──────┐                     │   │
│  │                    │ Rollback    │                     │   │
│  │                    │ 回滚机制    │                     │   │
│  │                    └─────────────┘                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │  Gateway    │  │   Skills    │  │   Memory    │  ← 保留    │
│  │  (兼容)     │  │  (兼容 + 扩展)│  │  (兼容)     │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              AI-One 定制层 (Phase 2+)                    │   │
│  │  • 主人使用习惯适配  • 专属技能  • 定制 UI              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

#### 2.2.1 Gateway (网关服务)

**职责**: WebSocket 网关，连接 Channels 和 Agents

| 组件 | 说明 | 状态 |
|------|------|------|
| `src/gateway/` | 网关核心逻辑 | ✅ 保留 |
| `src/channels/` | 渠道插件 (Telegram/Discord/WhatsApp 等) | ✅ 保留 |
| `src/agents/` | Agent 运行时 | ✅ 保留 |

#### 2.2.2 Skills (技能系统)

**职责**: 提供工具和能力扩展

| 组件 | 说明 | 状态 |
|------|------|------|
| `skills/` | 内置技能 (weather, pdf, xlsx 等) | ✅ 保留 |
| `src/tools/` | 工具系统 | ✅ 保留 |
| `extensions/` | 扩展插件 | ✅ 保留 |

#### 2.2.3 Memory (记忆系统)

**职责**: 长期记忆和上下文管理

| 组件 | 说明 | 状态 |
|------|------|------|
| `src/memory/` | 记忆核心 | ✅ 保留 |
| `plugins/memory-lancedb/` | LanceDB 记忆插件 | ✅ 保留 |

#### 2.2.4 Updater (热更新系统) ⭐ 新增

**职责**: 版本检测、下载、热加载、回滚

| 组件 | 说明 | 状态 |
|------|------|------|
| `src/updater/VersionDetector.ts` | 版本检测器 | ✅ 已实现 |
| `src/updater/UpdateManager.ts` | 更新管理器 | ⏳ 待实现 |
| `src/updater/HotLoader.ts` | 热加载器 | ⏳ 待实现 |
| `src/updater/Rollback.ts` | 回滚机制 | ⏳ 待实现 |

---

## 3. 核心功能设计 (Core Features)

### 3.1 热更新系统 (Hot Update System)

#### 3.1.1 版本检测

**方案**: 轮询 GitHub Release API

```typescript
interface VersionDetector {
  checkForUpdates(): Promise<ReleaseInfo | null>;
  getCurrentVersion(): string;
  getLatestVersion(): Promise<string>;
}

interface ReleaseInfo {
  version: string;
  releaseNotes: string;
  downloadUrl: string;
  sha256: string;
  publishedAt: Date;
}
```

**检测流程**:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   轮询      │    │   比较      │    │   通知      │
│ GitHub API  │───▶│  版本号     │───▶│  管理员     │
│ (5 分钟)     │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
```

**配置示例**:

```json5
{
  aiOne: {
    update: {
      enabled: true,
      checkInterval: '5m',
      githubToken: '${GITHUB_TOKEN}',
      repo: 'WangYan-Good/ai-one'
    }
  }
}
```

#### 3.1.2 热加载

**方案**: Node.js `require.cache` 清除 + 优雅重启

**流程**:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  下载更新包  │    │  验证完整性  │    │  保存快照   │
│             │───▶│  (SHA256)   │───▶│  (回滚用)   │
└─────────────┘    └─────────────┘    └─────────────┘
                                              │
                                              ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  清理缓存   │◀───│  迁移状态   │◀───│  停止旧进程  │
│             │    │  (Memory)   │    │  (优雅)     │
└─────────────┘    └─────────────┘    └─────────────┘
      │
      ▼
┌─────────────┐
│  启动新进程  │
│  (新版本)    │
└─────────────┘
```

**关键代码**:

```typescript
async function hotUpdate(releaseInfo: ReleaseInfo) {
  // 1. 下载更新包
  const packagePath = await downloadUpdate(releaseInfo.downloadUrl);
  
  // 2. 验证完整性
  await verifySha256(packagePath, releaseInfo.sha256);
  
  // 3. 保存当前版本快照 (回滚用)
  await saveSnapshot(currentVersion);
  
  // 4. 优雅停止当前进程
  await gracefulShutdown();
  
  // 5. 清除 require.cache
  clearRequireCache();
  
  // 6. 启动新版本
  await spawnNewVersion(packagePath);
}
```

#### 3.1.3 回滚机制

**触发条件**:

- 新版本启动失败
- 新版本运行异常 (崩溃/错误率>阈值)
- 管理员手动触发回滚

**回滚流程**:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  检测故障   │    │  恢复快照   │    │  重启旧版本  │
│             │───▶│  (本地)     │───▶│             │
└─────────────┘    └─────────────┘    └─────────────┘
```

**配置示例**:

```json5
{
  aiOne: {
    update: {
      rollback: {
        enabled: true,
        autoRollback: true,
        maxRollbackVersions: 3,
        healthCheckTimeout: '5m'
      }
    }
  }
}
```

### 3.2 Agent 管理系统

#### 3.2.1 Agent 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Agent 层级结构                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Level 0: 秘书处                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ secretary (AI-Secretary 小兰)                   │   │
│  │ • 总负责人 • 协调所有 Agents                     │   │
│  └─────────────────────────────────────────────────┘   │
│                        │                                │
│  Level 1: 高级总监                                       │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│  │ cto     │  │ coo     │  │ auditor │  │ cpo     │   │
│  │ 技术    │  │ 运营    │  │ 审计    │  │ 产品    │   │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘   │
│       │            │            │            │         │
│  Level 2: 执行层                                        │
│  ┌─────┴─────┐  ┌─┴─┐  ┌────┴─────┐  ┌───┴────┐      │
│  │ pm/backend│  │...│  │finance/..│  │...     │      │
│  └───────────┘  └───┘  └──────────┘  └────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 3.2.2 Agent 通信机制

**配置文件**: `subagents.json`

```json
{
  "version": "2.1",
  "requester": "secretary",
  "allowAny": true,
  "agentAllowList": ["secretary", "cto", "coo", "cpo", ...],
  "communicationRules": {
    "peerDirect": true,
    "crossLevelUp": "emergency_only",
    "crossDepartment": "via_peer"
  },
  "agents": [
    {
      "id": "secretary",
      "level": 0,
      "department": "秘书处",
      "subordinates": ["cto", "coo", "auditor", "cpo"],
      "canWakeUp": ["cto", "coo", ...],
      "mode": "always_on"
    },
    {
      "id": "cto",
      "level": 1,
      "department": "软件服务部",
      "superior": "secretary",
      "subordinates": ["pm", "backend", "frontend", ...],
      "canWakeUp": ["pm", "backend", ...],
      "canBeWokenBy": ["secretary", "cto"],
      "mode": "on_demand"
    }
  ]
}
```

**通信规则**:

| 规则 | 说明 | 示例 |
|------|------|------|
| `peerDirect` | 同级可直接通信 | CTO ↔ COO |
| `crossLevelUp` | 向上通信需紧急情况 | PM → CTO (紧急) |
| `crossDepartment` | 跨部门通过同级中介 | 软件部 ↔ 市场部 (CTO ↔ COO) |

#### 3.2.3 Agent 唤醒机制

**问题**: `sessions_spawn` 工具受 `agents_list` 限制，当前只允许 `secretary`

**解决方案**:

1. **短期**: 使用 inbox + HEARTBEAT 机制被动唤醒
2. **长期**: 修改 OpenClaw 源码或配置，解除 `agents_list` 限制

**HEARTBEAT 配置示例**:

```markdown
# HEARTBEAT.md (CTO)

## 🚨 紧急任务检查 (每 5 分钟)

### Inbox 检查
- [ ] 检查 `inbox/` 目录中是否有新的紧急任务
- [ ] 检查任务紧急程度 (🔴 P0)
- [ ] 检查任务截止时间

### 紧急任务处理
**如果有紧急任务**:
1. ✅ 立即开始处理
2. ✅ 按照任务要求完成
3. ✅ 将结果写入指定路径
```

### 3.3 配置管理

#### 3.3.1 配置文件结构

```
~/.openclaw/
├── openclaw.json          # 主配置文件
├── subagents.json         # Agent 通信配置
├── agents/                # Agent 目录
│   ├── secretary/
│   ├── cto/
│   ├── coo/
│   └── ...
└── workspace-secretary/   # 工作区
    ├── SOUL.md
    ├── AGENTS.md
    ├── HEARTBEAT.md
    └── memory/
```

#### 3.3.2 配置项说明

**`openclaw.json`**:

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `agents.defaults.model` | 默认模型 | `bailian/qwen3-coder-next` |
| `agents.list[]` | Agent 列表 | `[{id: "secretary", ...}]` |
| `gateway` | 网关配置 | `{mode: "local", port: 18789}` |
| `plugins` | 插件配置 | `{memory-lancedb: {...}}` |

**`subagents.json`**:

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `allowAny` | 允许任意唤醒 | `true` |
| `agentAllowList` | Agent 白名单 | `["secretary", "cto", ...]` |
| `communicationRules` | 通信规则 | `{peerDirect: true, ...}` |
| `agents[]` | Agent 详情 | `[{id: "cto", level: 1, ...}]` |

---

## 4. 开发流程 (Development Workflow)

### 4.1 开发规范

#### 4.1.1 代码规范

- **语言**: TypeScript (主) + Shell (脚本)
- **格式化**: Prettier + ESLint
- **提交规范**: Conventional Commits
- **分支策略**: Git Flow

#### 4.1.2 提交规范

```
<type>(<scope>): <subject>

<body>

<footer>
```

**type 类型**:

| 类型 | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `docs` | 文档更新 |
| `style` | 代码格式 |
| `refactor` | 重构 |
| `test` | 测试 |
| `chore` | 构建/工具 |

**示例**:

```bash
feat(updater): 实现版本检测器

- 添加 VersionDetector 类
- 实现 GitHub Release API 轮询
- 添加 SHA256 完整性校验

Closes #123
```

### 4.2 测试策略

#### 4.2.1 测试层次

```
┌─────────────────────────────────────────┐
│           E2E Tests (端到端)             │
├─────────────────────────────────────────┤
│         Integration Tests (集成)         │
├─────────────────────────────────────────┤
│           Unit Tests (单元)              │
└─────────────────────────────────────────┘
```

#### 4.2.2 测试覆盖要求

| 组件 | 覆盖率要求 | 测试框架 |
|------|-----------|---------|
| 核心模块 | >80% | Vitest |
| 工具函数 | >90% | Vitest |
| UI 组件 | >70% | Vitest + Testing Library |
| E2E 流程 | 关键路径 100% | Playwright |

#### 4.2.3 测试命令

```bash
# 单元测试
pnpm test:unit

# 集成测试
pnpm test:integration

# E2E 测试
pnpm test:e2e

# 覆盖率报告
pnpm test:coverage
```

### 4.3 发布流程

#### 4.3.1 版本管理

**语义化版本**: `MAJOR.MINOR.PATCH`

- **MAJOR**: 不兼容的 API 变更
- **MINOR**: 向后兼容的功能新增
- **PATCH**: 向后兼容的问题修复

#### 4.3.2 发布步骤

```bash
# 1. 更新版本号
pnpm version minor

# 2. 创建 Git 标签
git tag -a v1.2.0 -m "Release v1.2.0"

# 3. 推送标签
git push origin v1.2.0

# 4. 创建 GitHub Release
gh release create v1.2.0 --generate-notes

# 5. 触发 CI/CD 流水线
# (自动构建、测试、部署)
```

#### 4.3.3 CI/CD 流水线

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm install
      - run: pnpm build
      - run: pnpm test
  
  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm install
      - run: pnpm build
      - uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
```

---

## 5. 部署方案 (Deployment)

### 5.1 部署架构

```
┌─────────────────────────────────────────────────────────┐
│                    部署架构                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │   GitHub    │    │   VPS       │    │   CEO       │ │
│  │  Releases   │───▶│  (生产)     │───▶│  (本地)     │ │
│  │             │    │             │    │             │ │
│  └─────────────┘    └──────┬──────┘    └──────┬──────┘ │
│                            │                   │       │
│                     ┌──────┴──────┐            │       │
│                     │   Memory    │            │       │
│                     │   LanceDB   │            │       │
│                     └─────────────┘            │       │
│                                                │       │
│                     ┌──────────────────────────┘       │
│                     │                                  │
│              ┌──────┴──────┐                          │
│              │  Channels   │                          │
│              │ Telegram/   │                          │
│              │ Discord/    │                          │
│              │ WhatsApp    │                          │
│              └─────────────┘                          │
│                                                       │
└─────────────────────────────────────────────────────────┘
```

### 5.2 部署步骤

#### 5.2.1 VPS 部署 (生产环境)

```bash
# 1. 安装依赖
curl -fsSL https://openclaw.ai/install.sh | bash

# 2. 配置环境变量
export GITHUB_TOKEN=xxx
export DASHSCOPE_API_KEY=xxx
export TELEGRAM_BOT_TOKEN=xxx

# 3. 启动 Gateway
openclaw gateway start

# 4. 验证状态
openclaw gateway status
```

#### 5.2.2 本地部署 (开发环境)

```bash
# 1. 克隆仓库
git clone git@github.com:WangYan-Good/ai-one.git
cd ai-one

# 2. 安装依赖
pnpm install

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env 文件

# 4. 启动开发服务器
pnpm dev

# 5. 运行测试
pnpm test
```

### 5.3 监控告警

#### 5.3.1 监控指标

| 指标 | 阈值 | 告警级别 |
|------|------|---------|
| CPU 使用率 | >80% | ⚠️ Warning |
| 内存使用率 | >90% | 🔴 Critical |
| 磁盘使用率 | >85% | ⚠️ Warning |
| 错误率 | >5% | 🔴 Critical |
| 响应时间 | >5s | ⚠️ Warning |

#### 5.3.2 告警配置

```json5
{
  monitoring: {
    enabled: true,
    metrics: {
      cpu: { threshold: 80, level: 'warning' },
      memory: { threshold: 90, level: 'critical' },
      disk: { threshold: 85, level: 'warning' },
      errorRate: { threshold: 5, level: 'critical' },
      responseTime: { threshold: 5000, level: 'warning' }
    },
    notifications: {
      channels: ['telegram', 'email'],
      recipients: ['ceo@ai-one.com']
    }
  }
}
```

---

## 6. 安全设计 (Security)

### 6.1 安全原则

| 原则 | 说明 |
|------|------|
| **最小权限** | 只授予必要的权限 |
| **默认安全** | 默认配置为安全模式 |
| **显式授权** | 高风险操作需显式授权 |
| **审计日志** | 所有操作记录日志 |

### 6.2 安全措施

#### 6.2.1 认证授权

- **Gateway Token**: 随机生成，存储在 `~/.openclaw/gateway-token`
- **Channel Pairing**: DM 配对码机制，1 小时过期
- **Command Authorization**: 命令白名单 + 用户授权

#### 6.2.2 数据加密

- **配置文件**: 敏感信息使用环境变量
- **Memory**: LanceDB 支持加密存储
- **通信**: TLS 1.3 (HTTPS/WSS)

#### 6.2.3 沙箱隔离

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: 'non-main',
        scope: 'agent',
        docker: {
          network: 'none',
          capDrop: ['ALL'],
          readOnlyRoot: true
        }
      }
    }
  }
}
```

---

## 7. 性能优化 (Performance)

### 7.1 性能目标

| 指标 | 目标值 | 当前值 |
|------|--------|--------|
| 冷启动时间 | <10s | ~15s |
| 热更新中断 | <30s | N/A |
| 消息响应时间 | <2s | ~3s |
| 内存占用 | <500MB | ~600MB |

### 7.2 优化策略

#### 7.2.1 启动优化

- **懒加载**: 按需加载模块
- **缓存**: 配置/模型缓存
- **并行**: 并行初始化独立组件

#### 7.2.2 内存优化

- **上下文剪枝**: 自动清理旧工具结果
- **压缩**: Memory 数据压缩存储
- **分页**: 大数据集分页加载

#### 7.2.3 网络优化

- **连接复用**: WebSocket 长连接
- **重试退避**: 指数退避重试
- **CDN**: GitHub Releases 使用 CDN

---

## 8. 风险与缓解 (Risks & Mitigation)

### 8.1 技术风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| **热更新失败** | 中 | 高 | 回滚机制 + 手动部署预案 |
| **数据丢失** | 低 | 高 | Memory 持久化 + 定期备份 |
| **兼容性问题** | 中 | 中 | 兼容性测试 + 渐进式迁移 |
| **性能下降** | 低 | 中 | 性能监控 + 优化预案 |

### 8.2 业务风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| **工期延误** | 中 | 中 | 优先级聚焦 + 弹性调度 |
| **需求变更** | 高 | 中 | 敏捷迭代 + 快速响应 |
| **资源不足** | 低 | 高 | AI 团队弹性调度 |

---

## 9. 附录 (Appendix)

### 9.1 术语表

| 术语 | 说明 |
|------|------|
| **Agent** | 独立的 AI 助手实例 |
| **Gateway** | WebSocket 网关服务 |
| **Channel** | 通信渠道 (Telegram/Discord 等) |
| **Skill** | 工具/能力扩展 |
| **Memory** | 长期记忆系统 |
| **HEARTBEAT** | 定期自动检查机制 |

### 9.2 参考文档

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [AI-One Phase 1 设计文档](phase1-design.md)
- [AI-One 技术设计文档](AI-One-DESIGN.md)
- [OpenClaw Vision](../VISION.md)

### 9.3 修订历史

| 版本 | 日期 | 作者 | 说明 |
|------|------|------|------|
| v1.0 | 2026-03-07 | AI-Secretary | 初始版本 |

---

## 📋 审批流程

### 研讨小组审阅

| 角色 | 姓名 | 日期 | 意见 |
|------|------|------|------|
| AI-CTO | | | |
| AI-COO | | | |
| AI-CPO | | | |
| AI-Auditor | | | |

### CEO 批准

| 项目 | 内容 |
|------|------|
| **审批人** | CEO (主人) |
| **审批日期** | |
| **审批结果** | ☐ 批准 / ☐ 驳回 / ☐ 需修改 |
| **审批意见** | |

---

*AI-One 整体设计文档 v1.0 | AI-Secretary (小兰) 📋*  
*创建时间：2026-03-07 13:30 UTC*
