# Agent 架构深度学习计划 — 设计文档

## 愿景

通过系统性研读 Claude Code 源码并从零构建一个对标其架构的 TypeScript agent（项目名 `mini-claude`），深度掌握业界顶级 agent 系统的核心设计模式，形成可迁移到任意语言/场景的 agent 架构能力。

## 核心决策

| 维度 | 决策 | 理由 |
|------|------|------|
| 学习方式 | 边学边造（镜像还原法） | 每个模块先读源码、再实现精简版，理解与实践严格对齐 |
| 实现语言 | TypeScript (Bun) | 直接对标源码，零翻译损耗；掌握模式后可迁移到任意语言 |
| 学习节奏 | 自底向上，模块化递进 | 每一层都是下一层的基石，最终拼成完整系统 |
| 知识沉淀 | 原子化 Q&A 笔记（Zettelkasten） | 每条记录独立，支持发散式追问，后续可按需整理 |
| 产出形态 | 可运行的 agent + 学习轨迹文档 | 实践出真知，对话过程本身也是知识资产 |

## 学习笔记规范

存储位置：`docs/learning/`

每个 Q&A 文件格式：
```markdown
---
id: "001"
title: "简短标题"
date: "2026-04-01"
tags: [react-loop, streaming, tool-system]
source_files:
  - src/query.ts:307
  - src/Tool.ts:158
---

## 问题
用户的原始问题

## 分析
深入的源码分析，包含代码片段

## 解答
结论性的回答

## 关联
相关笔记的链接
```

索引文件：`docs/learning/INDEX.md`，按时间线排列，标注标签。

## 学习阶段总览

共 18 个 Phase，90+ 个学习单元。每个阶段结构：
1. **源码研读** — 一起读 Claude Code 对应模块的核心代码
2. **提问与深入** — 任何问题即时记录为原子笔记
3. **动手实现** — 在 `mini-claude` 中实现该模块的精简版
4. **验证** — 运行验证，确认理解正确

---

## Phase 1: 项目骨架与基础类型系统

**对标源码**: `cli.tsx` polyfill/MACRO, `types/message.ts`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 1.1 | Bun + ESM + TS 项目搭建 | `cli.tsx`, `tsconfig.json`, `package.json` |
| 1.2 | 消息类型体系（7 种 MessageType） | `types/message.ts:19` |
| 1.3 | 内容块类型（text, tool_use, tool_result, thinking） | `types/message.ts:33-49` |

## Phase 2: 配置与初始化体系

**对标源码**: `utils/config.ts`, `bootstrap/state.ts`, `entrypoints/init.ts`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 2.1 | 4 层配置系统（global → project → local → flag） | `utils/config.ts:76-148` |
| 2.2 | 设置文件发现与合并 | `utils/settings/settings.ts:812-868` |
| 2.3 | Bootstrap 状态单例 | `bootstrap/state.ts` |
| 2.4 | 初始化编排（并行预取、依赖排序） | `entrypoints/init.ts:57-210` |

## Phase 3: API 客户端与流式传输

**对标源码**: `services/api/claude.ts`, `services/api/client.ts`, `services/api/withRetry.ts`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 3.1 | Anthropic SDK 集成 | `services/api/client.ts:88-250` |
| 3.2 | 流式事件处理（5 种事件类型的完整链） | `services/api/claude.ts:1941-2170` |
| 3.3 | 部分消息累积（delta 拼接） | `services/api/claude.ts:2054-2170` |
| 3.4 | 重试与错误恢复（529/429/401） | `services/api/withRetry.ts:50-250` |
| 3.5 | 流式空闲超时与停滞检测 | `services/api/claude.ts:1901-1930` |
| 3.6 | Provider 抽象层 | `utils/model/providers.ts:6-14` |

## Phase 4: 最小对话循环

**对标源码**: `QueryEngine.ts:submitMessage()`, `query.ts` 基础循环

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 4.1 | 单轮对话 | `QueryEngine.ts:211-598` |
| 4.2 | 多轮对话（历史累积） | `utils/messages.ts` |
| 4.3 | Token 计数与估算 | `utils/tokens.ts` |
| 4.4 | 消息规范化边界（UI vs API） | `utils/messages.ts:normalizeMessagesForAPI` |

## Phase 5: 工具系统 — 类型与注册

**对标源码**: `Tool.ts`, `tools.ts`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 5.1 | Tool 接口设计（30+ 字段） | `Tool.ts:362-695` |
| 5.2 | ToolUseContext 执行环境 | `Tool.ts:158-300` |
| 5.3 | buildTool() 工厂函数 | `Tool.ts:783-792` |
| 5.4 | Zod Schema 验证 | `tools/FileReadTool/FileReadTool.ts` |
| 5.5 | 工具注册表与过滤 | `tools.ts:193-327` |

## Phase 6: 工具系统 — 执行管线

**对标源码**: `services/tools/toolExecution.ts`, `services/tools/toolOrchestration.ts`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 6.1 | 完整执行管线（validate → hook → permission → call → result） | `services/tools/toolExecution.ts:337-1350` |
| 6.2 | 实现 FileReadTool | `tools/FileReadTool/` |
| 6.3 | 实现 BashTool | `tools/BashTool/` |
| 6.4 | 实现 FileEditTool | `tools/FileEditTool/` |
| 6.5 | 工具结果格式化与持久化 | `utils/toolResultStorage.ts` |
| 6.6 | 工具 Prompt 注入 | `constants/prompts.ts:444-500` |

## Phase 7: ReAct 核心循环

**对标源码**: `query.ts:307` 的 while(true)

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 7.1 | while(true) + needsFollowUp 模式 | `query.ts:307-1360` |
| 7.2 | 循环状态结构 | `query.ts:241-304` |
| 7.3 | 并发安全分区（partitionToolCalls） | `services/tools/toolOrchestration.ts:91-116` |
| 7.4 | runTools() 批次执行 | `services/tools/toolOrchestration.ts:19-82` |
| 7.5 | StreamingToolExecutor | `services/tools/StreamingToolExecutor.ts` |
| 7.6 | 进度报告 | `services/tools/toolExecution.ts:onProgress` |

## Phase 8: 错误恢复与弹性

**对标源码**: `query.ts` 的 6 种恢复机制

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 8.1 | Prompt-too-long 恢复 | `query.ts:1068-1186` |
| 8.2 | Max-output-tokens 恢复 | `query.ts:1188-1259` |
| 8.3 | 模型降级（fallback） | `query.ts:896-957` |
| 8.4 | 流式中断处理 | `query.ts:1014-1055` |
| 8.5 | Stop Hooks | `query.ts:1270-1309` |

## Phase 9: 系统提示与上下文构建

**对标源码**: `constants/prompts.ts`, `utils/claudemd.ts`, `memdir/`, `outputStyles/`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 9.1 | 系统提示组合架构（static + dynamic boundary） | `constants/prompts.ts:444-577` |
| 9.2 | systemPromptSection 缓存机制 | `constants/systemPromptSections.ts` |
| 9.3 | CLAUDE.md 发现层次（4 级 + @include） | `utils/claudemd.ts` |
| 9.4 | Memory 目录系统 | `memdir/memdir.ts` |
| 9.5 | 环境上下文注入 | `context.ts:116-189` |
| 9.6 | 有效系统提示构建 | `utils/systemPrompt.ts:41-123` |
| 9.7 | 输出样式模板系统 | `outputStyles/` |

## Phase 10: 权限系统

**对标源码**: `types/permissions.ts`, `hooks/useCanUseTool.tsx`, `utils/permissions/`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 10.1 | 权限模型设计（5 种模式） | `types/permissions.ts:15-38` |
| 10.2 | 权限规则系统 | `types/permissions.ts:50-79` |
| 10.3 | 权限决策流程 | `utils/permissions/permissions.ts:122-200` |
| 10.4 | ToolPermissionContext 不可变传递 | `Tool.ts:123-148` |
| 10.5 | Plan 模式限制 | `utils/permissions/permissionSetup.ts:1458-1510` |
| 10.6 | 交互式权限对话 | `hooks/toolPermission/handlers/interactiveHandler.ts` |
| 10.7 | 权限持久化 | `hooks/toolPermission/PermissionContext.ts:139-146` |

## Phase 11: 状态管理

**对标源码**: `state/AppStateStore.ts`, `state/store.ts`, `state/AppState.tsx`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 11.1 | 发布-订阅 Store | `state/store.ts:1-35` |
| 11.2 | AppState 双区结构 | `state/AppStateStore.ts:89-452` |
| 11.3 | React 集成（useSyncExternalStore） | `state/AppState.tsx:142-163` |
| 11.4 | 状态变更副作用 | `state/onChangeAppState.ts:43-171` |
| 11.5 | 设置热重载 | `utils/settings/applySettingsChange.ts` |

## Phase 12: 上下文窗口管理（Compaction）

**对标源码**: `services/compact/`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 12.1 | Token 预算分配 | `utils/context.ts:8-222` |
| 12.2 | Auto-compact 触发 | `services/compact/autoCompact.ts:72-351` |
| 12.3 | Snip compaction | `query.ts:396-410` |
| 12.4 | Microcompact | `services/compact/microCompact.ts` |
| 12.5 | Full conversation compaction | `services/compact/compact.ts:389` |
| 12.6 | Partial compaction | `services/compact/compact.ts:774` |
| 12.7 | Post-compact 恢复 | `services/compact/compact.ts:1418-1571` |
| 12.8 | Session memory compaction | `services/compact/sessionMemoryCompact.ts` |

## Phase 13: 会话持久化

**对标源码**: `utils/sessionStorage.ts`, `utils/toolResultStorage.ts`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 13.1 | JSONL transcript 存储 | `utils/sessionStorage.ts` |
| 13.2 | Content replacement 系统 | `utils/toolResultStorage.ts` |
| 13.3 | Resume 流程 | `utils/sessionRestore.ts` |
| 13.4 | 文件历史快照 | `utils/fileHistory.ts:40-290` |

## Phase 14: Hooks 系统

**对标源码**: `types/hooks.ts`, `utils/hooks.ts`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 14.1 | Hook 事件分类（15+ 事件类型） | `types/hooks.ts` |
| 14.2 | 同步 vs 异步 Hook | `utils/hooks.ts` |
| 14.3 | Hook 配置 | `utils/hooks/hooksConfigManager.ts` |
| 14.4 | 权限决策 Hook | `utils/hooks.ts:PreToolUse` |
| 14.5 | Session 生命周期 Hook | `utils/hooks.ts:SessionStart/End` |

## Phase 15: 多 Agent 系统

**对标源码**: `tools/AgentTool/`, `utils/agentContext.ts`, `tasks/`, `coordinator/`, `assistant/`, `remote/`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 15.1 | AgentTool 设计 | `tools/AgentTool/AgentTool.tsx` |
| 15.2 | AsyncLocalStorage 隔离 | `utils/agentContext.ts:1-179` |
| 15.3 | Subagent 上下文创建 | `tools/AgentTool/runAgent.ts` |
| 15.4 | Fork Subagent | `tools/AgentTool/forkSubagent.ts` |
| 15.5 | 后台 Agent | `tasks/LocalAgentTask/` |
| 15.6 | Task 系统 | `tasks/LocalAgentTask/LocalAgentTask.tsx` |
| 15.7 | Worktree 隔离 | git worktree 相关 |
| 15.8 | Team/Swarm 协调 | `utils/swarm/teamHelpers.ts` |
| 15.9 | Coordinator 模式（leader/worker 任务分配） | `coordinator/` |
| 15.10 | 会话发现与 Agent 路由 | `assistant/` |
| 15.11 | WebSocket 远程会话通信 | `remote/` |

## Phase 16: MCP 协议与扩展性

**对标源码**: `services/mcp/`, `skills/`, `plugins/`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 16.1 | MCP 客户端（5 种传输） | `services/mcp/client.ts` |
| 16.2 | 工具/资源桥接 | `services/mcp/utils.ts:32-150` |
| 16.3 | 配置发现 | `services/mcp/config.ts` |
| 16.4 | 连接池与动态加载 | `services/mcp/client.ts:ensureConnectedClient` |
| 16.5 | 技能注册系统（bundled + MCP 技能发现） | `skills/` |
| 16.6 | 插件注册与管理 | `plugins/` |

## Phase 17: 终端 UI

**对标源码**: `screens/REPL.tsx`, `ink/`, `components/`, `keybindings/`, `native-ts/`, `context/`

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 17.1 | 自定义 Ink 框架 | `ink/reconciler.ts` |
| 17.2 | REPL 组件 | `screens/REPL.tsx` |
| 17.3 | 消息渲染（虚拟滚动） | `components/Messages.tsx` |
| 17.4 | PromptInput | `components/PromptInput/PromptInput.tsx` |
| 17.5 | 权限对话 UI | `components/permissions/` |
| 17.6 | 事件系统 | `ink/events/` |
| 17.7 | 键绑定 schema 与匹配 | `keybindings/` |
| 17.8 | Yoga 布局引擎 TS 实现 | `native-ts/` |
| 17.9 | UI 层 React Context（prompt overlays, metrics） | `context/` |

## Phase 18: 高级模式

| 单元 | 内容 | 对标文件 |
|------|------|----------|
| 18.1 | Query chain tracking | `Tool.ts:90-93` |
| 18.2 | Token budget enforcement | `query/tokenBudget.ts` |
| 18.3 | Deferred tool loading (ToolSearch) | `tools/ToolSearchTool/` |
| 18.4 | Slash 命令系统 | `commands.ts`, `utils/slashCommandParsing.ts` |
| 18.5 | Attribution 系统 | `utils/attribution.ts` |
| 18.6 | Plan mode V2 | `utils/planModeV2.ts` |

---

## 依赖关系图

```
Phase 1 (类型) ─→ Phase 2 (配置) ─→ Phase 3 (API) ─→ Phase 4 (对话)
                                                              ↓
Phase 5 (工具类型) ─→ Phase 6 (工具执行) ─→ Phase 7 (ReAct) ─→ Phase 8 (恢复)
                                                              ↓
Phase 9 (上下文) ─→ Phase 10 (权限) ─→ Phase 11 (状态) ─→ Phase 12 (压缩)
                                                              ↓
Phase 13 (持久化) ─→ Phase 14 (Hooks) ─→ Phase 15 (多Agent) ─→ Phase 16 (MCP)
                                                              ↓
                                          Phase 17 (UI) ─→ Phase 18 (高级)
```

## 项目结构

```
claude-code_full/                    # 当前仓库（Claude Code 源码参考）
├── docs/learning/                   # 原子化学习笔记
│   ├── INDEX.md                     # 笔记索引
│   ├── 001-xxx.md                   # 按序号命名的 Q&A 笔记
│   └── ...
├── docs/superpowers/specs/          # 设计文档
│   └── 2026-04-01-agent-architecture-learning-design.md  # 本文件
└── mini-claude/                     # 实践产出的 agent 项目
    ├── package.json
    ├── tsconfig.json
    ├── src/
    │   ├── types/                   # Phase 1: 消息类型
    │   ├── config/                  # Phase 2: 配置系统
    │   ├── api/                     # Phase 3: API 客户端
    │   ├── engine/                  # Phase 4+7: 对话循环 & ReAct
    │   ├── tools/                   # Phase 5+6: 工具系统
    │   ├── context/                 # Phase 9: 上下文构建
    │   ├── permissions/             # Phase 10: 权限系统
    │   ├── state/                   # Phase 11: 状态管理
    │   ├── compact/                 # Phase 12: Compaction
    │   ├── persistence/             # Phase 13: 会话持久化
    │   ├── hooks/                   # Phase 14: Hooks 系统
    │   ├── agents/                  # Phase 15: 多 Agent
    │   ├── mcp/                     # Phase 16: MCP + 扩展性
    │   ├── skills/                  # Phase 16: 技能系统
    │   ├── plugins/                 # Phase 16: 插件系统
    │   └── ui/                      # Phase 17: 终端 UI
    └── tests/                       # 各模块测试
```

## 协作方式

1. 每次会话聚焦 1-2 个学习单元
2. 用户可随时发散追问，所有 Q&A 原子化记录
3. 每个单元产出可运行代码 + 对应笔记
4. 学习轨迹通过 git 版本控制
5. 每个 Phase 完成后回顾整合
