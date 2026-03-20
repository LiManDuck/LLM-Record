# Session 模块交互设计分析

> 基于关注点分离原则，Session 系统拆分为多个职责单一的模块文件。

---

## 一、文件职责总览

| 文件 | 职责 |
|------|------|
| `index.ts` | Session 核心 CRUD、事件管理 |
| `schema.ts` | ID 类型定义 |
| `message-v2.ts` | Message/Part 数据结构和转换逻辑 |
| `session.sql.ts` | 数据库表结构定义 |
| `processor.ts` | 处理 LLM 流式响应（text、reasoning、tool 等事件） |
| `prompt.ts` | 用户提示处理和主循环逻辑 |
| `llm.ts` | LLM API 调用、工具解析 |
| `compaction.ts` | 会话压缩（减少 token 使用） |
| `summary.ts` | 代码差异统计和摘要 |
| `retry.ts` | API 重试逻辑（指数退避） |
| `status.ts` | 会话状态管理 |
| `revert.ts` | 回滚到之前状态 |
| `instruction.ts` | 指令处理（计划模式等） |
| `system.ts` | 系统提示生成 |
| `todo.ts` | 任务列表功能 |
| `prompt/` | 各种系统提示模板 |

---

## 二、核心类结构与属性

### 1. Session（index.ts）— 核心数据管理

```typescript
type Session.Info = {
  id: SessionID
  slug: string
  projectID: ProjectID
  workspaceID?: WorkspaceID
  directory: string
  parentID?: SessionID
  title: string
  version: string
  summary?: {
    additions: number
    deletions: number
    files: number
    diffs?: FileDiff[]
  }
  share?: { url: string }
  revert?: {
    messageID: MessageID
    partID?: PartID
    snapshot?: string
    diff?: string
  }
  permission?: Ruleset
  time: {
    created: number
    updated: number
    compacting?: number
    archived?: number
  }
}
```

主要方法：

```typescript
Session.get(id): Promise<Session.Info>
Session.create(input): Promise<Session.Info>
Session.messages(input): Promise<MessageV2.WithParts[]>
Session.updateMessage(msg): Promise<MessageV2.Info>
Session.updatePart(part): Promise<MessageV2.Part>
```

---

### 2. MessageV2（message-v2.ts）— 消息结构定义

```typescript
type User = {
  id: MessageID
  sessionID: SessionID
  role: "user"
  time: { created: number }
  agent: string
  model: { providerID: ProviderID; modelID: ModelID }
  format?: OutputFormat
  summary?: { title?: string; body?: string; diffs: FileDiff[] }
  system?: string
  tools?: Record<string, boolean>
  variant?: string
}

type Assistant = {
  id: MessageID
  sessionID: SessionID
  role: "assistant"
  parentID: MessageID
  modelID: ModelID
  providerID: ProviderID
  mode: string
  agent: string
  path: { cwd: string; root: string }
  time: { created: number; completed?: number }
  summary?: boolean
  cost: number
  tokens: {
    total?: number
    input: number
    output: number
    reasoning: number
    cache: { read: number; write: number }
  }
  structured?: any
  variant?: string
  finish?: string
  error?: ErrorObject
}

// Part 联合类型
type Part =
  | TextPart
  | FilePart
  | ToolPart
  | ReasoningPart
  | StepStartPart
  | StepFinishPart
  | CompactionPart
  | SubtaskPart
  | RetryPart
  | PatchPart
  | SnapshotPart
```

---

### 3. SessionPrompt（prompt.ts）— 对话流程控制

```typescript
type SessionState = {
  abort: AbortController
  callbacks: Array<{
    resolve: (msg: MessageV2.WithParts) => void
    reject: (reason?: any) => void
  }>
}

SessionPrompt.prompt(input: PromptInput): Promise<MessageV2.WithParts>
SessionPrompt.loop(input: LoopInput): Promise<MessageV2.WithParts>
SessionPrompt.cancel(sessionID): void
SessionPrompt.shell(input: ShellInput): Promise<MessageV2.WithParts>
```

---

### 4. SessionProcessor（processor.ts）— LLM 响应处理

```typescript
type SessionProcessor.Info = {
  message: MessageV2.Assistant
  partFromToolCall(toolCallID: string): MessageV2.ToolPart | undefined
  process(streamInput: LLM.StreamInput): Promise<"continue" | "stop" | "compact">
}
```

---

### 5. LLM（llm.ts）— LLM API 调用

```typescript
type LLM.StreamInput = {
  user: MessageV2.User
  sessionID: string
  model: Provider.Model
  agent: Agent.Info
  permission?: PermissionNext.Ruleset
  system: string[]
  abort: AbortSignal
  messages: ModelMessage[]
  small?: boolean
  tools: Record<string, Tool>
  retries?: number
  toolChoice?: "auto" | "required" | "none"
}

type LLM.StreamOutput = StreamTextResult<ToolSet, unknown>
```

---

### 6. SessionStatus（status.ts）— 会话状态管理

```typescript
type StatusInfo =
  | { type: "idle" }
  | { type: "busy" }
  | { type: "retry"; attempt: number; message: string; next: number }

SessionStatus.get(sessionID): StatusInfo
SessionStatus.set(sessionID, status): void
```

---

### 7. SessionCompaction（compaction.ts）— 会话压缩

```typescript
SessionCompaction.isOverflow(input): Promise<boolean>
SessionCompaction.process(input): Promise<"continue" | "stop">
SessionCompaction.prune(input): Promise<void>
```

---

### 8. SessionRetry（retry.ts）— 重试逻辑

```typescript
SessionRetry.retryable(error): string | undefined
SessionRetry.delay(attempt, error): number
SessionRetry.sleep(ms, signal): Promise<void>
```

---

### 9. SessionRevert（revert.ts）— 回滚功能

```typescript
SessionRevert.revert(input): Promise<Session.Info>
SessionRevert.unrevert(input): Promise<Session.Info>
SessionRevert.cleanup(session): Promise<void>
```

---

### 10. SessionSummary（summary.ts）— 摘要计算

```typescript
SessionSummary.summarize(input): Promise<void>
SessionSummary.diff(input): Promise<FileDiff[]>
SessionSummary.computeDiff(input): Promise<FileDiff[]>
```

---

### 11. InstructionPrompt（instruction.ts）— 指令管理

```typescript
InstructionPrompt.systemPaths(): Promise<string[]>
InstructionPrompt.system(): Promise<string[]>
InstructionPrompt.resolve(messages, filepath, messageID): Promise<...>
InstructionPrompt.clear(messageID): void
```

---

### 12. SystemPrompt（system.ts）— 系统提示生成

```typescript
SystemPrompt.instructions(): string
SystemPrompt.provider(model): string[]
SystemPrompt.environment(model): Promise<string[]>
SystemPrompt.skills(agent): Promise<string>
```

---

### 13. Todo（todo.ts）— 任务列表

```typescript
type Todo.Info = {
  content: string
  status: "pending" | "in_progress" | "completed" | "cancelled"
  priority: "high" | "medium" | "low"
}

Todo.update(input): void
Todo.get(sessionID): Todo.Info[]
```

---

## 三、核心调用流程

### 主流程：用户发送消息 → AI 响应

```
用户输入
  └─> SessionPrompt.prompt(PromptInput)
        ├─> createUserMessage(input)
        │     └─> Session.updateMessage(info)
        │     └─> Session.updatePart(part)   // 多个 parts
        │
        ├─> Session.touch(sessionID)
        │
        └─> SessionPrompt.loop(LoopInput)
              ├─> start(sessionID)            // 创建 AbortController
              │
              ├─> MessageV2.stream(sessionID) // 读取历史消息
              │
              ├─> 获取 lastUser, lastAssistant, lastFinished
              │
              ├─> Provider.getModel()         // 获取模型配置
              │
              ├─> Agent.get(agentName)        // 获取代理配置
              │
              ├─> SessionProcessor.create({
              │       assistantMessage: Session.updateMessage(...)
              │       sessionID, model, abort
              │     })
              │
              ├─> resolveTools({
              │       agent, session, model, tools, processor, messages
              │     })                        // 解析可用工具
              │     └─> ToolRegistry.tools() + MCP.tools()
              │
              ├─> SystemPrompt.environment(model)
              ├─> SystemPrompt.skills(agent)
              ├─> InstructionPrompt.system()
              │
              └─> processor.process({
                     user, agent, permission, abort,
                     sessionID, system, messages, tools, model
                   })
                    │
                    └─> LLM.stream(StreamInput)    // 调用 LLM API
                          │
                          ├─> SystemPrompt.provider() / SystemPrompt.instructions()
                          ├─> Plugin.trigger("chat.headers")
                          ├─> Plugin.trigger("chat.params")
                          ├─> Agent / Config / ProviderTransform 转换
                          │
                          └─> streamText()           // 返回流式输出
                                │
                                ├─> start:
                                │     └─> SessionStatus.set("busy")
                                │
                                ├─> text-start / text-delta / text-end:
                                │     └─> Session.updatePartDelta()
                                │
                                ├─> reasoning-start / reasoning-delta / reasoning-end:
                                │     └─> Session.updatePart()
                                │
                                ├─> tool-input-start / tool-input-delta / tool-input-end
                                │
                                ├─> tool-call:
                                │     └─> Session.updatePart(tool, running)
                                │     └─> doom loop 检测
                                │
                                ├─> tool-result:
                                │     └─> Session.updatePart(tool, completed)
                                │
                                ├─> tool-error:
                                │     └─> Session.updatePart(tool, error)
                                │
                                ├─> start-step / finish-step:
                                │     └─> Snapshot.track() / Snapshot.patch()
                                │
                                ├─> error:
                                │     └─> SessionRetry.retryable(error) / 手动错误
                                │
                                └─> finish: 更新 finish reason, cost, tokens

              └─> 根据 processor.process() 结果：
                    ├─> "continue"  → 继续下一轮循环
                    ├─> "stop"      → 结束，返回结果
                    └─> "compact"   → SessionCompaction.create()
                                          └─> 触发 compaction 任务
                                                └─> 继续循环
```

---

### 会话压缩流程

```
SessionCompaction.create()
  └─> Session.updateMessage(compaction user message)
  └─> Session.updatePart(compaction part)
  └─> SessionPrompt.loop() 中检测到 compaction part
      └─> SessionCompaction.process()
            ├─> 获取历史消息
            ├─> Agent.get("compaction")    // 专用压缩代理
            ├─> Session.updateMessage(assistant message)
            ├─> SessionProcessor.create()
            ├─> processor.process()        // 让 LLM 生成摘要
            │     └─> 插入 "What did we do so far?" 用户消息
            └─> 返回 "continue"            // 继续执行
```

---

### 回滚流程

```
SessionRevert.revert(input)
  ├─> SessionPrompt.assertNotBusy(sessionID)   // 检查会话状态
  ├─> Session.messages({ sessionID })           // 读取所有消息
  ├─> 找到回滚点（messageID 或 partID）
  ├─> 收集需要回滚的 PatchPart
  ├─> Snapshot.track()                          // 保存当前状态
  ├─> Snapshot.revert(patches)                  // 回滚文件系统
  ├─> SessionSummary.computeDiff()              // 计算差异
  ├─> Storage.write()                           // 存储差异
  ├─> Bus.publish(Event.Diff)                   // 发布差异事件
  └─> Session.setRevert()                       // 设置回滚状态

SessionRevert.unrevert(input)
  ├─> SessionPrompt.assertNotBusy(sessionID)
  ├─> Session.get(sessionID)
  ├─> Snapshot.restore(session.revert.snapshot) // 恢复文件
  └─> Session.clearRevert()                     // 清除回滚状态

SessionRevert.cleanup(session)
  └─> 清理回滚点之后的消息和部分
       ├─> 删除 MessageTable 记录（级联删除 PartTable）
       └─> Bus.publish(MessageV2.Event.Removed)
```

---

### 系统提示构建流程

```
LLM.stream() 中构建 system
  └─> [...inputs]
      ├─> agent.prompt || SystemPrompt.provider(model) || []
      ├─> input.system        // 用户自定义系统提示
      └─> input.user.system   // 用户消息中的系统提示

SystemPrompt.provider(model)
  └─> 根据 model.api.id 返回对应的 prompt 文件
      ├─> gpt-5      → PROMPT_CODEX
      ├─> gpt-/o1/o3 → PROMPT_BEAST
      ├─> gemini-    → PROMPT_GEMINI
      ├─> claude     → PROMPT_ANTHROPIC
      ├─> trinity    → PROMPT_TRINITY
      └─> default    → PROMPT_DEFAULT

SystemPrompt.environment(model)
  └─> 返回环境信息（工作目录、Git 状态、目录树）

SystemPrompt.skills(agent)
  ├─> PermissionNext.disabled(["skill"], agent.permission)
  ├─> Skill.available(agent)
  └─> 返回技能列表和描述

InstructionPrompt.system()
  ├─> InstructionPrompt.systemPaths()  // 查找配置文件
  │     ├─> Filesystem.findUp(["AGENTS.md", "CLAUDE.md"])
  │     ├─> Global.Path.config
  │     └─> config.instructions
  ├─> Filesystem.readText()            // 读取文件内容
  └─> fetch()                          // 读取远程 URL
```

---

## 四、事件发布订阅

### Session 事件

| 事件 | 说明 |
|------|------|
| `Session.Event.Created(sessionID, info)` | 会话创建 |
| `Session.Event.Updated(sessionID, info)` | 会话更新 |
| `Session.Event.Deleted(sessionID, info)` | 会话删除 |
| `Session.Event.Diff(sessionID, diff)` | 差异变化 |
| `Session.Event.Error(sessionID, error)` | 会话错误 |

### MessageV2 事件

| 事件 | 说明 |
|------|------|
| `MessageV2.Event.Updated(info)` | 消息更新 |
| `MessageV2.Event.Removed(sessionID, messageID)` | 消息删除 |
| `MessageV2.Event.PartUpdated(part)` | Part 更新 |
| `MessageV2.Event.PartDelta(sessionID, messageID, partID, field, delta)` | Part 增量更新 |
| `MessageV2.Event.PartRemoved(sessionID, messageID, partID)` | Part 删除 |

### 其他模块事件

| 事件 | 说明 |
|------|------|
| `SessionStatus.Event.Status(sessionID, status)` | 状态变更 |
| `SessionStatus.Event.Idle(sessionID)` | 空闲（已废弃） |
| `SessionCompaction.Event.Compacted(sessionID)` | 压缩完成 |
| `Todo.Event.Updated(sessionID, todos)` | 任务列表更新 |

> 所有事件通过 `Bus.publish(Event, payload)` 发布，其他模块可以订阅。

---

## 五、模块间依赖关系

```
                     ┌──────────────┐
                     │    Config    │
                     └──────────────┘
                            ↑
┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌─────────────┐
│ SystemPrompt │  │   Session    │  │ Instruction │  │    Agent    │
└─────────────┘  └──────────────┘  └─────────────┘  └─────────────┘
                        ↑                  ↑
┌─────────────┐  ┌──────────────┐  ┌─────────────┐
│     LLM     │  │SessionPrompt │  │  MessageV2  │
└─────────────┘  └──────────────┘  └─────────────┘
       ↑                ↑                  ↑
┌─────────────┐  ┌──────────────┐  ┌─────────────┐
│  Provider   │  │  Processor   │  │  Snapshot   │
└─────────────┘  └──────────────┘  └─────────────┘
                        ↑
┌─────────────┐  ┌──────────────┐  ┌──────────────┐
│ Plugin/MCP  │  │SessionRetry  │  │SessionRevert │
└─────────────┘  └──────────────┘  └──────────────┘
                                          ↓
                               ┌──────────────────┐
                               │  SessionStatus   │
                               └──────────────────┘
                                          ↓
                               ┌──────────────────┐
                               │SessionCompaction │
                               └──────────────────┘
                                          ↓
                               ┌──────────────────┐
                               │ SessionSummary   │
                               └──────────────────┘
                                          ↓
                    ┌─────────┐  ┌────────┐  ┌──────────┐
                    │   Bus   │  │Storage │  │ Database │
                    └─────────┘  └────────┘  └──────────┘
```

---

## 六、设计特点总结

| 特点 | 说明 |
|------|------|
| **分层清晰** | Session → Message → Part 三层结构，每层职责明确 |
| **事件驱动** | 通过 Bus 事件系统实现松耦合的模块通信 |
| **流程控制** | `SessionPrompt.loop` 是主控流程，协调各模块执行 |
| **流式处理** | Processor 处理 LLM 的增量输出，实时更新数据库 |
| **状态管理** | SessionStatus 管理会话生命周期状态 |
| **错误恢复** | SessionRetry 提供指数退避重试机制 |
| **资源管理** | SessionCompaction 和 SessionRevert 提供压缩和回滚能力 |



# Control Plane 模块交互设计分析

> 基于 `packages/opencode/src/control-plane` 目录分析，采用适配器模式实现不同类型工作空间的统一处理，结合 SSE + 全局事件总线实现实时事件同步。

---

## 一、文件结构总览

| 文件 | 核心功能 | 主要类/模块 |
|------|----------|-------------|
| `types.ts` | 核心类型定义 | `WorkspaceInfo`, `Adaptor` |
| `workspace.ts` | 工作空间 CRUD 操作 | `Workspace` 命名空间 |
| `workspace-context.ts` | 请求上下文管理 | `WorkspaceContext` |
| `workspace-server.server.ts` | HTTP 服务器 | `WorkspaceServer` |
| `workspace-server.routes.ts` | SSE 事件流路由 | `WorkspaceServerRoutes()` |
| `schema.ts` | WorkspaceID 类型系统 | `WorkspaceID` |
| `workspace.sql.ts` | 数据库表定义 | `WorkspaceTable` |
| `sse.ts` | SSE 解析工具 | `parseSSE` |
| `adaptors/index.ts` | 适配器注册中心 | `getAdaptor`, `installAdaptor` |
| `adaptors/worktree.ts` | Worktree 适配器 | `WorktreeAdaptor` |
| `workspace-router-middleware.ts` | 请求转发中间件 | `WorkspaceRouterMiddleware` |

---

## 二、核心类详细分析

### 1. WorkspaceInfo（types.ts:5-13）

```typescript
{
  id: WorkspaceID        // 唯一标识符
  type: string           // 工作空间类型（如 "worktree"）
  branch: string | null  // Git 分支名
  name: string | null    // 工作空间名称
  directory: string | null // 本地目录路径
  extra: unknown | null  // 额外配置信息
  projectID: ProjectID   // 所属项目 ID
}
```

---

### 2. Adaptor 接口（types.ts:16-21）

```typescript
{
  configure(input: WorkspaceInfo): WorkspaceInfo | Promise<WorkspaceInfo>
  create(input: WorkspaceInfo, from?: WorkspaceInfo): Promise<void>
  remove(config: WorkspaceInfo): Promise<void>
  fetch(config: WorkspaceInfo, input: RequestInfo | URL, init?: RequestInit): Promise<Response>
}
```

---

### 3. Workspace 命名空间（workspace.ts:15-153）

| 方法 | 说明 |
|------|------|
| `create(input)` | 创建新工作空间 |
| `list(project)` | 列出项目下所有工作空间 |
| `get(id)` | 获取指定工作空间 |
| `remove(id)` | 删除工作空间 |
| `startSyncing(project)` | 启动 SSE 同步循环 |

---

## 三、交互设计流程图

```
┌─────────────────────────────────────────────────────┐
│                   HTTP 请求入口                      │
│               WorkspaceServer.App()                  │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│               中间件层（Middleware Chain）            │
│  1. 提取 workspaceID 和 directory 参数               │
│  2. WorkspaceContext.provide() - 设置工作空间上下文  │
│  3. Instance.provide() - 设置实例上下文              │
└──────────────────────────┬──────────────────────────┘
                           │
               ┌───────────┴───────────┐
               │                       │
               ▼                       ▼
┌──────────────────────┐   ┌──────────────────────────┐
│   /event (SSE)       │   │ WorkspaceRouterMiddleware │
│   路由处理           │   │ 请求转发/本地处理         │
└──────────┬───────────┘   └───────────┬──────────────┘
           │                           │
           ▼                           ▼
┌──────────────────────┐   ┌──────────────────────────┐
│ GlobalBus.on("event")│   │ getAdaptor()             │
│ 接收事件             │◄──┤ fetch generic request    │
└──────────┬───────────┘   │ 返回 Response            │
           │               └──────────────────────────┘
           │                           │
           ▼                           ▼
┌─────────────────────────────────────────────────────┐
│                  Workspace 核心操作                  │
│  • create() → adaptor.configure() →                  │
│               adaptor.create() → DB.insert           │
│  • list()   → DB.select()                            │
│  • get()    → DB.select() by ID                      │
│  • remove() → adaptor.remove() → DB.delete()         │
└─────────────────────────────────────────────────────┘
```

---

## 四、关键调用链路

### 创建工作空间流程

```
Workspace.create(input)
  ↓
getAdaptor(type) → 返回 WorktreeAdaptor
  ↓
adaptor.configure({ ...input }) → 填充 name, branch, directory
  ↓
Database.insert(WorkspaceTable)
  ↓
adaptor.create(config) → Worktree.createFromInfo()
  ↓
返回 WorkspaceInfo
```

---

### SSE 事件同步循环

```
Workspace.startSyncing(project)
  ↓
list(project) → 过滤非 worktree 类型
  ↓
forEach workspace:
  ↓
  workspaceEventLoop(space, abortSignal)
    ↓
  adaptor.fetch(space, "/event", { signal }) → 建立 SSE 连接
    ↓
  parseSSE(res.body, signal, callback)
    ↓
  GlobalBus.emit("event", { directory, payload })
    ↓
  等待 250ms 后重试
```

---

### 请求代理流程（WorkspaceRouterMiddleware）

```
HTTP 请求
  ↓
WorkspaceContext.workspaceID 获取当前工作空间 ID
  ↓
Workspace.get(workspaceID) → 查询数据库
  ↓
getAdaptor(workspace.type)
  ↓
adaptor.fetch(workspace, url, init) → 转发到目标工作空间
  ↓
返回 Response
```

---

## 五、数据流向

```
┌─────────────────────────────────────────────────────────────┐
│                      数据持久化层                            │
│                  WorkspaceTable (SQLite)                     │
│   ┌──────┬────────┬────────┬──────────┬───────────┐         │
│   │  id  │  type  │ branch │   name   │ directory │  ...    │
│   │  pk  │  text  │  text  │   text   │   text    │         │
│   └──────┴────────┴────────┴──────────┴───────────┘         │
└───────────────────────────┬─────────────────────────────────┘
                            ▲
                            │  Database.use()
                            │
┌───────────────────────────┴─────────────────────────────────┐
│                      Workspace 业务层                        │
│   create(), list(), get(), remove(), startSyncing()          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Adaptors 抽象层                         │
│   ┌──────────────────┐   ┌──────────────────────────────┐   │
│   │ WorktreeAdaptor  │   │       其他适配器...           │   │
│   │  - configure()   │   └──────────────────────────────┘   │
│   │  - create()      │                                       │
│   │  - remove()      │   ┌──────────────────────────────┐   │
│   │  - fetch()       │   │ installAdaptor / getAdaptor  │   │
│   └──────────────────┘   └──────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            ▲
                            │
┌───────────────────────────┴─────────────────────────────────┐
│                      HTTP & 事件层                           │
│   WorkspaceServer (HTTP), SSE Streams, GlobalBus (Events)    │
└─────────────────────────────────────────────────────────────┘
```

---

## 六、关键依赖关系

| 模块 | 依赖 |
|------|------|
| `workspace.ts` | Database, GlobalBus, adaptors |
| `server.ts` | WorkspaceContext, Instance, routes |
| `routes.ts` | GlobalBus（事件） |
| `middleware.ts` | WorkspaceContext, Workspace, adaptors |
| `WorktreeAdaptor` | Worktree, server（用于 fetch） |

---

## 七、架构设计特点

| 模式 | 说明 |
|------|------|
| **适配器模式** | 通过 `Adaptor` 接口统一处理不同类型的工作空间（worktree 等） |
| **Context 模式** | 通过 `WorkspaceContext` 管理请求级别的工作空间状态 |
| **SSE + 事件总线** | 通过 SSE 长连接 + `GlobalBus` 实现实时工作空间事件同步 |
| **分层架构** | HTTP 层 → 业务层 → 适配器层 → 持久化层，各层职责清晰 |
| **请求代理** | `WorkspaceRouterMiddleware` 统一拦截并转发至目标工作空间 |
