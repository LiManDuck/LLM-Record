
# 核心架构

- 双层队列 ： 允许用户多个输入， session级串行

- 大量的重试保证顺畅：- llm api调用根据不同的返回错误类型采取不同的重试 - 多次压缩： 压缩仍超长，继续压缩，还不行，则丢弃最早消息


- 上下文的管理， context Engine  

- 不同厂商模型接口差异的适配，不是采用adapter, 而是采用装饰器链（Wrapper Chain）

- plugin系统

- 底层 依靠openclaw 
- Gateway 架构
- hooks系统： 


# 核心文件

- run.ts
- attempt.ts
- campaction.ts
- context-engine
- plugin
- skill 
- gateway

# 工作区

```json

~/.openclaw/workspace
/├── IDENTITY.md    # Agent 身份定义
├── SOUL.md        # 行为准则（"灵魂"）
├── USER.md        # 用户画像
├── AGENTS.md      # 操作指令和记忆规则
├── TOOLS.md       # 工具使用指南
├── MEMORY.md      # 长期记忆（仅 private session 加载）
├── memory/        # 每日 append-only 日志
└── HEARTBEAT.md   # 心跳检查清单
]
```


# 记忆

- Memory Flush 机制：context window 快满时，先触发 silent turn 让 agent 将可持久信息写入 memory 文件，然后再压缩——"先存档再遗忘" 


# Agent 循环





#  工具
## Browser




# Open Claw 源码


1. run.ts (693 行)

  类/类型

  - 无类定义，主要是函数
  - 导出 runEmbeddedPiAgent 函数
  - 使用 RunEmbeddedPiAgentParams (来自 run/params.ts) 作为参数类型

  主要属性/导入

  - 导入 failoverError, auth-profiles, context-window-guard 等模块
  - 导入 compactEmbeddedPiSessionDirect, runEmbeddedAttempt, buildEmbeddedRunPayloads
  - 导入 resolveGlobalLane, resolveSessionLane 用于队列管理

  调用逻辑

  runEmbeddedPiAgent() [run.ts]
    ├─ resolveSessionLane() → resolveGlobalLane() [lanes.ts]
    ├─ enqueueSession() → enqueueGlobal() (command-queue)
    ├─ resolveModel() [model.ts]
    │   ├─ discoverAuthStorage()
    │   ├─ discoverModels()
    │   ├─ find() or buildInlineProviderModels() → normalizeModelCompat()
    ├─ resolveContextWindowInfo() + evaluateContextWindowGuard()
    ├─ ensureAuthProfileStore() → resolveAuthProfileOrder()
    ├─ while 循环:
    │   ├─ applyApiKeyInfo() → getApiKeyForModel() [model-auth.ts]
    │   ├─ runEmbeddedAttempt() [run/attempt.ts]
    │   │   └─ 返回: embedded run attempt结果
    │   ├─ 错误处理:
    │   │   ├─ isContextOverflowError → compactEmbeddedPiSessionDirect() [compact.ts]
    │   │   ├─ isFailoverErrorMessage → advanceAuthProfile() → 重试
    │   │   │           或 PickFallbackThinkingLevel() → 重试
    │   │   └─ isFailoverAssistantError → markAuthProfileFailure()
    │   └─ buildEmbeddedRunPayloads() [run/payloads.ts]
    └─ 返回: EmbeddedPiRunResult [types.ts]

  ---
  2. run/attempt.ts (906 行)

  类/类型

  - 无类定义
  - 导入 EmbeddedRunAttemptParams, EmbeddedRunAttemptResult (来自 run/types.ts)

  主要属性/导入

  - 导入 SessionManager, createAgentSession 等 (从 @mariozechner/pi-coding-agent)
  - 导入 createOpenClawCodingTools, buildEmbeddedSystemPrompt
  - 导入 subscribeEmbeddedPiSession, denseSessionManagerAccess
  - 导入 detectAndLoadPromptImages [run/images.ts]

  调用逻辑

  runEmbeddedAttempt() [run/attempt.ts]
    ├─ resolveSandboxContext() [sandbox.ts]
    ├─ loadWorkspaceSkillEntries() → applySkillEnvOverrides() [skills.ts]
    ├─ resolveBootstrapContextForRun() [bootstrap-files.ts]
    ├─ createOpenClawCodingTools() [pi-tools.ts]
    │   └─ sanitizeToolsForGoogle() [google.ts] → logToolSchemasForGoogle()
    ├─ resolveChannelCapabilities() → resolveTelegramInlineButtonsScope() 等
    ├─ resolveSessionAgentIds() [agent-scope.ts]
    ├─ buildEmbeddedSandboxInfo() [sandbox-info.ts]
    ├─ buildSystemPromptParams() [system-prompt-params.ts]
    ├─ buildEmbeddedSystemPrompt() [system-prompt.ts]
    ├─ prewarmSessionFile() → trackSessionManagerAccess() [session-manager-cache.ts]
    ├─ SessionManager.open() → guardSessionManager() [session-tool-result-guard-wrapper.ts]
    ├─ prepareSessionManagerForRun() [session-manager-init.ts]
    ├─ SettingsManager.create() → ensurePiCompactionReserveTokens()
    ├─ buildEmbeddedExtensionPaths() [extensions.ts]
    ├─ splitSdkTools() → [tool-split.ts]
    ├─ toClientToolDefinitions() [pi-tool-definition-adapter.ts] (客户端工具)
    ├─ createAgentSession() → applySystemPromptOverrideToSession()
    ├─ createCacheTrace() → wrapStreamFn() [cache-trace.ts]
    ├─ createAnthropicPayloadLogger() → wrapStreamFn()
    ├─ applyExtraParamsToAgent() [extra-params.ts]
    ├─ sanitizeSessionHistory() [google.ts]
    │   ├─ sanitizeSessionMessagesImages() [tool-images.ts]
    │   ├─ sanitizeAntigravityThinkingBlocks()
    │   ├─ validateAnthropicTurns() / validateGeminiTurns()
    │   └─ applyGoogleTurnOrderingFix() + markGoogleTurnOrderingMarker()
    ├─ limitHistoryTurns() [history.ts]
    ├─ subscribeEmbeddedPiSession() [pi-embedded-subscribe.ts]
    │   └─ 返回: subscription对象 {assistantTexts, toolMetas, ...}
    ├─ setActiveEmbeddedRun() [runs.ts]
    ├─ detectAndLoadPromptImages() [run/images.ts]
    │   └─ loadImageFromRef() → assertSandboxPath()
    ├─ appendCacheTtlTimestamp() [cache-ttl.ts]
    ├─ activeSession.prompt() → 循环: steer() 如果需要
    ├─ waitForCompactionRetry()
    ├─ getGlobalHookRunner().runBeforeAgentStart() [hook-runner-global.ts]
    ├─ getGlobalHookRunner().runAgentEnd() (异步)
    └─ 返回: EmbeddedRunAttemptResult

  ---
  3. run/types.ts (105 行)

  类/类型

  export type EmbeddedRunAttemptParams = {
    sessionId, sessionKey, messageChannel, ... // ~30个属性
    sessionFile, workspaceDir, config, skillsSnapshot
    prompt, images?, clientTools?, disableTools?
    provider, modelId, model, authStorage, modelRegistry
    thinkLevel, verboseLevel, reasoningLevel
    execOverrides, bashElevated
    timeoutMs, runId, abortSignal
    shouldEmitToolResult, shouldEmitToolOutput
    onPartialReply, onAssistantMessageStart, onBlockReply, onBlockReplyFlush
    blockReplyBreak, blockReplyChunking
    onReasoningStream, onToolResult, onAgentEvent
    extraSystemPrompt, streamParams, ownerNumbers, enforceFinalTag
  }

  export type EmbeddedRunAttemptResult = {
    aborted, timedOut, promptError
    sessionIdUsed, systemPromptReport
    messagesSnapshot, assistantTexts, toolMetas
    lastAssistant, lastToolError
    didSendViaMessagingTool, messagingToolSentTexts, messagingToolSentTargets
    cloudCodeAssistFormatError, clientToolCall?
  }

  调用逻辑

  - 纯类型定义文件，被其他文件引用
  - 被声明相同名称的 run/params.ts 覆盖 RunEmbeddedPiAgentParams

  ---
  4. run/params.ts (99 行)

  类/类型

  export type ClientToolDefinition = { // 客户端提供的工具定义
    type: "function"
    function: { name, description?, parameters? }
  }

  export type RunEmbeddedPiAgentParams = { // 外部运行参数
    sessionId, sessionKey, messageChannel, messageProvider
    messageTo, messageThreadId, groupId, groupChannel, groupSpace
    spawnedBy, senderId, senderName, senderUsername, senderE164
    currentChannelId, currentThreadTs, replyToMode, hasRepliedRef
    sessionFile, workspaceDir, agentDir, config, skillsSnapshot
    prompt, images?, clientTools?, disableTools?
    provider?, model?, authProfileId?, authProfileIdSource
    thinkLevel?, verboseLevel?, reasoningLevel?
    toolResultFormat?, execOverrides?, bashElevated?
    timeoutMs, runId, abortSignal
    shouldEmitToolResult, shouldEmitToolOutput
    onPartialReply, onAssistantMessageStart, onBlockReply...
    lane?, enqueue, extraSystemPrompt, streamParams, ownerNumbers, enforceFinalTag
  }
  - RunEmbeddedPiAgentParams 传递给 runEmbeddedPiAgent() [run.ts]

  ---
  5. run/payloads.ts (256 行)

  类/类型

  - 无类定义
  - ToolMetaEntry = { toolName: string; meta?: string }

  主要函数

  - buildEmbeddedRunPayloads(): 构建响应载荷

  调用逻辑

  buildEmbeddedRunPayloads() [run/payloads.ts]
    ├─ formatAssistantErrorText() [pi-embedded-helpers.ts]
    ├─ formatRawAssistantErrorForUi()
    ├─ getApiErrorPayloadFingerprint()
    ├─ formatToolAggregate() [auto-reply/tool-meta.ts]
    ├─ parseReplyDirectives() [auto-reply/reply/reply-directives.ts]
    ├─ extractAssistantThinking() → formatReasoningMessage() [pi-embedded-utils.ts]
    ├─ extractAssistantText()
    ├─ normalizeTextForComparison()
    └─ 返回: 载荷数组 [{ text?, mediaUrls?, isError?, ... }]

  ---
  6. run/images.ts (448 行)

  类/类型

  export interface DetectedImageRef {
    raw: string
    type: "path" | "url"
    resolved: string
    messageIndex?: number // 历史图片所在的消息索引
  }

  主要函数

  - detectImageReferences(): 在提示词中检测图片引用
  - loadImageFromRef(): 从引用加载图片
  - modelSupportsImages(): 检查模型是否支持图片
  - detectAndLoadPromptImages(): 检测并加载提示词中的图片（包括历史）

  调用逻辑

  detectAndLoadPromptImages() [run/images.ts]
    ├─ modelSupportsImages()
    ├─ detectImageReferences(prompt)
    │   ├─ 匹配 mediaAttachedPattern: [media attached: path (type) | url]
    │   ├─ 匹配 messageImagePattern: [Image: source: /path/...]
    │   ├─ 匹配 fileUrlPattern: file:///...
    │   └─ 匹配 pathPattern: /path/to/image.png
    ├─ detectImagesFromHistory(messages) → extractTextFromMessage()
    ├─ loadImageFromRef(ref)
    │   ├─ resolveSandboxPath() [sandbox-paths.ts] (如果 sandboxRoot 设置)
    │   ├─ fs.stat()
    │   ├─ loadWebMedia() [web/media.ts]
    │   └─ sanitizeImageBlocks() [tool-images.ts]
    └─ 返回: { images, historyImagesByIndex, detectedRefs, loadedCount, skippedCount }

  ---
  7. types.ts (80 行)

  核心类型

  export type EmbeddedPiAgentMeta = {
    sessionId, provider, model
    usage?: { input?, output?, cacheRead?, cacheWrite?, total? }
  }

  export type EmbeddedPiRunMeta = {
    durationMs, aborted
    systemPromptReport?, error?
    stopReason?, pendingToolCalls? (客户端工具调用)
  }

  export type EmbeddedPiRunResult = {
    payloads?: [{ text?, mediaUrl?, mediaUrls?, replyToId?, isError? }]
    meta: EmbeddedPiRunMeta
    didSendViaMessagingTool?, messagingToolSentTexts?, messagingToolSentTargets?
  }

  export type EmbeddedPiCompactResult = {
    ok, compacted, reason?
    result?: { summary, firstKeptEntryId, tokensBefore, tokensAfter?, details? }
  }

  export type EmbeddedSandboxInfo = {
    enabled, workspaceDir?, workspaceAccess? ("none" | "ro" | "rw")
    agentWorkspaceMount?, browserBridgeUrl?, browserNoVncUrl?, hostBrowserAllowed?
    elevated?: { allowed, defaultLevel }
  }

  ---
  8. compact.ts (490 行)

  类/类型

  export type CompactEmbeddedPiSessionParams = {
    sessionId, sessionKey, messageChannel, messageProvider
    groupId, groupChannel, groupSpace, spawnedBy
    sessionFile, workspaceDir, agentDir, config, skillsSnapshot
    provider?, model?, thinkLevel?, reasoningLevel?
    bashElevated?, customInstructions?
    lane?, enqueue, extraSystemPrompt, ownerNumbers
  }

  主要函数

  - compactEmbeddedPiSessionDirect(): 核心压缩逻辑（无队列）
  - compactEmbeddedPiSession(): 队列封装

  调用逻辑

  compactEmbeddedPiSession() [compact.ts]
    ├─ resolveSessionLane() → enqueueCommandInLane(sessionLane)
    │   └─ resolveGlobalLane() → enqueueGlobal()
    └─ compactEmbeddedPiSessionDirect()
        ├─ resolveModel() [model.ts] → getApiKeyForModel() → setRuntimeApiKey()
        ├─ resolveSandboxContext() [sandbox.ts]
        ├─ ensureSessionHeader() [pi-embedded-helpers.ts]
        ├─ loadWorkspaceSkillEntries() → applySkillEnvOverrides()
        ├─ resolveBootstrapContextForRun()
        ├─ createOpenClawCodingTools() → sanitizeToolsForGoogle()
        ├─ buildEmbeddedSandboxInfo()
        ├─ 构建系统提示词相关 (类似 run/attempt.ts)
        ├─ SessionManager.open() → guardSessionManager()
        ├─ SettingsManager.create() → ensurePiCompactionReserveTokens()
        ├─ buildEmbeddedExtensionPaths() [extensions.ts]
        ├─ splitSdkTools()
        ├─ createAgentSession() → applySystemPromptOverrideToSession()
        ├─ sanitizeSessionHistory() [google.ts]
        ├─ limitHistoryTurns() [history.ts]
        ├─ session.compact(customInstructions)
        ├─ estimateTokens(session.messages) → tokensAfter
        └─ 返回: EmbeddedPiCompactResult

  ---
  9. model.ts (112 行)

  类/类型

  type InlineModelEntry = ModelDefinitionConfig & { provider: string; baseUrl?: string }
  type InlineProviderConfig = { baseUrl?, api?, models? }
  type ResolveModelResult = {
    model?: Model<Api>
    error?: string
    authStorage: AuthStorage
    modelRegistry: ModelRegistry
  }

  主要函数

  - buildInlineProviderModels(): 从配置构建内联模型列表
  - buildModelAliasLines(): 构建模型别名列表
  - resolveModel(): 解析模型（从注册表或内联配置）

  调用逻辑

  resolveModel() [model.ts]
    ├─ discoverAuthStorage(agentDir)
    ├─ discoverModels(authStorage, agentDir)
    ├─ modelRegistry.find(provider, modelId)
    │   ├─ 如果找到 → normalizeModelCompat(model)
    │   └─ 如果未找到 → fallback:
    │       ├─ buildInlineProviderModels(providers)
    │       ├─ 查找匹配的 inlineModel
    │       ├─ normalizeModelCompat() 或创建 fallbackModel
    │       └─ 返回 { model: ..., authStorage, modelRegistry }
    └─ 返回 ResolveModelResult

  ---
  10. extensions.ts (105 行)

  主要函数

  - resolvePiExtensionPath(): 解析扩展路径
  - resolveContextWindowTokens(): 解析上下文窗口token数
  - buildContextPruningExtension(): 构建上下文裁剪扩展
  - resolveCompactionMode(): 解析压缩模式
  - buildEmbeddedExtensionPaths(): 构建嵌入式扩展路径

  调用逻辑

  buildEmbeddedExtensionPaths() [extensions.ts]
    ├─ resolveCompactionMode() → "default" | "safeguard"
    ├─ 如果 "safeguard":
    │   ├─ setCompactionSafeguardRuntime() [pi-extensions/compaction-safeguard-runtime.ts]
    │   └─ resolvePiExtensionPath("compaction-safeguard")
    └─ buildContextPruningExtension()
        ├─ if mode !== "cache-ttl" → return {}
        ├─ isCacheTtlEligibleProvider() [cache-ttl.ts]
        ├─ computeEffectiveSettings() [pi-extensions/context-pruning/settings.ts]
        ├─ makeToolPrunablePredicate() [pi-extensions/context-pruning/tools.ts]
        ├─ readLastCacheTtlTimestamp() [cache-ttl.ts]
        ├─ setContextPruningRuntime() [pi-extensions/context-pruning/runtime.ts]
        └─ 返回扩展路径列表

  ---
  11. session-manager-init.ts (54 行)

  类/类型

  type SessionHeaderEntry = { type: "session"; id?: string; cwd?: string }
  type SessionMessageEntry = { type: "message"; message?: { role?: string } }

  主要函数

  - prepareSessionManagerForRun(): 准备SessionManager以供运行

  调用逻辑

  prepareSessionManagerForRun() [session-manager-init.ts]
    ├─ 检查 sessionManager 属性
    ├─ 如果!hadSessionFile && header存在 → 设置 session header info
    ├─ 如果 hadSessionFile && header存在 && !hasAssistant
    │   ├─ fs.writeFile(sessionFile, "")
    │   ├─ 重置 fileEntries, byId, labelsById, leafId
    │   └─ flushed = false (允许首次 assistant flush 包含 header+user)
    └─ 确保会话状态正确（初始化或已存在文件时的正确行为）

  ---
  12. history.ts (99 行)

  主要函数

  - stripThreadSuffix(): 移除线程后缀
  - limitHistoryTurns(): 限制历史记录轮数
  - getDmHistoryLimitFromSessionKey(): 从session key获取DM历史限制

  调用逻辑

  limitHistoryTurns() [history.ts]
    └─ 从后往前扫描，保留最后N个user消息 + 对应的assistant响应

  getDmHistoryLimitFromSessionKey() [history.ts]
    ├─ 解析 session key: [agent:]provider[:dm[...]]
    ├─ 提取 provider, userId (去掉 :thread: 或 :topic: 后缀)
    ├─ check kind === "dm"
    ├─ resolveProviderConfig(cfg, provider)
    ├─ 查找 dms[userId].historyLimit 或 dmHistoryLimit
    └─ 返回限制数 或 undefined

  ---
  13. lanes.ts (16 行)

  主要函数

  - resolveSessionLane(key): 解析会话lane
  - resolveGlobalLane(lane?): 解析全局lane
  - resolveEmbeddedSessionLane(key): 别名

  调用逻辑

  - 简单的字符串解析和格式化
  - resolveSessionLane: 返回 session:key 或 session: + key
  - resolveGlobalLane: 返回 lane 或 CommandLane.Main

  ---
  14. logger.ts (4 行)

  类/类型

  - 无类，导出 log 常量

  调用逻辑

  export const log = createSubsystemLogger("agent/embedded")

  ---
  15. sandbox-info.ts (31 行)

  类/类型

  - 无类，导出 buildEmbeddedSandboxInfo() 函数

  调用逻辑

  buildEmbeddedSandboxInfo() [sandbox-info.ts]
    ├─ if !sandbox?.enabled → return undefined
    ├─ elevatedAllowed = execElevated?.enabled && execElevated?.allowed
    └─ 返回 EmbeddedSandboxInfo:
        { enabled, workspaceDir, workspaceAccess, agentWorkspaceMount
          browserBridgeUrl, browserNoVncUrl, hostBrowserAllowed
          elevated?: { allowed, defaultLevel } }

  ---
  16. runs.ts (141 行)

  类/类型

  export type EmbeddedPiQueueHandle = {
    queueMessage: (text: string) => Promise<void>
    isStreaming: () => boolean
    isCompacting: () => boolean
    abort: () => void
  }

  主要函数

  - queueEmbeddedPiMessage(): 队列消息到运行中的会话
  - abortEmbeddedPiRun(): 中止运行中的会话
  - isEmbeddedPiRunActive(): 检查会话是否活跃
  - isEmbeddedPiRunStreaming(): 检查会话是否正在流式传输
  - waitForEmbeddedPiRunEnd(): 等待会话结束（带超时）
  - setActiveEmbeddedRun(): 设置活跃运行
  - clearActiveEmbeddedRun(): 清除活跃运行

  调用逻辑

  ACTIVE_EMBEDDED_RUNS: Map<sessionId, EmbeddedPiQueueHandle>
  EMBEDDED_RUN_WAITERS: Map<sessionId, Set<EmbeddedRunWaiter>>

  queueEmbeddedPiMessage() [runs.ts]
    └─ ACTIVE_EMBEDDED_RUNS.get(sessionId) → handle.queueMessage(text)

  abortEmbeddedPiRun() [runs.ts]
    └─ handle.abort()

  setActiveEmbeddedRun() [runs.ts]
    ├─ ACTIVE_EMBEDDED_RUNS.set(sessionId, handle)
    └─ logSessionStateChange()

  clearActiveEmbeddedRun() [runs.ts]
    ├─ if handle 匹配 → delete(sessionId)
    ├─ logSessionStateChange()
    └─ notifyEmbeddedRunEnded() → 清理 waiters

  ---
  17. tool-split.ts (18 行)

  类/类型

  type AnyAgentTool = AgentTool
  export function splitSdkTools(options: {
    tools: AnyAgentTool[]
    sandboxEnabled: boolean
  }): {
    builtInTools: AnyAgentTool[]
    customTools: ReturnType<typeof toToolDefinitions>
  }

  调用逻辑

  splitSdkTools() [tool-split.ts]
    └─ 返回 { builtInTools: [], customTools: toToolDefinitions(tools) }

  ---
  18. abort.ts (13 行)

  类/类型

  - 无类

  主要函数

  - isAbortError(err): 检查错误是否为中止错误

  调用逻辑

  isAbortError()
    └─ 检查 name === "AbortError" 或 message包含 "aborted"

  ---
  19. google.ts (390 行)

  类/类型

  export type CompactionFailureListener = (reason: string) => void

  type ModelSnapshotEntry = { timestamp, provider?, modelApi?, modelId? }

  const compactionFailureEmitter = new EventEmitter()

  主要函数

  - sanitizeAntigravityThinkingBlocks(): 清理Antigravity思考块
  - findUnsupportedSchemaKeywords(): 查找不支持的schema关键字
  - sanitizeToolsForGoogle(): 为Google清理工具参数
  - logToolSchemasForGoogle(): 记录Google工具schema
  - onUnhandledCompactionFailure(): 监听未处理的压缩失败
  - readLastModelSnapshot(): 读取最后的模型快照
  - appendModelSnapshot(): 添加模型快照
  - applyGoogleTurnOrderingFix(): 应用Google轮序修复
  - sanitizeSessionHistory(): 清理会话历史

  调用逻辑

  sanitizeSessionHistory() [google.ts]
    ├─ resolveTranscriptPolicy()
    ├─ sanitizeSessionMessagesImages() [tool-images.ts]
    ├─ 如果 policy.normalizeAntigravityThinkingBlocks
    │   └─ sanitizeAntigravityThinkingBlocks()
    ├─ 如果 policy.repairToolUseResultPairing
    │   └─ sanitizeToolUseResultPairing() [session-transcript-repair.ts]
    ├─ readLastModelSnapshot() / appendModelSnapshot()
    ├─ 如果 modelChanged
    │   ├─ 降级OpenAI推理块: downgradeOpenAIReasoningBlocks()
    │   └─ appendModelSnapshot()
    └─ 如果 policy.applyGoogleTurnOrdering
        └─ applyGoogleTurnOrderingFix() → sanitizeGoogleTurnOrdering() + markGoogleTurnOrderingMarker()

  onUnhandledCompactionFailure() [google.ts]
    └─ registerUnhandledRejectionHandler() [infra/unhandled-rejections.ts]
         └─ 检测 isCompactionFailureError() → emit("failure")

  ---
  20. cache-ttl.ts (62 行)

  类/类型

  export const CACHE_TTL_CUSTOM_TYPE = "openclaw.cache-ttl"

  export type CacheTtlEntryData = {
    timestamp: number
    provider?: string
    modelId?: string
  }

  主要函数

  - isCacheTtlEligibleProvider(): 检查提供商是否支持Cache TTL
  - readLastCacheTtlTimestamp(): 读取最后的Cache TTL时间戳
  - appendCacheTtlTimestamp(): 添加Cache TTL时间戳

  调用逻辑

  readLastCacheTtlTimestamp() [cache-ttl.ts]
    └─ 遍历 sessionManager.getEntries() → 查找 customType === CACHE_TTL_CUSTOM_TYPE
        └─ 返回 timestamp (最后匹配的)

  appendCacheTtlTimestamp() [cache-ttl.ts]
    └─ sessionManager.appendCustomEntry(CACHE_TTL_CUSTOM_TYPE, data)

  isCacheTtlEligibleProvider() [cache-ttl.ts]
    └─ provider === "anthropic" 或 (openrouter + model.startsWith("anthropic/"))

  ---
  21. session-manager-cache.ts (70 行)

  类/类型

  type SessionManagerCacheEntry = { sessionFile: string; loadedAt: number }
  const SESSION_MANAGER_CACHE = new Map<string, SessionManagerCacheEntry>()

  主要函数

  - trackSessionManagerAccess(): 跟踪SessionManager访问
  - isSessionManagerCached(): 检查SessionManager是否已缓存
  - prewarmSessionFile(): 预热SessionFile（读取一小块数据）

  调用逻辑

  prewarmSessionFile() [session-manager-cache.ts]
    ├─ 检查缓存状态
    ├─ fs.open() → handle.read(buffer)
    └─ trackSessionManagerAccess()

  trackSessionManagerAccess() [session-manager-cache.ts]
    └─ SESSION_MANAGER_CACHE.set(sessionFile, { sessionFile, loadedAt: now })

  ---
  22. system-prompt.ts (97 行)

  主要函数

  - buildEmbeddedSystemPrompt(): 构建嵌入式系统提示词
  - createSystemPromptOverride(): 创建系统提示词覆盖函数
  - applySystemPromptOverrideToSession(): 应用系统提示词覆盖到会话

  调用逻辑

  buildEmbeddedSystemPrompt() [system-prompt.ts]
    └─ buildAgentSystemPrompt() [system-prompt.ts]
        ├─ workspaceDir, defaultThinkLevel, reasoningLevel
        ├─ extraSystemPrompt, ownerNumbers, reasoningTagHint
        ├─ heartbeatPrompt, skillsPrompt, docsPath, ttsHint
        ├─ reactionGuidance, promptMode ("full" | "minimal")
        ├─ runtimeInfo: { host, os, arch, node, model, channel, capabilities, channelActions }
        ├─ messageToolHints, sandboxInfo
        ├─ tools (仅传递名称), toolSummaries, modelAliasLines
        ├─ userTimezone, userTime, userTimeFormat
        └─ contextFiles

  createSystemPromptOverride() [system-prompt.ts]
    └─ 返回 () → trimmedPrompt

  applySystemPromptOverrideToSession() [system-prompt.ts]
    ├─ session.agent.setSystemPrompt(prompt)
    └─ 设置 session._baseSystemPrompt / _rebuildSystemPrompt

  ---
  23. extra-params.ts (157 行)

  类/类型

  type CacheRetention = "none" | "short" | "long"
  type CacheRetentionStreamOptions = Partial<SimpleStreamOptions> & {
    cacheRetention?: CacheRetention
  }

  主要函数

  - resolveExtraParams(): 从模型配置解析额外参数
  - resolveCacheRetention(): 解析缓存保留策略（支持旧版cacheControlTtl）
  - createStreamFnWithExtraParams(): 创建带额外参数的流函数
  - createOpenRouterHeadersWrapper(): 创建OpenRouter头部包装器
  - applyExtraParamsToAgent(): 应用额外参数到agent

  调用逻辑

  applyExtraParamsToAgent() [extra-params.ts]
    ├─ resolveExtraParams({ cfg, provider, modelId })
    ├─ 合并 extraParams + extraParamsOverride
    ├─ createStreamFnWithExtraParams(agent.streamFn, merged, provider)
    │   ├─ 解析: temperature, maxTokens, cacheRetention
    │   └─ 返回 wrappedStreamFn
    ├─ if provider === "openrouter"
    │   └─ createOpenRouterHeadersWrapper(agent.streamFn)
    └─ 设置 agent.streamFn

  ---
  24. utils.ts (38 行)

  主要函数

  - mapThinkingLevel(): 映射思考等级
  - resolveExecToolDefaults(): 解析exec工具默认值
  - describeUnknownError(): 描述未知错误

  调用逻辑

  mapThinkingLevel() [utils.ts]
    └─ 直接返回 level (支持 "xhigh" 等)

  resolveExecToolDefaults() [utils.ts]
    └─ return config?.tools?.exec

  describeUnknownError() [utils.ts]
    ├─ if Error → error.message
    ├─ if string → error
    └─ JSON.stringify(error) or "Unknown error"

  ---
  25. 测试文件

  run.overflow-compaction.test.ts

  - 测试溢出压缩逻辑

  model.test.ts

  - 测试 buildInlineProviderModels() 和 resolveModel()
  - 验证provider配置、baseUrl、api继承

  run/attempt.test.ts

  - 测试 injectHistoryImagesIntoMessages()
  - 验证历史图片注入、内容转换、去重逻辑

  run/payloads.test.ts

  - 测试 buildEmbeddedRunPayloads()
  - 验证错误消息抑制、工具错误处理、可恢复错误隐藏

  ---
  完整调用流程图

  外部调用
    ↓
  runEmbeddedPiAgent() [run.ts]
    ├─ resolveModel() → [model.ts]
    │   └─ buildInlineProviderModels() 或 discoverModels().find()
    ├─ while 循环:
    │   ├─ applyApiKeyInfo() → [model-auth.ts]
    │   ├─ runEmbeddedAttempt() → [run/attempt.ts]
    │   │   ├─ createOpenClawCodingTools() → [pi-tools.ts]
    │   │   ├─ buildEmbeddedSystemPrompt() → [system-prompt.ts]
    │   │   ├─ SessionManager.open() → [session-manager-init.ts]
    │   │   ├─ createAgentSession() → applySystemPromptOverrideToSession()
    │   │   ├─ setActiveEmbeddedRun() → [runs.ts]
    │   │   ├─ sanitizeSessionHistory() → [google.ts]
    │   │   │   ├─ sanitizeSessionMessagesImages() → [tool-images.ts]
    │   │   │   └─ applyGoogleTurnOrderingFix()
    │   │   ├─ subscribeEmbeddedPiSession() → [pi-embedded-subscribe.ts]
    │   │   ├─ detectAndLoadPromptImages() → [run/images.ts]
    │   │   ├─ activeSession.prompt()
    │   │   ├─ 失败 → compactEmbeddedPiSessionDirect() [compact.ts]
    │   │   └─ 返回 EmbeddedRunAttemptResult
    │   └─ buildEmbeddedRunPayloads() → [run/payloads.ts]
    └─ 返回 EmbeddedPiRunResult [types.ts]

  compactEmbeddedPiSession() [compact.ts]
    ├─ 队列: resolveSessionLane() → enqueueCommandInLane(sessionLane)
    │   └─ `compactEmbeddedPiSessionDirect()`
    └─ 类似 runEmbeddedAttempt 的初始化流程，但调用 session.compact()

  辅助模块:
  - [lanes.ts] → lane 解析
  - [runs.ts] → 活跃运行管理
  - [session-manager-cache.ts] → SessionManager缓存
  - [ Extensions.ts] → 扩展路径构建
  - [google.ts] → Google特定清理和工具处理
  - [cache-ttl.ts] → Cache TTL追踪
  - [logger.ts] → 日志
  - [abort.ts] → Abort错误检测
  - [sandbox-info.ts] → 沙箱信息构建
