# 权威服务器 API 与事件协议

> 本文档面向下一阶段实现，定义“客户端（当前 `src/`）—权威服务器”之间的命令、事件、投影、幂等与重连协议。它建立在既定结论之上：客户端支持 `offline / online` 双模；在线模式下所有共享控制面对象都归属某个工作区（个人工作区或共享工作区）；运行时能力只能通过受控回写进入服务端。

---

## 1. 文档目标

本文档回答以下问题：

1. 在线模式下客户端如何注册、鉴权、绑定工作区。
2. 客户端如何向服务端发送治理命令、协作命令与状态回写。
3. 服务端如何以 projection/event 的形式回推 team、task、mailbox、governance、workspace 状态。
4. 如何处理幂等、版本、游标、重放、断线重连与兼容问题。

本文档不约束：

- 具体数据库表结构
- 具体框架实现
- 具体 UI 组件写法
- 本地工具执行细节

---

## 2. 协议设计原则

### 2.1 双模前提

- `offline`：不连接服务端，当前 `src/` 的本地控制面继续作为真相源。
- `online`：连接服务端，所有共享控制面对象都必须挂载 `workspaceId`。

因此，本文档中的 API 与事件默认只作用于 **online 模式**。

### 2.2 命令与投影分离

协议严格区分两类数据流：

1. **命令流**
   - 客户端 → 服务端
   - 用于提交治理动作、协作动作、状态回写动作

2. **投影流**
   - 服务端 → 客户端
   - 用于广播权威状态变化结果

客户端不得把“命令已发出”误当成“状态已经成功变更”；必须以服务端投影结果为准。

### 2.3 运行结果只允许受控回写

本地 agent / tool / tmux / REPL / remote runtime 的结果不能直接写服务端权威状态，只能通过：

- `task.progress.writeback`
- `task.result.writeback`
- `summary.writeback`
- `artifact.register`

等接口进入服务端。

### 2.4 工作区是在线模式的顶层命名空间

在线模式下：

- team / task / governance / mailbox / projection 都归属于 `workspaceId`
- `workspaceId` 是共享协作隔离边界
- 个人工作区与共享工作区之间的数据移动必须通过治理命令完成

---

## 3. 连接与会话模型

## 3.1 在线模式连接阶段

在线模式建议分为三段：

1. **注册阶段**：建立服务端 session
2. **绑定阶段**：选择 `activeWorkspace`
3. **订阅阶段**：接收 projection / mailbox / governance / task 事件流

时序如下：

```text
client -> session.register
server -> session.registered

client -> workspace.bind
server -> workspace.bound

client -> ws subscribe / projection resume
server -> projection.* / mailbox.* / governance.* / task.*
```

## 3.2 客户端基础标识

所有在线模式命令建议都携带以下标识：

- `tenantId`
- `userId`
- `clientId`
- `sessionId`
- `runtimeMode`
- `workspaceId`（若已绑定）
- `commandId`
- `idempotencyKey`

---

## 4. 通用 Envelope

## 4.1 命令 Envelope

```ts
type ClientCommandEnvelope = {
  commandId: string
  idempotencyKey: string
  tenantId: string
  userId: string
  clientId: string
  sessionId?: string
  runtimeMode: 'offline' | 'online'
  workspaceId?: string
  type: string
  payload: Record<string, unknown>
  sentAt: number
  trace?: {
    parentCommandId?: string
    parentTaskId?: string
    requestId?: string
  }
}
```

### 字段说明

- `commandId`：单次命令唯一 ID。
- `idempotencyKey`：幂等键；客户端重试时必须保持一致。
- `workspaceId`：在线模式下若该命令影响共享控制面对象，则必须携带。
- `trace`：用于串联 spawn / writeback / governance / publish 的链路。

## 4.2 命令应答 Envelope

```ts
type CommandAckEnvelope = {
  commandId: string
  accepted: boolean
  status: 'accepted' | 'rejected' | 'rewritten'
  reason?: string
  rewrittenPayload?: Record<string, unknown>
  generatedEventIds?: string[]
  generatedAggregateIds?: string[]
  serverTime: number
}
```

### 语义说明

- `accepted=true` 仅表示服务端接受并开始处理命令。
- 客户端最终状态更新仍然应等待 projection/event。

## 4.3 投影事件 Envelope

```ts
type ProjectionEventEnvelope = {
  eventId: string
  tenantId: string
  workspaceId?: string
  stream: 'workspace' | 'team' | 'task' | 'mailbox' | 'governance' | 'session'
  projectionVersion: number
  cursor: string
  type: string
  payload: Record<string, unknown>
  createdAt: number
}
```

### 核心语义

- `projectionVersion`：该 stream 的当前版本号。
- `cursor`：断线重连时使用的游标。

---

## 5. Session 与模式协议

## 5.1 `session.register`

### 用途

- 注册在线会话
- 返回个人工作区与可访问共享工作区
- 初始化 projection resume 信息

### 请求

```ts
type SessionRegisterPayload = {
  protocolVersion: string
  clientVersion?: string
  deviceId?: string
  requestedMode: 'online'
  capabilities?: {
    hasLocalBash?: boolean
    hasTmux?: boolean
    hasRepl?: boolean
    hasWorktree?: boolean
    hasRemoteRuntime?: boolean
  }
}
```

### 响应

```ts
type SessionRegisteredPayload = {
  sessionId: string
  personalWorkspaceId: string
  availableSharedWorkspaces: Array<{
    workspaceId: string
    name: string
    role: 'owner' | 'admin' | 'member' | 'viewer'
  }>
  recommendedActiveWorkspaceId?: string
  projectionResume: {
    cursor?: string
    version?: number
  }
}
```

## 5.2 `session.heartbeat`

### 用途

- 维持在线状态
- 更新 presence
- 携带最小运行态摘要

### 请求

```ts
type SessionHeartbeatPayload = {
  activeWorkspaceId?: string
  activeTaskIds?: string[]
  connectionHealth?: 'healthy' | 'degraded'
  localRuntimeSummary?: {
    hasPendingMailbox?: boolean
    runningTaskCount?: number
  }
}
```

### 响应

```ts
type SessionHeartbeatAck = {
  ok: true
  serverTime: number
  projectionStale?: boolean
}
```

## 5.3 `mode.switch`

### 用途

- 客户端显式切换 `offline / online`

### 约束

- `offline -> online` 时，必须先重新走 `session.register`。
- `online -> offline` 时，客户端应停止接收 projection 流，并将共享控制面状态从“权威真相”降为只读历史或本地快照。

---

## 6. 工作区协议

## 6.1 `workspace.list`

列出当前用户可访问的：

- 个人工作区
- 共享工作区

## 6.2 `workspace.bind`

### 用途

- 将当前 session 绑定到某个 `activeWorkspaceId`

### 请求

```ts
type WorkspaceBindPayload = {
  workspaceId: string
  resumeCursor?: string
}
```

### 响应

```ts
type WorkspaceBoundPayload = {
  workspaceId: string
  workspaceType: 'personal' | 'shared'
  name: string
  projectionVersion: number
  initialProjection: {
    teams?: unknown[]
    tasks?: unknown[]
    mailboxSummary?: unknown
    governanceSummary?: unknown
  }
}
```

## 6.3 `workspace.create_shared`

### 用途

- 创建新的共享工作区

### 请求

```ts
type CreateSharedWorkspacePayload = {
  name: string
  description?: string
  initialMemberUserIds?: string[]
}
```

## 6.4 `workspace.join`

### 用途

- 接受邀请或主动加入共享工作区

## 6.5 `workspace.publish_from_personal`

### 用途

- 将个人工作区中的任务、摘要、计划或产物发布到共享工作区

### 请求

```ts
type PublishFromPersonalPayload = {
  sourceWorkspaceId: string
  targetWorkspaceId: string
  objectType: 'task' | 'summary' | 'artifact' | 'evidence' | 'plan'
  objectId: string
  publishMode: 'copy' | 'reference'
  note?: string
}
```

### 约束

- 必须经过治理裁决。
- 不允许客户端直接把目标共享工作区对象写成最终真相。

---

## 7. Team 与任务治理命令

## 7.1 `team.create`

### 用途

- 在当前活动工作区中创建 team

### 请求

```ts
type TeamCreatePayload = {
  teamName: string
  description?: string
  leadAgentType?: string
}
```

## 7.2 `team.delete`

### 用途

- 解散 team

### 服务端要求

- 必须校验该 team 是否仍有活跃成员或任务。

## 7.3 `agent.spawn`

### 用途

- 在当前工作区中创建 task / agent 生命周期实体

### 请求

```ts
type AgentSpawnPayload = {
  teamId?: string
  parentTaskId?: string
  description: string
  promptSummary: string
  agentType?: string
  executionMode: 'sync' | 'background' | 'teammate' | 'remote'
  backendHint?: 'in-process' | 'tmux' | 'remote' | 'standalone'
  permissionMode?: 'default' | 'plan' | 'acceptEdits' | 'auto'
}
```

### 语义

- 服务端创建 `TaskAggregate`
- 然后把任务分发给目标客户端执行
- 客户端只负责执行与回写，不负责写最终任务真相

## 7.4 `task.stop`

### 用途

- 请求停止某个任务

### 请求

```ts
type TaskStopPayload = {
  taskId: string
  reason?: string
}
```

---

## 8. Mailbox 与协作消息协议

## 8.1 `message.send`

### 用途

- 替代本地 `SendMessageTool` + `teammateMailbox`

### 请求

```ts
type MessageSendPayload = {
  toAgentId: string
  teamId?: string
  kind:
    | 'direct_message'
    | 'task_notification'
    | 'permission_request'
    | 'permission_response'
    | 'plan_request'
    | 'plan_response'
    | 'shutdown_request'
    | 'shutdown_response'
  text?: string
  payload?: Record<string, unknown>
}
```

### 服务端职责

- 校验发送者与目标的工作区归属
- 写入 mailbox 权威层
- 通过 projection 流或实时消息推送给目标客户端

## 8.2 `mailbox.ack`

### 用途

- 客户端确认消息已送达/已读/已消费

### 请求

```ts
type MailboxAckPayload = {
  messageId: string
  status: 'delivered' | 'acknowledged' | 'consumed'
}
```

---

## 9. 状态回写协议

## 9.1 `task.progress.writeback`

### 用途

- 上报任务进度摘要

### 请求

```ts
type TaskProgressWritebackPayload = {
  taskId: string
  summary: {
    activity?: string
    toolUses?: number
    tokenCount?: number
    percent?: number
  }
  localTimestamp: number
}
```

## 9.2 `task.result.writeback`

### 用途

- 上报任务完成/失败/终止结果摘要

### 请求

```ts
type TaskResultWritebackPayload = {
  taskId: string
  finalStatus: 'completed' | 'failed' | 'killed'
  resultSummary?: string
  outputRef?: string
  evidenceRefs?: string[]
  usage?: {
    totalTokens?: number
    totalToolUses?: number
    durationMs?: number
  }
}
```

## 9.3 `summary.writeback`

### 用途

- 回写 transcript / memory / diagnostic 的摘要

### 请求

```ts
type SummaryWritebackPayload = {
  targetType: 'task' | 'session' | 'workspace'
  targetId: string
  summaryType: 'transcript' | 'memory' | 'diagnostic' | 'plan'
  text: string
  evidenceRefs?: string[]
}
```

## 9.4 `artifact.register`

### 用途

- 注册产物引用，而不是默认上传原始大对象

### 请求

```ts
type ArtifactRegisterPayload = {
  taskId?: string
  workspaceId: string
  artifactType: 'patch' | 'log' | 'output' | 'diagnostic' | 'evidence'
  ref: string
  summary?: string
}
```

---

## 10. Governance 协议

## 10.1 `governance.request`

### 用途

- 发起 plan approval / permission / sandbox / shutdown / mode change 请求

### 请求

```ts
type GovernanceRequestPayload = {
  kind:
    | 'permission'
    | 'sandbox_permission'
    | 'plan_approval'
    | 'mode_change'
    | 'shutdown_approval'
  relatedTaskId?: string
  relatedTeamId?: string
  details: Record<string, unknown>
}
```

## 10.2 `governance.respond`

### 用途

- 响应治理请求

### 请求

```ts
type GovernanceRespondPayload = {
  governanceId: string
  decision: 'allow' | 'deny' | 'rewrite'
  reason?: string
  rewrittenDetails?: Record<string, unknown>
}
```

---

## 11. 服务端投影事件

## 11.1 Workspace 事件

- `projection.workspace.bound`
- `projection.workspace.patch`
- `projection.workspace.member_joined`
- `projection.workspace.member_left`

## 11.2 Team 事件

- `projection.team.created`
- `projection.team.updated`
- `projection.team.deleted`
- `projection.team.member_added`
- `projection.team.member_removed`

## 11.3 Task 事件

- `projection.task.created`
- `projection.task.updated`
- `projection.task.progress`
- `projection.task.notification`
- `projection.task.completed`
- `projection.task.failed`
- `projection.task.killed`

## 11.4 Mailbox 事件

- `projection.mailbox.deliver`
- `projection.mailbox.updated`

## 11.5 Governance 事件

- `projection.governance.request`
- `projection.governance.resolved`

---

## 12. 游标、版本与重放

## 12.1 Projection 游标

客户端应持久保存每个 stream 的最近 `cursor`：

- `workspace`
- `team`
- `task`
- `mailbox`
- `governance`

## 12.2 断线重连

客户端重连后：

1. 带上最近 `cursor`
2. 服务端尝试按 cursor 增量补发
3. 若 cursor 已失效，则返回 `projection.resync.required`

## 12.3 `projection.resync`

### 用途

- 客户端请求全量重建投影

### 请求

```ts
type ProjectionResyncPayload = {
  workspaceId: string
  requestedStreams?: Array<'workspace' | 'team' | 'task' | 'mailbox' | 'governance'>
}
```

---

## 13. 幂等与一致性规则

## 13.1 必须支持幂等的命令

- `team.create`
- `workspace.create_shared`
- `workspace.join`
- `workspace.publish_from_personal`
- `agent.spawn`
- `task.stop`
- `message.send`
- `governance.respond`
- `task.result.writeback`

## 13.2 客户端一致性要求

客户端必须遵循：

1. 在线模式下，本地 `AppState.tasks / teamContext / inbox` 只是 projection cache。
2. 不允许本地先修改为最终状态，再等待服务端纠偏。
3. 允许本地做 optimistic UI，但必须标记为 `pending`，并以 projection 最终收束。

---

## 14. 错误模型

统一建议错误返回结构：

```ts
type CommandError = {
  commandId: string
  errorCode:
    | 'UNAUTHENTICATED'
    | 'FORBIDDEN'
    | 'WORKSPACE_NOT_BOUND'
    | 'INVALID_WORKSPACE'
    | 'INVALID_TEAM'
    | 'INVALID_TASK'
    | 'INVALID_GOVERNANCE_REQUEST'
    | 'IDEMPOTENCY_CONFLICT'
    | 'PROJECTION_STALE'
    | 'VALIDATION_FAILED'
    | 'INTERNAL_ERROR'
  message: string
  retriable: boolean
}
```

---

## 15. 推荐最小实现范围（MVP）

第一阶段建议只实现以下协议：

1. `session.register`
2. `session.heartbeat`
3. `workspace.list`
4. `workspace.bind`
5. `team.create`
6. `agent.spawn`
7. `message.send`
8. `mailbox.ack`
9. `task.progress.writeback`
10. `task.result.writeback`
11. `governance.request`
12. `governance.respond`
13. `projection.resync`

这样就足够先替换：

- `TeamFile`
- `teammateMailbox`
- `SendMessage`

以及最关键的任务注册与回流链路。

---

## 16. 与当前代码的直接对接点

这份协议最先会影响以下客户端模块：

- `teammateMailbox.ts`
- `SendMessageTool.ts`
- `useInboxPoller.ts`
- `TeamCreateTool.ts`
- `TeamDeleteTool.ts`
- `LocalAgentTask`
- `RemoteAgentTask`
- `InProcessTeammateTask`
- `AppState.tasks`
- `AppState.teamContext`
- `AppState.inbox`

因此它应被视为：

> **后续客户端适配层重构与服务端第一阶段实现的共同契约。**
