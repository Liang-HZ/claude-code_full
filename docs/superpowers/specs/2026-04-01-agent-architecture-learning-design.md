# Agent 架构深度学习计划 — 修订版

## 目标

通过系统性研读当前仓库，并在理解主数据流之后再构建一个精简版 `mini-claude`，掌握 Claude Code 类 agent 系统的真实架构，而不是只复制其目录结构或类型表面。

本修订版基于对当前代码库的再次核实，重点修正了以下问题：

1. 学习顺序不再从孤立的类型定义起步，而是从运行主链起步。
2. 增加环境与验证前提闸门，避免在依赖未就绪时空谈“运行验证”。
3. 降低对精确行号锚点的依赖，改为优先使用文件路径 + 函数名 / 类型名。
4. 将 `mini-claude` 的镜像复刻后置，先理解数据流，再做实现。

## 修订原则

1. 主链优先：先理解 `输入 -> QueryEngine -> query -> tools -> API`。
2. 先读后造：至少走通一条完整请求路径后，再开始 `mini-claude`。
3. 锚点稳定：笔记引用优先使用 `文件 + 符号名`，避免绑定脆弱行号。
4. 区分 stub 与完整实现：遇到自动生成的 stub，只做识别，不作为核心学习材料。
5. 验证分级：阅读验证、静态验证、运行验证分开，不再假设每一步都能直接运行。
6. 以数据流组织知识，而不是以静态模块名组织知识。

## 当前仓库的主学习坐标

当前仓库更适合作为以下五层结构来理解：

1. 交互层：`src/main.tsx`、`src/screens/REPL.tsx`
2. 编排层：`src/QueryEngine.ts`
3. 核心循环层：`src/query.ts`
4. 工具层：`src/tools.ts`、`src/Tool.ts`
5. 通信层：`src/services/api/claude.ts`

学习时应始终围绕这条主链回看支撑模块，而不是先拆散为类型、配置、状态、权限等孤立章节。

## Phase 0: 环境与验证前提闸门

在进入任何“运行验证”之前，必须先通过本闸门。

### 0.1 目标

确认当前工作区具备最基本的依赖与执行条件；若未通过，则本轮学习只做源码阅读与结构分析，不做运行类验证承诺。

### 0.2 闸门清单

必须至少核实以下项目：

1. `node_modules` 已存在，或已完成 `bun install`
2. `bun run dev --help` 可执行
3. `bun test` 可执行
4. 若涉及 API 链路验证，认证与密钥前提已明确

### 0.3 闸门未通过时的规则

如果闸门未通过：

1. 允许源码阅读
2. 允许架构图与消息流梳理
3. 允许设计 `mini-claude` 的模块边界
4. 不承诺运行验证
5. 不把“跑不起来”误判为“理解错误”

## 学习笔记规范

存储位置：`docs/learning/`

每篇笔记优先记录一个可独立回答的问题，而不是一段大而全的章节总结。

建议 frontmatter：

```markdown
---
id: "001"
title: "为什么 QueryEngine 不是主循环本体"
date: "2026-04-02"
tags: [query-engine, message-lifecycle, orchestration]
source_refs:
  - src/QueryEngine.ts#submitMessage
  - src/query.ts#query
  - src/utils/processUserInput/processUserInput.ts#processUserInput
verification:
  level: "read" # read | static | runtime
---
```

说明：

1. `source_refs` 优先写 `文件#函数名` 或 `文件#类型名`
2. 只有在某一段逻辑必须精确定位时才补行号
3. `verification.level` 用于标记本条结论是阅读验证、静态验证还是运行验证

索引文件：`docs/learning/INDEX.md`

## 修订后的学习顺序

不再使用“18 个 Phase，90+ 单元”的固定切分，而采用更贴近主数据流的 10 步顺序。

### Step 1: 消息生命周期与主链边界

核心问题：

1. 用户输入是如何进入系统的？
2. 消息在进入 `query()` 之前发生了哪些变形？
3. 哪些消息给 UI 看，哪些消息给 API 看？

优先文件：

1. `src/utils/processUserInput/processUserInput.ts`
2. `src/utils/messages.ts`
3. `src/QueryEngine.ts#submitMessage`
4. `src/query.ts#query`

产出：

1. 一张“消息从输入到循环”的时序图
2. 一篇解释 `normalizeMessagesForAPI` 边界的笔记

### Step 2: 入口与会话启动

核心问题：

1. CLI 是如何启动并分流不同模式的？
2. 初始化时哪些前置工作会先发生？
3. 哪些逻辑在交互前，哪些逻辑在会话中？

优先文件：

1. `src/entrypoints/cli.tsx`
2. `src/main.tsx`
3. `src/entrypoints/init.ts`

产出：

1. 一张入口分发图
2. 一篇解释 startup fast-path 的笔记

### Step 3: QueryEngine 作为编排层

核心问题：

1. `QueryEngine` 与 `query()` 的边界是什么？
2. 为什么它不是主循环，但又是会话编排核心？
3. 它如何管理状态、转录、权限拒绝与成本统计？

优先文件：

1. `src/QueryEngine.ts`
2. `src/utils/queryContext.ts`
3. `src/utils/sessionStorage.ts`

产出：

1. 一篇解释 `submitMessage()` 职责边界的笔记
2. 一张 QueryEngine 内部状态图

### Step 4: API 客户端与流式传输

核心问题：

1. API 请求如何发出？
2. 流式事件如何累积为 assistant 消息？
3. 重试、fallback、流式错误恢复如何发生？

优先文件：

1. `src/services/api/claude.ts`
2. `src/services/api/client.ts`
3. `src/services/api/withRetry.ts`

产出：

1. 一张 streaming 事件处理图
2. 一篇解释 delta 累积与恢复策略的笔记

### Step 5: 工具系统的抽象、注册与执行

核心问题：

1. Tool 抽象的最小必要字段是什么？
2. 工具是如何被注册与过滤的？
3. 单个 tool use 从校验到结果返回会经过哪些阶段？

优先文件：

1. `src/Tool.ts`
2. `src/tools.ts`
3. `src/services/tools/toolExecution.ts`
4. `src/services/tools/toolOrchestration.ts`

建议先选三个具代表性的工具精读：

1. `FileReadTool`
2. `BashTool`
3. `FileEditTool`

产出：

1. 一张 tool execution pipeline 图
2. 三篇工具样板分析笔记

### Step 6: ReAct 核心循环与工具编排

核心问题：

1. `query.ts` 中 `while (true)` 的真正继续条件是什么？
2. 工具批次如何切分？
3. 流式输出与工具执行如何交织？

优先文件：

1. `src/query.ts`
2. `src/services/tools/StreamingToolExecutor.ts`
3. `src/services/tools/toolOrchestration.ts`

产出：

1. 一张 ReAct 循环状态图
2. 一篇解释 `needsFollowUp` 与继续条件的笔记

### Step 7: 系统提示、上下文构建与记忆

核心问题：

1. 系统提示由哪些静态与动态部分构成？
2. CLAUDE.md / memory / git status 如何注入？
3. 这些上下文是如何为主循环服务的？

优先文件：

1. `src/constants/prompts.ts`
2. `src/constants/systemPromptSections.ts`
3. `src/context.ts`
4. `src/utils/claudemd.ts`
5. `src/memdir/memdir.ts`

产出：

1. 一张 system prompt 组装图
2. 一篇解释 `getUserContext` / `getSystemContext` 的笔记

### Step 8: 权限、设置与运行边界

核心问题：

1. 权限模式与规则如何作用于工具执行？
2. 配置和设置的真实来源有哪些？
3. 企业管控、managed settings、remote managed settings 如何进入系统？

优先文件：

1. `src/utils/permissions/permissions.ts`
2. `src/utils/permissions/permissionSetup.ts`
3. `src/utils/settings/settings.ts`
4. `src/utils/config.ts`
5. `src/state/onChangeAppState.ts`

注意：

这一阶段不再把配置系统简化为“4 层配置”，而是明确区分：

1. user / project / local / flag
2. policy / managed / remote managed
3. MDM / HKCU / drop-ins

产出：

1. 一张 settings 来源合并图
2. 一张 permission decision 流程图

### Step 9: 上下文窗口管理、持久化与恢复

核心问题：

1. token 预算如何限制循环？
2. compaction 为什么是系统核心而不是优化项？
3. transcript、resume、content replacement 如何与主循环衔接？

优先文件：

1. `src/query/tokenBudget.ts`
2. `src/services/compact/`
3. `src/utils/sessionStorage.ts`
4. `src/utils/sessionRestore.ts`
5. `src/utils/toolResultStorage.ts`

产出：

1. 一张 compact 触发与恢复图
2. 一篇解释 resume 如何重建会话状态的笔记

### Step 10: 多 Agent、MCP、扩展与 UI

核心问题：

1. 子 agent、后台任务、worktree、swarm 如何叠加在主链之上？
2. MCP / skills / plugins 如何扩展工具能力？
3. REPL UI 如何承接这些状态？

优先文件：

1. `src/tools/AgentTool/`
2. `src/tasks/`
3. `src/coordinator/`
4. `src/services/mcp/`
5. `src/skills/`
6. `src/plugins/`
7. `src/screens/REPL.tsx`

产出：

1. 一张 agent / task / coordinator 关系图
2. 一张 MCP / skill / plugin 扩展关系图

## `mini-claude` 的实施策略

`mini-claude` 不再与每一步强绑定，而采用延后实施策略。

### 实施时机

只有在完成 Step 1 到 Step 4 之后，才开始实现第一版 `mini-claude`。

原因：

1. 这时已经走通主链
2. 已经知道 QueryEngine 与 query 的边界
3. 已经知道工具系统与 API 链路的最小必要抽象

### 第一版 `mini-claude` 只做什么

第一版只实现最小主链：

1. 非交互输入
2. 消息对象与规范化边界
3. 单轮 / 多轮 query loop
4. 1 到 2 个基础工具
5. 基础系统提示注入

### 第一版 `mini-claude` 暂不做什么

1. worktree
2. swarm / coordinator
3. remote sessions
4. plugin marketplace
5. 完整 compaction 家族
6. 企业级 settings / policy 系统

## 验证策略

不再统一写成“每个阶段运行验证”，而采用三级验证：

### Level 1: 阅读验证

适用于：

1. 入口结构
2. 消息生命周期
3. 系统提示构建
4. 权限与设置来源梳理

验证方式：

1. 画图
2. 写笔记
3. 用“能否解释清楚边界”来验证

### Level 2: 静态验证

适用于：

1. 类型设计
2. 配置 Schema
3. `mini-claude` 的接口边界

验证方式：

1. `tsc --noEmit`
2. 小范围单元测试
3. Schema 校验

### Level 3: 运行验证

适用于：

1. query loop
2. tool execution
3. streaming
4. compaction
5. resume

验证方式：

1. `bun test`
2. 本地演示命令
3. 简化版集成链路

## 本计划不再做的承诺

以下承诺已被移除：

1. 不再承诺每一步都有稳定的精确行号锚点
2. 不再承诺每一步都能直接运行验证
3. 不再承诺从 Phase 1 就同步镜像复刻目录结构
4. 不再把自动生成 stub 当作核心学习入口

## 一句话总结

Claude Code 的真实学习单元不是“类型、配置、状态”这些静态切片，而是“消息如何沿着主数据流穿过系统并被不断重写、执行、压缩和持久化”。
