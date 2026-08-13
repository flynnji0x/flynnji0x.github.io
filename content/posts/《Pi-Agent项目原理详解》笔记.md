+++
title = "《Pi-Agent 项目原理详解》笔记"
date = '2026-07-29'
tags = ["ai-agent", "LLM", "智能体", "typescript", "源码解读"]
categories = ["读书笔记"]
+++

> 资料来源：`dg-ai-notes/pi-agent/docs/typescript`

## 总览：Pi-Agent 是什么

Pi 的核心哲学是**减法**："What we didn't build"——不做 MCP（13,700+ token 工具描述开销太大）、不做子 Agent（降低可观察性）、不做权限弹窗（弹窗疲劳沦为安全表演）、不做计划模式（plan.md 文件更持久）…… 在一个正朝着"全包"狂奔的赛道里，做减法本身就是一种有竞争力的产品立场。

---

## 第1章：开篇 —— Pi-Agent 框架总览

回答"Pi 是什么、为什么值得学"。三个身份对应三种读者：想用好工具的人、想学 Agent 设计的人、想用 SDK 二次开发的人。

**关键数字**：

| 指标 | 数值 |
| ------ | ------ |
| GitHub Stars | 64,000+ |
| 内置工具 | 4 核心（read/write/edit/bash）+ 3 辅助（grep/find/ls） |
| 系统提示词 | 静态模板 ~90 词，对比 Claude Code 数万字 |
| 支持供应商 | 30+ 家（`KnownProvider` 枚举实际 35 个） |
| 核心包 | pi-ai / pi-agent-core / pi-tui / pi-coding-agent |
| 运行模式 | 交互 / print-JSON / RPC / SDK |

**三个视角的要点**：

- **编码工具视角**：上下文干净（系统提示词+工具定义 < 1,000 token）、透明到骨头（能看到模型收到的每条消息、每个工具调用、完整成本追踪）、模型自由（会话中途用 `/model` 切换）、树状会话（`/tree` 分叉，调试时可在同一起点试多种方案）、默认 YOLO 模式（无审批弹窗）
- **学习素材视角**：TerminalBench 基准排名第二（仅次于 Terminus），核心循环只有几百行；10 章教程中前 6 章建立核心理解、后 4 章进阶工程议题，每章回答"是什么/怎么做/为什么"
- **SDK 视角**：`pi-ai` 只管调模型、`pi-agent-core` 只管跑循环、`pi-coding-agent` 是完整 CLI + SDK（`createAgentSession()` 无头模式）；扩展系统支持热重载——可以让编码 Agent 修改和增强自己的能力

还介绍了用 `~/.pi/agent/models.json` 配置国内第三方模型（智谱、DeepSeek、Kimi 等）的方法：`providers` 字典 + `api` 协议（`openai-completions` / `anthropic-messages` / `openai-responses`）+ `baseUrl` + `models` 列表。

---

## 第2章：三层架构 —— Pi-Agent 项目的骨骼

**五个包，各管各的**：

| 包 | 职责 | 不做什么 |
| ---- | ------ | --------- |
| pi-ai | 调模型（统一 API、流式调用、30+ 提供商适配） | 没有 agent / tool / loop 概念 |
| pi-agent-core | 跑循环（状态、事件、会话、压缩） | 不知道 read / bash 是什么 |
| pi-coding-agent | 具体业务（7 个编程工具、扩展、CLI、持久化） | — |
| pi-tui | 显示（终端 UI 库，只依赖 marked + get-east-asian-width） | 与 AI 体系完全解耦 |
| pi-orchestrator | 多 Agent 编排（实验性，站在 coding-agent 之上） | 不实现内核逻辑 |

**分层的真正规则不是"限制引用层级"，而是"依赖方向必须单向向上"**：底层永远不知道上层的存在——coding-agent 可以直接依赖 pi-ai（因为 `ImageContent` 等原子类型必须统一定义），但 pi-ai 里没有任何一个 import 指向上层。

**类型在层间流转（原子→分子→材料）**：

```
Tool（pi-ai）         → 名片：name / description / parameters
AgentTool（agent-core）→ + 执行：label / execute / executionMode / prepareArguments
ToolDefinition（coding-agent）→ + 展示：promptSnippet / renderCall / ExtensionContext
```

每一层只加自己层级需要的能力，底层类型从不修改。

**三个可迁移的方法**：①"依赖漏斗"分层法（去掉上层这一层还能跑吗）；②"类型递进扩展"模式（底层定义最小接口，上层用联合类型 `|` 和继承扩展）；③"可独立使用"测试（在 package.json 里暂时移除上层依赖验证）。

---

## 第3章：Agent Loop —— 让模型转动起来的引擎（核心）

**大模型的三种用法**：直接调用（一次问答）、Workflow（步骤由你的代码控制流转）、**Agent Loop**（步骤流转由模型输出内容驱动，决策权交给模型）。

**Trace vs Turn**：Trace 是从 `agent_start` 到 `agent_end` 的一次完整运行；Turn 是**一次模型调用 + 这次调用触发的所有工具执行**（一个 Turn 只有一次模型调用）。

**stopReason —— 唯一的信号灯**，五种值来自两个地方：

| 来源 | 值 | 含义 |
|------|----|------|
| 模型 API 返回 | toolUse / stop / length | 要调工具 / 自然终止 / 被截断 |
| 框架流式层注入 | error / aborted | 调用异常 / 用户中止 |

**一条规则驱动整个循环**：不是模型在说"我完成了"，而是"你没要工具，那就当你完成了"。实际驱动循环的是 `toolCalls 数组长度 > 0 && !terminate`（而不是 stopReason === toolUse）。退出路径：正常退出（stop/length）、硬停止（error/aborted，不检查 followUp）、外部钩子停（shouldStopAfterTurn）、工具终止（一批工具全部 `terminate: true`，是 every 不是 some）。

**最简 Loop 只要十几行**：`while(true){ 调模型; 无工具调用则返回; 执行工具; 结果喂回去 }`。coding-agent 在此基础上叠加了四层：

- **steering**（叠加1）：用户在 Agent 工作时输入的新指令，可在 Turn 之间"紧急插队"
- **followUp**（叠加2）：外层循环，Agent 完成后系统可追加任务（"顺便跑个测试"），在同一个 Trace 内续命
- **prepareNextTurn**：每个 turn_end 后可按任务复杂度动态切换模型 / context / thinkingLevel
- **shouldStopAfterTurn**：产品层安全阀（上下文快满、达最大 Turn 数）

**流式响应原地替换**：先 push 一个"空壳"消息，随 text_delta / toolcall_delta 用 `context.messages[last] = partialMessage` 原地覆盖，最终用完整消息替换——这样 UI 能逐字渲染，且 context 消息数量不变。

**Prompt Cache**：Anthropic 在三个位置打 `cache_control` 标记——system 末尾、**最后一个 tool**、**最后一条 user message（rolling cache，跟着最新消息走）**。每圈重建 llmContext 不破坏 cache，因为缓存是内容寻址的。OpenAI 体系走 `prompt_cache_key: sessionId`。

**四条核心设计**：ReAct 循环模式、stopReason 驱动机制、内核+叠加架构（剥掉任何一层里层仍能跑）、决策外包给模型输出模式。

---

## 第4章：模型调用 —— 一行代码驾驭多个模型（核心）

`streamSimple(model, context)` 一行代码背后，是 AI 抽象层的三层架构。

**问题**：同一段对话在不同 Provider 下要"翻译"成不同格式，差异有四个维度——消息格式（`content[]` vs `parts[]`）、流式传输（原始 SSE vs 结构化 chunk）、思考模式（`thinking.budget_tokens` vs `reasoning_effort`）、缓存控制（`cache_control` 标记 vs `cachePoint` 节点）。

**三层解法**（类比国际翻译公司）：

1. **统一入口**（前台）：`stream()` 只做"查表 + 派活"——拿 `model.api` 去注册表找翻译器
2. **事件协议**（标准报告）：12 种 `AssistantMessageEvent`——文字/思考/工具调用三类内容各有"开始→逐字增量→结束"三步，加上 start / done / error。每个事件携带 `partial: AssistantMessage` 完整快照（配合第3章的原地替换）
3. **翻译器**（翻译员）：每个翻译器遵循 5 步骨架（创建客户端 → 构建请求 → 发送 → 处理响应流 → 发终止事件），核心工作量在第 2、4 步

**StreamFunction 是"宪法"**——统一输入（model/context/options）、统一输出（AssistantMessageEventStream）、错误不抛异常（发 `{ type: "error" }` 事件）。

**接入新模型只需三步**：写翻译器 → `registerApiProvider()` 注册 → 配置 Model 信息。Agent Loop 一行不用改。

**进阶**：

- **ThinkingLevel 五级刻度**（off/minimal/low/medium/high/xhigh）：上层说 `reasoning: "high"`，翻译器查 `thinkingLevelMap` 翻译成各家参数；`clampThinkingLevel()` 回退策略先向上找再向下
- **缓存控制**：`cacheRetention: "none" | "short" | "long"` 语义接口，四家四种协议设计——Anthropic 贴便签（cache_control）、Bedrock 插路标（独立 cachePoint 节点）、OpenAI 按会员卡号查记录（prompt_cache_key）、兼容厂商直接抄便签协议。缓存命中按读取价计费，通常省 ~90% 输入成本
- **错误处理**：`stopReason: "error"/"aborted"` 就是翻译器 catch 块注入的；`isContextOverflow()` 三重检测上下文溢出

**三条设计精华**：①"协议 > 实现"（用事件协议+函数签名，不用继承抽象类）；②"统一枚举 + 映射表"（上层说 high，底层自己查表）；③"语义统一，实现分散"（cacheRetention 描述"做什么"，不管"怎么做"）。

---

## 第5章：工具系统 —— Agent 的手脚是怎么被管住的（核心）

**三层类型定义"一个工具"**（为什么不能塞进一个接口？因为每层有独立的依赖范围）：

- `Tool`（pi-ai）：名片，只管怎么告诉模型——name / description / parameters
- `AgentTool`（agent-core）：+ label / execute / executionMode / prepareArguments
- `ToolDefinition`（coding-agent）：+ promptSnippet / renderCall 等 UI 能力，execute 多一个 `ctx: ExtensionContext` 参数
- `wrapToolDefinition()` 十几行包装器通过闭包注入 ctx，Agent Loop 永远不知道 ExtensionContext 的存在

**五步管道**（模型说"读文件"到结果回模型面前）：

```
① prepareArguments（兼容垫片：把模型序列化成字符串的数组解析回来）
② validateToolArguments（TypeBox Schema 运行时类型检查）
③ beforeToolCall（权限拦截，返回 { block: true, reason } 可阻止执行）
④ tool.execute（真正干活，支持 onUpdate 流式进度回调）
⑤ afterToolCall（后置钩子：脱敏/审计/修错/早停，字段级覆盖）
   ↓
ToolResultMessage（所有错误的统一出口）
```

**并行 vs 串行——"一票否决"**：只要有一个工具声明 `executionMode: "sequential"`，整批串行（判断哪些工具会冲突太难，宁可多等不可出错）。并行是三阶段设计：**准备顺序执行**（验证和权限检查不能并行）、**只有 execute 并行**（Promise.all）、**事件有序**（result 按调用顺序发，保证 LLM 看到的上下文顺序正确）。内置工具默认全并行，Edit 靠内部 `withFileMutationQueue` 对同一文件编辑串行化兜底。

**永不抛出：工具出错也是一条消息**。6 种错误（工具未找到 / 参数预处理异常 / Schema 验证失败 / 前置拦截 / execute 抛异常 / afterToolCall 抛异常）→ 1 种产物：`isError: true` 的 ToolResultMessage。关键代码 `executePreparedToolCall` 的双重防护：catch 住异常 → `createErrorToolResult()` 翻译成正常结果 → **先等所有进度事件发完再编码错误**（防乱序）。还有 `acceptingUpdates` 闸门丢弃工具 settle 后的孤儿回调。

**核心哲学：错误信息是给模型的反馈，不是给框架的终止信号。** 错误描述越具体，模型纠错能力越强（"文件不存在，目录下有 [b.ts, c.ts]" 远好于 "Read failed"）。真实做法是两层错误处理分层负责：工具内部主动识别已知错误并重新包装（Bash 工具把"已输出的内容 + 具体原因"打包进 Error），框架兜底 catch 只做"搬运工"原样透传 error.message。

**Operations 抽象**：工具不直接调 `fs` / `child_process`，而是调最小化接口（ReadOperations / BashOperations / …），测试可 Mock、远程可 SSH，工具代码一行不用改。

**四个方法论**：分层接口递进法、管道+钩子模式、错误即消息原则、Operations 抽象法。

---

## 第6章：消息系统 —— Agent 的记忆如何组织与传递（核心）

**矛盾**：LLM 只认识三种消息（UserMessage / AssistantMessage / ToolResultMessage），但 Agent 内部还有大量功能性数据（Bash 执行记录、压缩摘要、分支记录……）需要结构化字段给 UI 和持久化用。两个读者需求冲突。

**解法：两层消息，内富外严**。

```
AgentMessage（Agent 内部）＝ Message（3 种标准，直接发 LLM）
                          ＋ CustomAgentMessages（自定义扩展）
```

- `CustomAgentMessages` 在核心包里**默认为空**，应用通过 TypeScript **声明合并**（`declare module`）注入自己的消息类型（coding-agent 注册了 bashExecution / custom / branchSummary / compactionSummary 4 种）——核心包零依赖，应用包全栈类型安全
- **自定义消息带来三个能力**：UI 专用渲染（按 role 分派）、持久化恢复（session 文件存结构化数据）、精细化可见性控制

**convertToLlm 翻译边界**：在调用 LLM 的最后一刻做**有损、单向**的翻译——所有自定义消息都变成 `user` 角色的消息（因为 LLM 要求 user/assistant 交替，自定义消息本质是"系统注入的信息"，放 user 最安全）。BashExecutionMessage 的 `command/output/exitCode` 被格式化成一段文本。

**两阶段管道为什么分开**：`transformContext`（同层变换，`AgentMessage[] → AgentMessage[]`，管"怎么裁剪"）和 `convertToLlm`（跨层翻译，类型变了，管"怎么翻译成标准格式"）职责分离，可独立替换。**换 LLM 提供商不需要动 convertToLlm**——那是 pi-ai 层翻译器的事，消息类型翻译和 Provider 协议翻译是两层独立的抽象。

**excludeFromContext 过滤**：`!!` 前缀的 Bash 命令对 LLM 隐身但 UI 照常渲染。三种可见性级别：全可见 / LLM 不可见（excludeFromContext）/ 仅持久化。

**主线**：数据结构要同时照顾两个读者——模型（协议强制，不能改）和功能层（结构化字段，能力来源）。做法：以内层结构化为"源"、外层翻译为"流"，不要在协议边界提前拍扁数据。

---

## 第7章：事件驱动 —— Agent 的神经系统（进阶）

**为什么需要事件**：把"发生了什么"和"谁关心什么"彻底分离。给 Agent 加日志/存储/渲染，只需 `session.subscribe()`，不碰 Agent 一行源码。

**10 种事件，4 层嵌套**：Agent（agent_start/end）→ Turn（turn_start/end）→ Message（message_start/update/end）→ Tool Execution（tool_execution_start/update/end）。每层都是"开始→更新×N→结束"配对。不同消费者关注不同粒度：TUI 看 message_update 逐字渲染，Session 管理器只看 turn_end。

**emit 不是"通知"，是"同步屏障"**：每次 `await emit(...)` 都等所有监听器处理完才走下一步（`processEvents` 先更新内部状态、再逐一 await 所有 listener）。保证状态一致性，代价是性能。

**例外：tool_execution_update 先收集后批量等待**——高频低价值事件用 `updateEvents.push(emit(...))` 收集，工具结束后 `Promise.all` 一次性等完；`acceptingUpdates` 闸门丢弃迟到回调。

**错误处理：监听器异常直接冒泡**（不 try-catch）——监听器出错运行就停，问题立刻可见（保险丝哲学）。实践建议：自己写的 listener 务必内部 try-catch；第三方扩展由框架隔离。

**应用场景**：实时观测 Agent、工具调用拦截（beforeToolCall 的实现机制）、上下文预处理（transformContext 钩子）、流式转发 Web（SSE 推给浏览器）。

**text_delta 的完整旅程**：LLM SSE → AI 层 EventStream.push → Agent Loop 转成 message_update（原始事件通过 assistantMessageEvent 字段透传）→ Agent.processEvents 同步屏障 → Session 分发 → TUI 渲染。5 层各管各的，唯一通信协议就是"事件"。

**两层事件**：Agent 内核 10 种 + Session 层扩展 7 种（queue_update / compaction_start/end / auto_retry_start/end 等）。判断标准：去掉某个事件后内核还能正常运行，它就属于外层。

**三个设计决策**：同步屏障、异常直接暴露、两层事件。

---

## 第8章：上下文工程 —— 让有限窗口装下无限对话（进阶）

**问题**：窗口是硬上限，对话无限增长。跑一次 `npm install` 的 stderr 十几 KB，read 一个 5000 行文件 80KB，几十轮对话轻松破 100K token。

**两层防护 4 种互补技巧**：

**① 工具输出截断**（输入侧·减法，每次工具调用都触发）

- 双重限制：`DEFAULT_MAX_LINES = 2000` 行 + `DEFAULT_MAX_BYTES = 50KB`，**先触者胜**（行数管可读性，字节管硬体积，互相兜底）
- 双向策略：`truncateHead` 保留开头（read 文件，import/接口最密）、`truncateTail` 保留末尾（bash 输出，错误堆栈在末尾）
- 边界安全：逐字符累加字节数，代理对当不可分割整体；单行超限时取末尾 maxBytes 并设 `lastLinePartial` 标志；grep 单行限长 500 字符
- 逃生通道：截断后追加 `[Showing lines 6501-8500 of 8500. Full output: /tmp/xxx.log]`——提示会进 LLM 上下文，模型想看完整输出就自己 read

**② 系统提示词动态组装**（输入侧·加法，每轮 prompt 触发）

- 多级上下文文件：从 cwd 向上递归找 `AGENTS.md` / `CLAUDE.md`，合并顺序为 agentDir 全局 → 祖先目录 → 当前项目（最具体覆盖在前），适配 monorepo 嵌套
- XML 包装：`<project_instructions path="...">` 边界明确、带 path 属性让 LLM 区分优先级
- **Skills 懒加载（拉模式）**：只放轻量清单（name/description/location，~500 token），LLM 按需调 read 工具加载全文（对比全文塞入 50K token）——**用工具调用做按需上下文加载**是核心范式

**③ Compaction**（历史侧，联动第9章）：阈值触发，旧消息变结构化摘要。

**④ 分支摘要**（历史侧，用户切换分支时）：用户从会话树分叉后，旧分支的探索成果会丢失。解法：`collectEntriesForBranchSummary` 用 **LCA 算法**找两个叶子的分叉点，收集被放弃分支 → 复用 Compaction 的底层管道（convertToLlm / serializeConversation / SUMMARIZATION_SYSTEM_PROMPT）生成摘要。差异：**只有 5 section**（无 Critical Context）、maxTokens 固定 2048（辅助上下文不能喧宾夺主）、前言注明 "The user explored a different conversation branch before returning here"。产物是 `BranchSummaryMessage`，注入新分支上下文开头。

**设计精华**：

1. **多层防护，没有银弹**——每层只解决自己擅长的问题，互不替代
2. **加法 + 减法（塑形）**——上下文工程不是单纯压缩，同等 token 下结构化的信息密度更高
3. **工具调用 = 按需上下文加载**——推模式（全塞）vs 拉模式（清单 + 按需 read），Claude Code 的 Skills、Cursor 的 docs 本质都是这套机制

---

## 第9章：上下文压缩 —— 当对话太长怎么办（进阶）

**关键时序**：压缩**发生在两轮对话之间**——agent_end 事件后检查是否超阈值，超了立刻压缩，下一轮用户开始时通过 `buildSessionContext()` 重建上下文。

**触发条件**：`shouldCompact`：`contextTokens > contextWindow - reserveTokens`（如 200K - 16K = 183,616）。reserveTokens 给 LLM 回复留空间。

**Token 估算**：`chars / 4` 启发式，不精确但够用。注意**中文会严重低估**（1 汉字实际 1-2 token，只算 0.25），纯中文场景可能到压缩阈值时实际已接近上限——已知精度问题。宁可高估不可低估（高估多压缩一次无害，低估 API 报错）。

**切割点算法**：`findValidCutPoints` 规则——**user 和 assistant 是合法切点，toolResult 不是**（toolResult 必须紧跟 toolCall，否则模型"调了工具找不到结果"）。关键语义：**切点不是"被切掉的最后一条"，而是"保留区的第一条"**。user 切点天然保证 Turn 完整；`findCutPoint` 从最新消息**往回走**累积 token 到 `keepRecentTokens`（20K），因为**最近的上下文最重要**。

**结构化摘要（填表不是自由发挥）**：6 个固定 section——Goal / Constraints & Preferences / Progress（Done / In Progress / **Blocked**）/ Key Decisions / Next Steps / Critical Context。固定模板强制 LLM 在每个维度扫一遍，防止"被有趣细节吸引忘了用户核心需求"。**增量更新**（`UPDATE_SUMMARIZATION_PROMPT`）：多次压缩时新摘要在旧摘要基础上更新而非重写，避免累积漂移。**文件跟踪**：`<read-files>` / `<modified-files>` 列表跨压缩累积——"改过哪些文件"是编码 Agent 最重要的元信息。

**极端情况 Turn 分割**：为什么允许 assistant 切点？因为只允许 user 切点会导致保留区永远过大（压缩失败）。允许 assistant 切点 → 精确控制 token 但切断 Turn → 用 **turnPrefix 机制**弥补：被切断的半个 Turn（user 在压缩区、assistant 在保留区）用专门的 `TURN_PREFIX_SUMMARIZATION_PROMPT` 生成 3 段轻量摘要（Original Request / Early Progress / Context for Suffix），与主摘要 `Promise.all` **并行生成**后合并。

**压缩结果如何生效**：每次压缩产生 `CompactionEntry`（含 summary / tokensBefore / firstKeptEntryId / details 文件跟踪），存在 Session Tree 上。下次运行 `buildSessionContext()` 把它转成 `CompactionSummaryMessage`（convertToLlm 翻译成 `<summary>` 标签的 UserMessage）替换旧消息。自动压缩集成在 agent_end 事件处理中，发出 `compaction_start` / `compaction_end` 两套事件供 UI 显示进度。

**设计精华**：①向后遍历 + 合法切点保护最重要的东西；②结构化模板对抗 LLM"自由发挥"；③文件跟踪累积领域特定知识。

---

## 第10章：会话管理 —— 对话的存储、恢复与分叉（进阶）

**两个独立子问题，必须拆开**：

- **存哪里（介质）**：coding-agent 选**本地 JSONL 文件**（单用户本地运行、会话跟项目走 `.pi/sessions/`、零依赖、可读可调试），但 agent-core 提供 `SessionStorage` 接口允许其他应用换数据库。注意：coding-agent 的 SessionManager 是**独立实现**，并未实现该接口
- **长什么样（结构）**：不是线性数组，而是 **Session Tree**——一棵**只追加、不修改、不删除**的树。回退/分支不是删数据，而是**移动 leafId 指针**

**8 步真实会话演示树的生长**（调试 auth bug）：切换模型 → 提问 → Agent 调 read → 工具返回 → Agent 分析 → **回退**（`leafId = "e2"`，一行代码，e3-e5 一个不删）→ 换思路重问（e6 与 e3 共享 parentId = e2，分支诞生）→ 新分支继续长。

**节点解剖**：`parentId` **认父不认子**——父节点不知道自己有哪些孩子（否则追加新子节点要修改父节点，违反 append-only）。**9 种 Entry 类型按"对 LLM 调用的影响"分三组**：进上下文（message / customMessage / compaction / branchSummary）、影响状态（modelChange / thinkingLevelChange）、纯元数据（label / sessionInfo / custom）。

**三个核心操作**：追加（创建节点 + byId 映射 + 移 leafId，O(1) 不改旧节点）、回退（只移 leafId）、分支（= 回退 + 追加的组合效果）。`branchWithSummary()` 可选地给被抛弃的分支生成 BranchSummaryEntry（"遗言"）。

**buildSessionContext（树 → LLM 线性数组）**：

1. **路径遍历**：从 leaf 走回 root，只看当前分支（被抛弃的分支数据不发给 LLM）
2. **按类型分派**：消息进 messages 数组，model_change 更新状态变量
3. **状态变量覆盖式提取**：路径上最后一条 model_change 生效；**节点化的状态让回退天然正确**（回退到切换模型之前的节点，model 自动回到旧值）
4. **CompactionEntry 选择性收集**：按 `firstKeptEntryId` 只收集保留区消息，跳过被压缩的旧消息——压缩不是破坏性的，回退到压缩点之前旧消息会重新出现

**JSONL 细节**：一行一个 Entry，第一行 Session Header；`id` 是 8 位短 UUID；**延迟写入**避免"有问无答"的半截对话（首次 assistant 消息到达前不落盘，首次 flush 用原子性全文件重写）；偶尔全文件重写（创建分支副本、修复损坏文件）。

**三个可迁移思路**：①拆开"存哪里"和"长什么样"两个维度；②append-only 数据结构用于撤销/回退/分支场景；③**节点化状态变量，让回退天然正确**。

---

## 总结：一条主线串起十章

从下往上理解 Pi 的完整设计：

1. **模型调用层**（第4章）用"统一入口 + 事件协议 + 翻译器"抹平 30+ 供应商差异，`streamSimple()` 一行驾驭所有模型
2. **Agent Loop**（第3章）用 stopReason 驱动 ReAct 循环，内核极简 + coding-agent 叠加 steering/followUp/钩子
3. **工具系统**（第5章）用五步管道 + 错误即消息 + Operations 抽象管住 Agent 的手脚
4. **消息系统**（第6章）用"内富外严"两层设计同时照顾模型和功能层两个读者
5. **事件驱动**（第7章）用同步屏障把 Agent 与 UI/日志/扩展彻底解耦
6. **上下文工程**（第8-9章）用"截断 + 组装 + 压缩 + 分支摘要"四层防线让有限窗口装下无限对话，核心是 Compaction（向后遍历找切点 + 结构化摘要 + 增量更新 + 文件跟踪）
7. **会话管理**（第10章）用 append-only 的 Session Tree 实现"不丢数据的回退/分叉"

**贯穿全系列的设计哲学**：做减法（内核极简、功能按需叠加）、依赖方向单向向上（分层不是教条，依赖控制才是）、每一层只加自己该关心的字段（Tool → AgentTool → ToolDefinition）、错误不抛异常而是变成消息（模型自己决定下一步）、数据结构同时照顾两个读者（结构化存储 + 边界翻译）。
