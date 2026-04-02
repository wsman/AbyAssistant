# 社区版解耦 Anthropic 官方服务的 Provider 架构设计

> 目标：在保留当前 `src/` 代码体系与已规划的“权威服务器 + 双模客户端 + 工作区体系”前提下，把所有依赖 Anthropic 官方账户、Claude.ai OAuth、CCR 云端环境、claude.ai/code、Anthropic Hosted Connectors、Anthropic Voice Stream 的功能，抽象成可替换的 Provider 能力面，使社区版可以同时支持：**官方服务实现、第三方托管实现、社区自托管实现**。

---

## 1. 设计背景

当前项目中有一批能力虽然在 CLI/本地 UI 中被调用，但其真实可用性建立在 Anthropic 官方服务之上，例如：

- Claude.ai OAuth 登录
- claude.ai subscription entitlement
- remote session / CCR 远端执行
- remote-control / bridge / claude.ai/code
- hosted connectors / claude.ai/settings/connectors
- voice stream

如果社区版仍然在业务层直接写死这些能力与 Anthropic 官方服务的绑定关系，那么即使新增了权威服务器、双模客户端和工作区体系，也仍然只是“官方服务的前端壳”，而不是一个真正解耦、可社区演化的版本。

因此，本设计文档提出一个新的核心原则：

> **任何当前依赖 Anthropic 官方账户、Claude.ai OAuth、CCR、claude.ai/code、Anthropic Hosted Connectors 或 Anthropic Voice Stream 的能力，都不得在业务层作为唯一实现写死，而必须经由 Provider 抽象层暴露为可替换能力。**

---

## 2. 设计目标

本设计的目标有四个：

1. **把 Anthropic 强耦合能力拆成可替换的 Provider 能力面**
2. **允许官方实现、第三方实现、自托管实现并存**
3. **不破坏当前 `src/` 的离线能力与本地运行时能力**
4. **让权威服务器与客户端适配层能够在 Provider 层之上演化，而不是继续把官方服务当成唯一上游**

---

## 3. 非目标

本设计不追求：

1. 第一阶段就完全删除 Anthropic 相关代码。
2. 第一阶段就替换所有远程能力实现。
3. 让所有社区版用户都必须使用第三方服务或自托管服务。
4. 在没有 Provider 抽象层之前就贸然重写全部业务逻辑。

换句话说，本设计主张的是：

> **先抽象，再替换；先 provider 化，再去 Anthropic 化。**

---

## 4. 需要解耦的能力面总览

从当前代码看，和 Anthropic 官方服务存在强耦合的，不是单一 OAuth 登录，而是至少 6 类能力面。

### 4.1 认证能力面（Auth Provider）

当前强耦合对象：

- `getClaudeAIOAuthTokens()`
- `isAnthropicAuthEnabled()`
- `getOAuthHeaders()`
- Claude.ai OAuth scopes
- claude.ai subscription 登录态

职责：

- 获取 access token / refresh token
- 刷新 token
- 生成认证头
- 判断当前身份来源与认证方式

### 4.2 远端执行能力面（Remote Execution Provider）

当前强耦合对象：

- CCR / remote environment
- `teleportToRemote()`
- remote review / ultraplan / remote isolation
- code session / remote session API

职责：

- 创建远端执行环境
- 派发 remote agent
- 管理远端 session 生命周期
- 拉取远端执行事件与结果

### 4.3 远程控制与桥接能力面（Bridge Provider）

当前强耦合对象：

- `bridge/*`
- `remote-control`
- `claude.ai/code`
- worker JWT / session ingress token / trusted device token

职责：

- 本地 CLI 与 Web/App 端会话桥接
- 远程控制本地环境
- 远端继续会话

### 4.4 连接器能力面（Connector Provider）

当前强耦合对象：

- hosted connectors
- `claude.ai/settings/connectors`
- scheduleRemoteAgents 里的 connector 检查
- hosted MCP/集成生态

职责：

- 提供托管外部服务连接
- 列出已连接服务
- 检查 task/agent 所需 connector 是否可用

### 4.5 语音能力面（Voice Provider）

当前强耦合对象：

- `voice_stream`
- 需要 Anthropic OAuth token 的 voice mode

职责：

- 建立双向语音流
- 提供语音交互会话与认证

### 4.6 权限与套餐能力面（Entitlement Provider）

当前强耦合对象：

- `isClaudeAISubscriber()`
- claude.ai subscription gating
- 某些 bridge / remote / voice 能力基于订阅判断

职责：

- 判断用户是否具备某项能力
- 抽象订阅/套餐/组织策略/管理员策略

---

## 5. Provider 化的总体原则

## 5.1 业务层不得直接依赖 Anthropic-specific API

以下模式应逐步被替换：

- 业务代码直接调用 `getClaudeAIOAuthTokens()`
- 业务代码直接拼 `claude.ai` URL
- 业务代码直接判断 `isClaudeAISubscriber()`
- 业务代码直接假设 remote session 一定来自 CCR

应改成：

- 业务代码依赖 `authProvider`
- 业务代码依赖 `remoteExecutionProvider`
- 业务代码依赖 `bridgeProvider`
- 业务代码依赖 `connectorProvider`
- 业务代码依赖 `voiceProvider`
- 业务代码依赖 `entitlementProvider`

## 5.2 官方服务只是默认实现之一

社区版必须允许三类实现并存：

1. **Anthropic Official Provider**
2. **Third-Party Hosted Provider**
3. **Community Self-Hosted Provider**

## 5.3 Provider 不改变控制面原则

即使引入第三方服务或自托管服务，也不改变既定的控制面约束：

- 权威服务器仍然是共享控制面的唯一真相源
- 客户端仍然是运行时 + projection cache
- 运行时结果仍然必须经过受控回写

换句话说：

> **Provider 只替换能力来源，不替换控制面原则。**

---

## 6. 三类 Provider Profile

## 6.1 Official Provider Profile（官方服务实现）

用于保留与 Anthropic 官方服务的兼容。

### 特征

- Auth：Claude.ai OAuth
- Remote Execution：CCR
- Bridge：claude.ai/code
- Connectors：Anthropic hosted connectors
- Voice：Anthropic voice stream
- Entitlement：claude.ai subscription / org policy / trusted device

### 价值

- 社区版不丢失现有 Anthropic 用户路径
- 便于逐步迁移，而不是一刀切断官方兼容

## 6.2 Third-Party Hosted Provider Profile（第三方托管实现）

用于支持外部 SaaS 或合作方服务。

### 可能形态

- Auth：第三方 OAuth / OIDC / SSO
- Remote Execution：第三方远端执行平台
- Bridge：第三方 Web 控制台 / Web IDE
- Connectors：第三方 connector hub
- Voice：第三方 realtime voice 服务
- Entitlement：第三方套餐/权限系统

### 价值

- 让社区版不再单点依赖 Anthropic 官方生态

## 6.3 Community Self-Hosted Profile（社区自托管实现）

用于支持完全自建部署。

### 可能形态

- Auth：本地 token / JWT / 自建 OAuth / OIDC（Keycloak / Authentik 等）
- Remote Execution：自建 worker pool / sandbox cluster / k8s jobs
- Bridge：自建 authority server + web console + websocket bridge
- Connectors：自建 MCP registry / connector broker
- Voice：自建或第三方 voice gateway
- Entitlement：管理员策略 / 配置文件 / 自建 IAM

### 价值

- 真正实现和 Anthropic 官方服务解耦
- 符合“社区版”长期目标

---

## 7. 推荐的 Provider 接口分层

建议新增：

```text
src/providers/
  auth/
  remoteExecution/
  bridge/
  connectors/
  voice/
  entitlements/
```

每类至少包含：

- `types.ts`：接口定义
- `official.ts`：官方实现
- `selfHosted.ts`：自托管实现占位
- `thirdParty.ts`：第三方实现占位
- `registry.ts`：根据配置解析当前启用的 provider

---

## 8. 各 Provider 建议接口

## 8.1 AuthProvider

```ts
interface AuthProvider {
  getAuthType(): 'oauth' | 'api_key' | 'jwt' | 'none'
  getAccessToken(): Promise<string | null>
  refreshAccessToken(): Promise<string | null>
  getAuthHeaders(scope?: string): Promise<Record<string, string>>
  getAccountIdentity(): Promise<{
    userId: string
    email?: string
    tenantId?: string
  } | null>
  isLoggedIn(): Promise<boolean>
}
```

## 8.2 RemoteExecutionProvider

```ts
interface RemoteExecutionProvider {
  checkEligibility(): Promise<{
    eligible: boolean
    reasons?: string[]
  }>
  createEnvironment(input: Record<string, unknown>): Promise<{
    environmentId: string
    metadata?: Record<string, unknown>
  }>
  launchTask(input: {
    taskType: string
    prompt: string
    repo?: string
    workspaceId?: string
  }): Promise<{
    remoteTaskId: string
    sessionUrl?: string
  }>
  pollTask(remoteTaskId: string): Promise<Record<string, unknown>>
  stopTask(remoteTaskId: string): Promise<void>
}
```

## 8.3 BridgeProvider

```ts
interface BridgeProvider {
  isAvailable(): Promise<boolean>
  createBridgeSession(input: {
    cwd: string
    name?: string
  }): Promise<{
    sessionId: string
    connectUrl?: string
    sessionUrl?: string
  }>
  connect(sessionId: string): Promise<void>
  disconnect(sessionId: string): Promise<void>
}
```

## 8.4 ConnectorProvider

```ts
interface ConnectorProvider {
  listConnectedServices(): Promise<Array<{
    id: string
    name: string
    category?: string
  }>>
  validateRequiredServices(serviceIds: string[]): Promise<{
    ok: boolean
    missing: string[]
  }>
  getSetupUrl(): string | null
}
```

## 8.5 VoiceProvider

```ts
interface VoiceProvider {
  isAvailable(): Promise<boolean>
  connectVoiceSession(): Promise<{
    sessionId: string
    streamUrl?: string
  }>
  disconnectVoiceSession(sessionId: string): Promise<void>
}
```

## 8.6 EntitlementProvider

```ts
interface EntitlementProvider {
  hasCapability(capability: string): Promise<boolean>
  getCapabilityReason?(capability: string): Promise<string | null>
}
```

---

## 9. 当前代码与 Provider 的直接映射

### 9.1 AuthProvider 首先要接管的当前调用点

优先替换：

- `getClaudeAIOAuthTokens()`
- `isAnthropicAuthEnabled()`
- `getOAuthHeaders()`
- `checkAndRefreshOAuthTokenIfNeeded()`

直接涉及文件包括：

- `src/utils/auth.ts`
- `src/bridge/createSession.ts`
- `src/bridge/bridgeConfig.ts`
- `src/bridge/initReplBridge.ts`
- `src/voice/voiceModeEnabled.ts`
- `src/skills/bundled/scheduleRemoteAgents.ts`

### 9.2 RemoteExecutionProvider 首先要接管的当前调用点

优先替换：

- `teleportToRemote()`
- `checkRemoteAgentEligibility()`
- remote review / ultraplan / remote agent launch

直接涉及文件包括：

- `src/commands/ultraplan.tsx`
- `src/tools/AgentTool/AgentTool.tsx`
- `src/commands/review/reviewRemote.ts`
- `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`
- `src/utils/background/remote/remoteSession.ts`

### 9.3 BridgeProvider 首先要接管的当前调用点

优先替换：

- remote-control / bridge session 创建
- claude.ai/code 连接与桥接控制

直接涉及文件包括：

- `src/bridge/bridgeMain.ts`
- `src/bridge/createSession.ts`
- `src/bridge/remoteBridgeCore.ts`
- `src/bridge/replBridge.ts`
- `src/bridge/initReplBridge.ts`

### 9.4 ConnectorProvider 首先要接管的当前调用点

优先替换：

- hosted connector 检查
- connector setup URL
- remote schedule 依赖的 connector 可用性判断

直接涉及文件包括：

- `src/skills/bundled/scheduleRemoteAgents.ts`
- `src/tools/MCPTool/*`
- `src/tools/McpAuthTool/McpAuthTool.ts`

### 9.5 VoiceProvider 首先要接管的当前调用点

优先替换：

- voice auth check
- voice stream availability check

直接涉及文件包括：

- `src/voice/voiceModeEnabled.ts`
- `src/commands/voice/voice.ts`

### 9.6 EntitlementProvider 首先要接管的当前调用点

优先替换：

- `isClaudeAISubscriber()`
- bridge/voice/remote 依赖的 subscription gating

直接涉及文件包括：

- `src/bridge/bridgeEnabled.ts`
- `src/bridge/envLessBridgeConfig.ts`
- `src/utils/auth.ts`

---

## 10. 优化方案：分阶段实施

## Phase A：建立 Provider 抽象层，不改业务语义

目标：先把接口提炼出来。

包含：

- `src/providers/*`
- provider registry
- 默认 official provider 实现
- 业务层禁止继续直接 import Anthropic-specific helper 的约束

## Phase B：先替换认证与 entitlement

原因：

- 这是几乎所有 Anthropic 绑定能力的共同入口
- 不先抽 auth/entitlement，后面 bridge/remote/voice 都会继续绑死

## Phase C：替换 RemoteExecutionProvider

优先覆盖：

- `ultraplan`
- remote isolation agent
- remote review
- scheduled remote agents

## Phase D：替换 BridgeProvider

优先覆盖：

- `remote-control`
- bridge session
- claude.ai/code 强耦合链接与控制流

## Phase E：替换 ConnectorProvider 与 VoiceProvider

优先覆盖：

- hosted connectors
- MCP auth integration
- voice mode

---

## 11. 社区版的推荐默认策略

如果目标是社区版，建议默认策略如下：

### 11.1 默认不再把 Anthropic 官方服务设为唯一上游

推荐做法：

- 配置中显式声明当前 provider profile
- 支持：
  - `official`
  - `third_party`
  - `self_hosted`

### 11.2 官方实现保留，但不写死

Anthropic provider 可以继续作为默认实现之一，但不能让业务层假设“只会是这个 provider”。

### 11.3 自托管路径优先保证可跑通

社区版最重要的是：

- 不依赖 claude.ai 账户也能用
- 不依赖官方 remote service 也能跑远端执行
- 不依赖官方 connector hub 也能接入第三方或自建服务

---

## 12. 与权威服务器架构的关系

这份 Provider 设计文档不是替代《权威服务器架构设计》，而是补充它：

- 《权威服务器架构设计》解决的是：**控制面归谁、状态如何收束**
- 本文档解决的是：**能力从谁那里获得、如何不再写死 Anthropic 官方服务**

两者结合后的完整理念是：

> **控制面收束到社区版自己的权威服务器；运行时能力则通过 Provider 抽象接入官方服务、第三方服务或社区自托管服务。**

---

## 13. 最终结论

如果社区版的目标是“与 Anthropic 官方服务解耦”，那真正需要抽象的不是单一 OAuth 登录，而是整条能力供应链：

- Auth
- Remote Execution
- Bridge
- Connectors
- Voice
- Entitlements

因此，最正确的优化路径不是“去掉 Anthropic 代码”，而是：

1. 建立 Provider 抽象层；
2. 把 Anthropic 实现降级为默认 provider 之一；
3. 允许第三方和自托管实现接入同一控制面；
4. 保持客户端双模能力与权威服务器控制面设计不变。

只有做到这一点，项目才会从“Anthropic 生态前端壳”真正进化成“可社区化演进的独立版本”。
