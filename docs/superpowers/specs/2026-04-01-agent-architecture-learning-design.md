# Agent 架构深度学习计划 — 融合版

## 目标

通过系统性研读 Claude Code 源码并构建精简版 `mini-claude`，深度掌握业界顶级 agent 系统的核心设计模式，形成可迁移到任意语言/场景的 agent 架构能力。

## 核心原则

1. **主链优先**：以数据流 `输入 → processUserInput → QueryEngine → query() → [API call → tool execution → loop]` 为骨架，围绕主链回看支撑模块
2. **先读后造**：走通完整请求路径后再开始 `mini-claude`（Step 6 之后）
3. **锚点稳定**：笔记引用优先使用 `文件#符号名`，仅在必须精确定位时补行号
4. **验证分级**：read / static / runtime 三级验证，不假设每步都能直接运行
5. **stub 识别**：遇到自动生成 stub 只做识别，不作为核心学习材料
6. **细节不丢**：每个支撑模块保留完整子单元拆解，确保无遗漏

## 当前仓库的五层主学习坐标

```
1. 交互层    src/main.tsx, src/screens/REPL.tsx
2. 编排层    src/QueryEngine.ts
3. 核心循环  src/query.ts
4. 工具层    src/tools.ts, src/Tool.ts
5. 通信层    src/services/api/claude.ts
```

学习时始终围绕这条主链回看支撑模块。

---

## Phase 0: 环境与验证前提闸门

在进入任何"运行验证"之前，必须先通过本闸门。

### 闸门清单

1. `node_modules` 已存在，或 `bun install` 成功
2. `bun run dev --help` 可执行
3. `bun test` 可执行
4. 若涉及 API 链路验证，认证与密钥前提已明确

### 闸门未通过时的规则

- 允许源码阅读、架构图、模块边界设计
- 不承诺运行验证
- 不把"跑不起来"误判为"理解错误"

---

## Part I: 主链理解（Step 1-6）

### Step 1: 消息生命周期与主链边界

**核心问题**：用户输入如何进入系统？消息在进入 `query()` 之前经过哪些变形？哪些消息给 UI 看，哪些给 API 看？

**优先文件**：
- `src/utils/processUserInput/processUserInput.ts#processUserInput`
- `src/utils/messages.ts#normalizeMessagesForAPI`
- `src/QueryEngine.ts#submitMessage`
- `src/query.ts#query`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 1.1 | 消息类型分层与 stub 识别（`src/types/message.ts` 是 auto-generated stub，只做识别，不作为主实现锚点） | `src/utils/messages.ts#normalizeMessages`, `src/types/message.ts` |
| 1.2 | 内容块类型在规范化链中的进入/筛选/保留方式（text, tool_use, tool_result, thinking） | `src/utils/messages.ts#normalizeMessagesForAPI` |
| 1.3 | processUserInput 的输入预处理（slash 命令、附件、图片） | `src/utils/processUserInput/processUserInput.ts` |
| 1.4 | normalizeMessagesForAPI — UI 消息 vs API 消息的分离边界 | `src/utils/messages.ts#normalizeMessagesForAPI` |

**产出**：一张"消息从输入到循环"的时序图 + 笔记

**验证级别**：read

---

### Step 2: 入口与会话启动

**核心问题**：CLI 如何启动并分流不同模式？初始化 30+ 步的编排逻辑？

**优先文件**：
- `src/entrypoints/cli.tsx`
- `src/main.tsx`
- `src/entrypoints/init.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 2.1 | cli.tsx 启动薄层：env 修正 + `feature()` build-time gating + `MACRO` 常量内联 | `src/entrypoints/cli.tsx` |
| 2.2 | Commander.js CLI 定义与模式分流（interactive/headless/pipe） | `src/main.tsx` |
| 2.3 | init() 30 步初始化编排（并行预取、依赖排序、懒加载） | `src/entrypoints/init.ts#init` |
| 2.4 | 分层配置系统（user → project → local → flag + policy → managed → remote managed → MDM） | `src/utils/config.ts`, `src/utils/settings/settings.ts#getInitialSettings` |
| 2.5 | Bootstrap 状态单例（sessionId, cwd, tokenCounts） | `src/bootstrap/state.ts` |
| 2.6 | Auth 流程（OAuth, API key, keychain, 多 Provider） | `src/utils/auth.ts`, `src/services/oauth/client.ts` |
| 2.7 | Provider 选择（Anthropic / Bedrock / Vertex / Foundry） | `src/utils/model/providers.ts#getAPIProvider` |

**产出**：入口分发图 + startup fast-path 笔记

**验证级别**：read

---

### Step 3: QueryEngine 编排层

**核心问题**：QueryEngine 与 query() 的边界是什么？为什么它不是主循环本体，但又是会话编排核心？

**优先文件**：
- `src/QueryEngine.ts#submitMessage`
- `src/utils/queryContext.ts`
- `src/utils/sessionStorage.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 3.1 | submitMessage() 职责边界（预处理 → 系统提示 → query() → 后处理） | `src/QueryEngine.ts#submitMessage` |
| 3.2 | 单轮 vs 多轮对话（历史累积机制） | `src/QueryEngine.ts` |
| 3.3 | Token 计数与估算 | `src/utils/tokens.ts` |
| 3.4 | 转录记录与会话状态管理 | `src/utils/sessionStorage.ts` |

**产出**：submitMessage() 职责边界笔记 + QueryEngine 内部状态图

**验证级别**：read

---

### Step 4: API 客户端与流式传输

**核心问题**：API 请求如何发出？流式事件如何累积为 assistant 消息？重试/fallback/错误恢复如何发生？

**优先文件**：
- `src/services/api/claude.ts#queryModelWithStreaming`
- `src/services/api/client.ts#getAnthropicClient`
- `src/services/api/withRetry.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 4.1 | Anthropic SDK 集成与多 Provider 客户端初始化 | `src/services/api/client.ts` |
| 4.2 | 流式事件处理链（message_start → content_block_start → delta → stop → message_delta） | `src/services/api/claude.ts` |
| 4.3 | 部分消息累积（contentBlocks[] 索引追加，delta 拼接） | `src/services/api/claude.ts` |
| 4.4 | 重试与错误恢复（529 过载 / 429 限流 / 401 认证刷新 / 连接错误） | `src/services/api/withRetry.ts` |
| 4.5 | 流式空闲超时与停滞检测（默认 90s idle timeout，45s warning，30s stall detection） | `src/services/api/claude.ts` |

**产出**：streaming 事件处理图 + delta 累积与恢复策略笔记

**验证级别**：read → runtime（如果 Phase 0 闸门通过）

---

### Step 5: 工具系统 — 抽象、注册与执行

**核心问题**：Tool 抽象的最小必要字段？工具如何注册与过滤？单次 tool use 从校验到结果返回经过哪些阶段？

**优先文件**：
- `src/Tool.ts`
- `src/tools.ts#getAllBaseTools`
- `src/services/tools/toolExecution.ts`
- `src/services/tools/toolOrchestration.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 5.1 | Tool 接口设计（name, inputSchema, call, description, prompt, isConcurrencySafe, isReadOnly, checkPermissions...） | `src/Tool.ts#Tool` |
| 5.2 | ToolUseContext 执行环境（abortController, readFileState LRU, getAppState/setAppState） | `src/Tool.ts#ToolUseContext` |
| 5.3 | ToolPermissionContext 权限上下文（mode, rules, additionalWorkingDirectories） | `src/Tool.ts#ToolPermissionContext` |
| 5.4 | buildTool() 工厂函数与 Zod Schema 验证 | `src/Tool.ts#buildTool` |
| 5.5 | 工具注册表与过滤（getAllBaseTools → getTools → assembleToolPool，feature flag + deny 规则前置过滤） | `src/tools.ts` |
| 5.6 | 完整执行管线（validate → backfillObservableInput → Pre-Hook → permission → call → result → Post-Hook） | `src/services/tools/toolExecution.ts` |
| 5.7 | 精读 FileReadTool（设备路径拦截、图片/PDF、LRU 缓存） | `src/tools/FileReadTool/` |
| 5.8 | 精读 BashTool（命令解析、沙箱、超时、搜索/只读检测、后台任务） | `src/tools/BashTool/` |
| 5.9 | 精读 FileEditTool（字符串匹配、patch 生成、文件历史追踪） | `src/tools/FileEditTool/` |
| 5.10 | 工具结果格式化与持久化（超过 maxResultSizeChars → 磁盘 → preview + 路径） | `src/utils/toolResultStorage.ts` |
| 5.11 | 工具 Prompt 注入（每个工具通过 prompt() 贡献到系统提示） | `src/constants/prompts.ts#getSystemPrompt` |

**产出**：tool execution pipeline 图 + 三篇工具样板分析笔记

**验证级别**：read → static（类型验证）

---

### Step 6: ReAct 核心循环与工具编排

**核心问题**：while(true) 的真正继续条件？工具批次如何切分？流式输出与工具执行如何交织？

**优先文件**：
- `src/query.ts#queryLoop`
- `src/services/tools/StreamingToolExecutor.ts`
- `src/services/tools/toolOrchestration.ts#runTools`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 6.1 | while(true) + needsFollowUp 模式（tool_use 块检测 → 执行 → 追加结果 → 继续） | `src/query.ts` |
| 6.2 | 循环状态结构（messages, toolUseContext, turnCount, autoCompactTracking...） | `src/query.ts` |
| 6.3 | 并发安全分区（partitionToolCalls — isConcurrencySafe + isReadOnly） | `src/services/tools/toolOrchestration.ts#partitionToolCalls` |
| 6.4 | runTools() 批次执行（concurrent 批并行 max 10，serial 批串行） | `src/services/tools/toolOrchestration.ts#runTools` |
| 6.5 | StreamingToolExecutor（模型流式输出期间同步执行工具 — 性能关键优化） | `src/services/tools/StreamingToolExecutor.ts` |
| 6.6 | 7 个 continue 站点与 10 个 exit 条件 | `src/query.ts` |
| 6.7 | 进度报告（onProgress 回调，2s 阈值后开始） | `src/services/tools/toolExecution.ts` |

**产出**：ReAct 循环状态图 + needsFollowUp 与继续条件笔记

**验证级别**：read → runtime（跑通 bun test 中相关测试）

---

## Part II: mini-claude 第一版

**实施时机**：完成 Step 1-6 后（已走通主链）

**第一版只做**：
1. 非交互输入（pipe mode）
2. 消息对象与规范化边界
3. 单轮 / 多轮 query loop（while + needsFollowUp）
4. 1-2 个基础工具（FileRead, Bash）
5. 基础系统提示注入

**第一版暂不做**：worktree, swarm, remote sessions, compaction 全家族, 企业级 settings

**项目结构**：
```
mini-claude/
├── src/
│   ├── types/          # 消息类型
│   ├── api/            # API 客户端 + 流式
│   ├── engine/         # QueryEngine + query loop
│   ├── tools/          # 工具系统
│   └── context/        # 系统提示
├── package.json
└── tsconfig.json
```

---

## Part III: 支撑系统深入（Step 7-13）

### Step 7: 错误恢复与弹性

**核心问题**：query 循环中有哪些恢复机制？它们如何使 agent 在面对各种错误时继续工作？

**优先文件**：`src/query.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 7.1 | Prompt-too-long 恢复（context collapse drain → reactive compact） | `src/query.ts` |
| 7.2 | Max-output-tokens 恢复（escalate 到 64k → recovery 重试最多 3 次） | `src/query.ts` |
| 7.3 | 模型降级（FallbackTriggeredError → 切换 fallbackModel） | `src/query.ts` |
| 7.4 | 流式中断处理（abortController → getRemainingResults → 补充缺失 tool_result） | `src/query.ts` |
| 7.5 | Stop Hooks（post-completion 阻塞检查 → preventContinuation） | `src/query.ts` |

**验证级别**：read

---

### Step 8: 系统提示、上下文构建与记忆

**核心问题**：系统提示由哪些静态与动态部分构成？CLAUDE.md / memory / git status 如何注入？

**优先文件**：
- `src/constants/prompts.ts#getSystemPrompt`
- `src/constants/systemPromptSections.ts`
- `src/context.ts#getUserContext`, `#getSystemContext`
- `src/utils/claudemd.ts`
- `src/memdir/memdir.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 8.1 | 系统提示组合架构（static 段 + DYNAMIC_BOUNDARY + dynamic 段 — 缓存范围分界） | `src/constants/prompts.ts` |
| 8.2 | systemPromptSection 缓存 vs DANGEROUS_uncached 机制 | `src/constants/systemPromptSections.ts` |
| 8.3 | CLAUDE.md 发现层次（managed → user → project → local，@include 指令，40000 字符上限） | `src/utils/claudemd.ts#getMemoryFiles` |
| 8.4 | Memory 目录系统（MEMORY.md 入口，200 行截断，4 种记忆类型） | `src/memdir/memdir.ts` |
| 8.5 | 环境上下文注入（git status, 当前日期, 工作目录） | `src/context.ts#getSystemContext` |
| 8.6 | 有效系统提示构建（overrideSystemPrompt > agent > custom > default > append） | `src/utils/systemPrompt.ts#buildEffectiveSystemPrompt` |
| 8.7 | 输出样式模板系统 | `src/outputStyles/` |

**产出**：system prompt 组装图 + getUserContext/getSystemContext 笔记

**验证级别**：read

---

### Step 9: 权限、设置与运行边界

**核心问题**：权限模式与规则如何作用于工具执行？配置的真实来源与优先级？

**优先文件**：
- `src/utils/permissions/permissions.ts#hasPermissionsToUseTool`
- `src/utils/permissions/permissionSetup.ts`
- `src/utils/settings/settings.ts#getInitialSettings`
- `src/utils/config.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 9.1 | 权限模型（外部模式：default, acceptEdits, bypassPermissions, plan, dontAsk；internal 另含 gated `auto` / `bubble`） | `src/types/permissions.ts#PermissionMode` |
| 9.2 | 权限规则（PermissionRule 格式 `Bash(npm install)`, 源追踪） | `src/types/permissions.ts#PermissionRule` |
| 9.3 | 权限决策流程（rules → mode → hooks → classifier → interactive dialog） | `src/utils/permissions/permissions.ts#hasPermissionsToUseTool` |
| 9.4 | ToolPermissionContext 不可变传递（DeepImmutable 包装） | `src/Tool.ts#ToolPermissionContext` |
| 9.5 | Plan 模式限制（stash pre-plan mode, dangerous rule 剥离） | `src/utils/permissions/permissionSetup.ts#prepareContextForPlanMode` |
| 9.6 | 交互式权限对话（竞赛：classifier vs 用户交互，200ms 宽限期） | `src/hooks/toolPermission/handlers/interactiveHandler.ts` |
| 9.7 | 权限持久化（用户允许 → 写 settings → 立即生效） | `src/hooks/toolPermission/PermissionContext.ts` |
| 9.8 | 设置来源完整图谱（user / project / local / flag + policy / managed / remote managed / MDM / drop-ins） | `src/utils/settings/settings.ts` |

**产出**：settings 来源合并图 + permission decision 流程图

**验证级别**：read → static

---

### Step 10: 状态管理

**核心问题**：AppState 如何在 QueryEngine、工具、UI 之间流动？

**优先文件**：
- `src/state/AppStateStore.ts`
- `src/state/store.ts`
- `src/state/AppState.tsx`
- `src/state/onChangeAppState.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 10.1 | 发布-订阅 Store（getState, setState, subscribe — Object.is 浅比较） | `src/state/store.ts` |
| 10.2 | AppState 双区结构（不可变区 DeepImmutable + 可变区） | `src/state/AppStateStore.ts` |
| 10.3 | React 集成（useAppState selector + useSyncExternalStore 并发安全） | `src/state/AppState.tsx` |
| 10.4 | 状态变更副作用（onChangeAppState — 权限模式同步、设置持久化） | `src/state/onChangeAppState.ts` |
| 10.5 | 设置热重载（文件变更检测 → applySettingsChange → AppState 更新） | `src/utils/settings/applySettingsChange.ts` |

**验证级别**：read → static

---

### Step 11: 上下文窗口管理（Compaction）

**核心问题**：token 预算如何限制循环？compaction 为什么是系统核心而不仅是优化项？

**优先文件**：`src/services/compact/`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 11.1 | Token 预算分配（effective window = context - reserved output，三级阈值） | `src/utils/context.ts#getEffectiveContextWindowSize` |
| 11.2 | Auto-compact 触发（shouldAutoCompact → 电路断路器 3 次失败保护） | `src/services/compact/autoCompact.ts` |
| 11.3 | Snip compaction（历史截断，保留结构） | `src/query.ts` |
| 11.4 | Microcompact（清除旧 tool result → `[Old tool result content cleared]`） | `src/services/compact/microCompact.ts` |
| 11.5 | Full conversation compaction（语义摘要 → CompactionResult → boundary marker） | `src/services/compact/compact.ts#compactConversation` |
| 11.6 | Partial compaction（后缀保留 — reactive/session-memory 场景） | `src/services/compact/compact.ts#partialCompactConversation` |
| 11.7 | Post-compact 恢复（文件 top-5 还原、plan 重注入、skill 重注入、async agent 状态） | `src/services/compact/compact.ts` |
| 11.8 | Session memory compaction（轻量级，在 full compact 之前尝试） | `src/services/compact/sessionMemoryCompact.ts` |

**产出**：compact 触发与恢复图 + resume 重建会话笔记

**验证级别**：read → runtime

---

### Step 12: 会话持久化

**核心问题**：transcript、resume、content replacement 如何与主循环衔接？

**优先文件**：
- `src/utils/sessionStorage.ts`
- `src/utils/toolResultStorage.ts`
- `src/utils/sessionRestore.ts`
- `src/utils/fileHistory.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 12.1 | JSONL transcript 存储（每条消息一行，parentUuid 链） | `src/utils/sessionStorage.ts` |
| 12.2 | Content replacement 系统（大 tool result → 磁盘 → preview + filepath） | `src/utils/toolResultStorage.ts` |
| 12.3 | Resume 流程（加载 JSONL → 修复 parentUuid 断链 → 恢复相同系统提示） | `src/utils/sessionRestore.ts` |
| 12.4 | 文件历史快照（FileHistorySnapshot — 追踪修改文件用于恢复） | `src/utils/fileHistory.ts` |

**验证级别**：read → runtime

---

### Step 13: Hooks 系统

**核心问题**：Hooks 如何为工具执行、权限决策、会话生命周期提供扩展点？

**优先文件**：
- `src/types/hooks.ts`
- `src/utils/hooks.ts`
- `src/utils/hooks/hooksConfigManager.ts`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 13.1 | Hook 事件分类（Session/Tool/Agent/Task/Config/UserInteraction — 15+ 事件类型） | `src/types/hooks.ts` |
| 13.2 | 同步 vs 异步 Hook 执行（sync 返回决策，async 返回 `{async: true}` + 回调） | `src/utils/hooks.ts` |
| 13.3 | Hook 配置（settings.json 中定义，shell/HTTP/callback） | `src/utils/hooks/hooksConfigManager.ts` |
| 13.4 | 权限决策 Hook（PreToolUse → approve/deny/passthrough + updatedInput） | `src/utils/hooks.ts` |
| 13.5 | Session 生命周期 Hook（start/stop/failure，1500ms 超时） | `src/utils/hooks.ts` |

**验证级别**：read → static

---

## Part IV: 高级架构（Step 14-17）

### Step 14: 多 Agent 系统

**核心问题**：子 agent、后台任务、worktree、swarm 如何叠加在主链之上？

**优先文件**：
- `src/tools/AgentTool/AgentTool.tsx`
- `src/utils/agentContext.ts`
- `src/tasks/LocalAgentTask/`
- `src/coordinator/`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 14.1 | AgentTool 设计（description, prompt, subagent_type, model, run_in_background） | `src/tools/AgentTool/AgentTool.tsx` |
| 14.2 | AsyncLocalStorage 上下文隔离（多后台 agent 同进程并发） | `src/utils/agentContext.ts` |
| 14.3 | Subagent 上下文创建（MCP 继承、工具池继承、权限克隆、独立 abortController） | `src/tools/AgentTool/runAgent.ts` |
| 14.4 | Fork Subagent（完整对话继承 + renderedSystemPrompt 缓存共享 + FORK_BOILERPLATE 防递归） | `src/tools/AgentTool/forkSubagent.ts` |
| 14.5 | 后台 Agent（支持按 env / gate 控制的 auto-background，常见默认值 120s；LocalAgentTask 状态追踪，pendingMessages 队列） | `src/tasks/LocalAgentTask/`, `src/tools/AgentTool/AgentTool.tsx` |
| 14.6 | Task 系统（LocalAgentTask vs RemoteAgentTask — 状态机、进度追踪） | `src/tasks/` |
| 14.7 | Worktree 隔离（git worktree 创建，独立工作目录） | AgentTool 相关 |
| 14.8 | Team/Swarm 协调（TeamFile 结构、subscription 模型、leader/member） | `src/utils/swarm/teamHelpers.ts` |
| 14.9 | Coordinator 模式（leader/worker 任务分配） | `src/coordinator/` |
| 14.10 | 会话发现与 Agent 路由 | `src/assistant/` |
| 14.11 | WebSocket 远程会话通信 | `src/remote/` |

**验证级别**：read → runtime（subagent 测试）

---

### Step 15: MCP 协议与扩展性

**核心问题**：MCP / skills / plugins 如何扩展工具能力？

**优先文件**：
- `src/services/mcp/client.ts`
- `src/services/mcp/config.ts`
- `src/skills/`
- `src/plugins/`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 15.1 | MCP 客户端（5 种传输：stdio, SSE, HTTP, WebSocket, SDK） | `src/services/mcp/client.ts` |
| 15.2 | 工具/资源桥接（mcp__\<server\>__\<tool\> 命名规范） | `src/services/mcp/utils.ts` |
| 15.3 | 配置发现（.mcp.json, settings.json, managed-mcp.json 合并） | `src/services/mcp/config.ts` |
| 15.4 | 连接池与动态加载（ensureConnectedClient 缓存） | `src/services/mcp/client.ts` |
| 15.5 | 技能注册系统（bundled + MCP 技能发现加载） | `src/skills/` |
| 15.6 | 插件注册与管理 | `src/plugins/` |

**验证级别**：read → runtime

---

### Step 16: 终端 UI

**核心问题**：REPL UI 如何承接主链的所有状态？

**优先文件**：
- `src/screens/REPL.tsx`
- `src/ink/`
- `src/components/`

**细分单元**：

| 单元 | 内容 | 对标 |
|------|------|------|
| 16.1 | 自定义 Ink 框架（React reconciler + Yoga 布局适配层，底层为 TS 版 yoga） | `src/ink/reconciler.ts`, `src/ink/layout/yoga.ts`, `src/native-ts/yoga-layout/index.ts` |
| 16.2 | REPL 组件（主循环、feature-gated 模块） | `src/screens/REPL.tsx` |
| 16.3 | 消息渲染（虚拟滚动、内容块分类渲染、流式 markdown） | `src/components/Messages.tsx` |
| 16.4 | PromptInput（多模式输入、自动补全、历史导航） | `src/components/PromptInput/PromptInput.tsx` |
| 16.5 | 权限对话 UI | `src/components/permissions/` |
| 16.6 | 事件系统（键盘、鼠标、焦点管理） | `src/ink/events/` |
| 16.7 | 键绑定 schema 与匹配 | `src/keybindings/` |
| 16.8 | Yoga 布局引擎 TS 实现 | `src/native-ts/` |
| 16.9 | UI 层 React Context | `src/context/` |

**验证级别**：read → runtime

---

### Step 17: 高级模式

| 单元 | 内容 | 对标 |
|------|------|------|
| 17.1 | Query chain tracking（chainId + depth — 嵌套工具调用遥测关联） | `src/Tool.ts#QueryChainTracking` |
| 17.2 | Token budget enforcement（自动继续直到预算耗尽，diminishing return 检测） | `src/query/tokenBudget.ts` |
| 17.3 | Deferred tool loading（ToolSearch — 工具数量超阈值时按需加载 schema） | `src/tools/ToolSearchTool/` |
| 17.4 | Slash 命令系统（注册、解析、生命周期追踪） | `src/commands/`, `src/utils/slashCommandParsing.ts` |
| 17.5 | Attribution 系统（commit Co-Authored-By、PR 标注） | `src/utils/attribution.ts` |
| 17.6 | Plan mode V2（多阶段结构化工作流） | `src/utils/planModeV2.ts` |

**验证级别**：read

---

## 验证策略

### Level 1: 阅读验证（read）
- 适用：入口结构、消息生命周期、系统提示、权限来源
- 方式：画图、写笔记、"能否解释清楚边界"

### Level 2: 静态验证（static）
- 适用：类型设计、配置 Schema、mini-claude 接口边界
- 方式：`tsc --noEmit`、小范围单元测试、Schema 校验

### Level 3: 运行验证（runtime）
- 适用：query loop、tool execution、streaming、compaction、resume
- 方式：`bun test`、本地演示命令、简化版集成链路

---

## 学习笔记规范

存储位置：`docs/learning/`

**frontmatter**：
```markdown
---
id: "001"
title: "简短标题"
date: "YYYY-MM-DD"
tags: [标签1, 标签2]
source_refs:
  - src/QueryEngine.ts#submitMessage
  - src/query.ts#query
phase: "Step X"
verification:
  level: "read"  # read | static | runtime
---
```

**引用规则**：
1. `source_refs` 优先 `文件#函数名` 或 `文件#类型名`
2. 仅在必须精确定位时补行号
3. `verification.level` 标记本条结论的验证级别

索引文件：`docs/learning/INDEX.md`

---

## 协作方式

1. 每次会话聚焦 1-2 个 Step 或其子单元
2. 用户可随时发散追问，所有 Q&A 原子化记录
3. 每个 Step 产出笔记 + 图（阅读阶段）或可运行代码（实现阶段）
4. 学习轨迹通过 git 版本控制
5. 主链（Step 1-6）完成后启动 mini-claude，后续 Step 边学边补充

---

## 一句话总结

Claude Code 的真实学习单元不是"类型、配置、状态"这些静态切片，而是"消息如何沿着主数据流穿过系统并被不断重写、执行、压缩和持久化"——同时不丢失每一个支撑模块的细节。
