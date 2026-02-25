---
title: 从 LLM 到 Agent（下）：工程、框架与交付实践
date: 2026-02-25 11:39:16
tags: [Agent,LangGraph,LangChain,ClaudeCode,Manus,AI]
categories: Backend
toc: true
---

## 前言

上篇我们把“Agent 从哪来”讲清楚了：LLM/Chatbot 的结构性局限决定了它“会答”不等于“会做事”，而 Tool Calling / MCP / ReAct / DeepResearch 则让推理如何与证据、行动与产物形成闭环。

但当你真正开始落地时，会发现 Agent 的难点不在“让模型更聪明”，而在“让系统更可靠”：工具会失败、上下文会爆、成本与时延要控、权限与风险要收口、产物要可追溯、效果要能回归。也正是在这里，Agent 从概念走向工程。

本篇聚焦“工程与选型”：一方面复用上篇第 8/9 章的 checklist，把状态、持久化、上下文工程、评测与可观测性视作运行时能力；另一方面在此基础上对 LangChain/LangGraph 等框架与多种交付形态（Flow-based、CLI Agent）做对比拆解，给出更可落地的决策视角。

上篇
* 理解 LLM/Chatbot 的结构性局限：为什么“会答”不等于“会做事”
* 搞清 Tool Calling / MCP / ReAct / DeepResearch 的演化脉络：推理如何与证据、行动与产物形成闭环

本篇
* 掌握 Agent 工程骨架：状态、持久化、上下文工程、评测与可观测性
* 能做框架与交付形态选型：LangChain/LangGraph、高层框架、Flow-based、CLI Agent

阅读指引：如果只关心框架/形态对比，可以直接从第 10 章（本篇）开始；如果希望从概念开始理解，建议先回看上篇第 8/9 章的工程骨架，再来对照本篇的框架能力。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_1.mkdc68kf.webp)

本文 @coauthor: ChatGPT DeepResearch, Codex, Manus, AnyGen, Nano Banana 

## 10. Elemental Agent Frameworks：LangChain / LangGraph

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_37.8z70lko3on.webp)

第 8 章讨论了把 Agent loop 变成可交付工程系统的三块骨架：**状态、持久化、上下文工程**；第 9 章则回答“跑起来之后怎么变好、怎么排障、怎么上线止血”：**评测与可观测性**。把这些要求写成 checklist 后，会发现真正需要的是一套能把“状态演进 + 工具调用 + 轨迹数据”组织成确定性系统的框架原语。

这一节聚焦 LangChain 与 LangGraph：它们更接近“基础积木层”（elemental frameworks），常作为高层 Agent 框架/产品的底座（下一节会展开）。可以用一句话区分：

* **LangChain**：偏组件与集成层——解决“每一步怎么做、怎么接模型/工具/解析器、怎么挂 tracing/evals”。
* **LangGraph**：偏运行时与控制层——解决“多步怎么跑、状态如何演进、如何中断/恢复/回放、如何做人类介入”。

典型组合方式是：用 LangChain 写节点内部逻辑（RAG、tool calling、输出解析），用 LangGraph 把节点组织成显式状态图，并接入 checkpointer 与可观测/评测闭环。

### 10.1 LangChain：组件抽象层（可组合、可集成）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_38.51en4wd31h.webp)

LangChain 的核心价值不是“提供某一种 Agent”，而是把 LLM 应用里最常见的部件抽象成可替换的积木，并提供一致的组合与执行协议，从而在多模型、多工具、多数据源条件下保持链路的**可组合、可测试、可观测**。

从工程视角，LangChain 可以理解为两层：

* **原语层（Primitives）**：model / prompt / output parser / tool / retriever / memory 等部件的标准接口；
* **组合层（Composition）**：以 `Runnable`（LCEL）为中心，把“串行、并行、分支、流式输出、批处理执行”做成一等能力。

示意图（组件抽象 + 组合执行）：

```text
                         +-----------------------------+
                         |  config / tags / metadata   |
                         |  callbacks / tracing / eval  |
                         +--------------+--------------+
                                        |
input ----------------------------------v---------------------------+
                                                                      |
                     +----------------- Runnable (LCEL) -------------+-----> output
                     |                    (composition layer)        |
                     |
                     +--> [PromptTemplate] -> [LLM] -> [OutputParser] --------+
                     |                                                       |
                     +--> [Router/Branch] -----------------------------------+
                              |                          |
                              |                          +--> (Tool path) -> [Tools...] -> [LLM] -> ...
                              |
                              +--> (RAG path)  -> [Retriever] -> [Context] -> [LLM] -> ...
```

工程关键点：

* **统一执行协议（Runnables）**
  * 同一链路可用 `invoke`（单次）、`batch`（离线批量）、`stream`（流式）等形态运行，便于把线上请求与离线评测复用为同一套实现；
  * 运行期 config 可携带 `tags/metadata`、超时、并发度、回调等参数，使观测与治理从“外部包裹”变成“随链路流动”的上下文。
* **提示词与结构化输出**
  * Prompt templates 把“静态模板 + 变量填充 + system/user 分层”显式化，降低 prompt 漂移；
  * Output parser/结构化输出把“模型输出”收敛到 schema，使结果可直接进入后续步骤或写入状态，减少脆弱的正则解析。
* **工具与集成（Tooling）**
  * 工具以 schema 描述输入输出，与模型的 tool calling 对齐；工具集合可封装为 toolkits，形成可移植能力包；
  * Router/选择器用于把输入分流到不同链路（不同模型、不同检索策略、不同工具集合），便于实现路由、降级与 A/B。
* **RAG 生态**
  * 文档加载、切分、向量库、retriever 与 rerank 形成可插拔管线，适合把“上下文工程”落为可运营组件；
  * 可与第 8 章的 Working set 思路配合：长内容外置，链路只保留摘要与引用，并按需检索回填。
* **可观测与评测接口**
  * Callback/Tracing 把 LLM 生成、tool call、retrieval 等事件变成统一轨迹，可对接 LangSmith 或自建 tracing；
  * 便于把第 9 章的 eval 机制落到可回放链路：同一 Runnable 既能服务线上，也能被离线批量回归。

适用场景：

* 强集成（多模型/多工具/多数据源）、链路相对可控、希望快速搭建可测试的 RAG/工具调用应用；
* 团队协作需要“组件标准化”的边界：工具、检索、解析、模型路由可以并行迭代并统一治理。

### 10.2 LangGraph：状态机/图执行运行时（可控、可恢复）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_39.4n87e14s6a.webp)

LangGraph 解决的问题是：当链路从“线性流程”演进为“带循环、分支、并行与人类介入的长任务”时，如何让执行具备可控的状态演进与可恢复性。其基本模型是把 Agent 表达为显式状态机（state machine）：节点对共享状态做增量更新，运行时负责调度、合并、持久化与回放。

示意图（显式状态图 + 可恢复执行）：

```text
  +-------+        +---------+        +----------+        +------+
  | START | -----> | Planner | -----> | Executor | -----> | END  |
  +-------+        +---------+        +----------+        +------+
                          |                |
                          |                +----> [Tools / I-O] ----+
                          |                                         |
                          +----> (Interrupt: approval/input) <------+
                                   ||
                                   || resume(thread_id, state)
                                   \/
                                (continue)

  每个 super-step：节点读 state -> 产出 patch -> reducer 合并 -> 写入 checkpoint(thread)
```

核心抽象与执行语义：

* **StateGraph（状态图）**：定义状态 schema、节点函数与边（条件路由/分支规则）；节点产出的是“对状态的 patch”，而不是仅依赖自由文本对话推进；
* **Channels / Reducers（状态合并）**：同一 super-step 内的并行节点可各自写入更新，通过 reducer 合并为下一轮状态，减少隐式冲突并显式定义“可合并的状态更新路径”；
* **Super-step 调度（图运行时）**：满足条件的节点被激活运行，直到到达 `END` 或无新的状态更新，从而自然支持循环、分支与并行；
* **Checkpointer / Threads（持久化与恢复）**：在每个 super-step 持久化状态快照与事件轨迹，形成 thread；可据此实现 resume、replay 与 time-travel debugging，把第 8 章的 durable execution 变为运行时能力；
* **Interrupts / Human-in-the-loop**：在关键节点触发 interrupt 暂停执行，将“等待外部输入/审批”纳入图语义；恢复依赖 thread_id 与持久化状态；
* **Subgraphs / Parallelism（规模化编排）**：子图用于 context quarantine，把子任务上下文与状态隔离；并行 worker 节点适配 deep research/批量检索/多候选生成，再由聚合节点做对齐与投票。

示意图（并行节点 + reducer 合并）：

```text
                   (same super-step)
                +---------------------+
                |       STATE S       |
                +----------+----------+
                           |
               +-----------+-----------+
               |                       |
        +------+------+\        /+------+------+
        | worker A    | \      / | worker B    |
        | (patch A)   |  \    /  | (patch B)   |
        +-------------+   \  /   +-------------+
                           \/
                    +--------------+
                    |  reducer()   |   merge patches
                    +------+-------+
                           |
                    +------+------+
                    |  STATE S'   |   checkpoint -> thread
                    +-------------+
```

与 LangChain 的关系（常见组合方式）：

* LangChain 更适合表达“节点内部怎么做”（提示词、RAG、tool calling、解析与路由组件）；
* LangGraph 更适合表达“节点之间怎么跑”（循环、分支、并行、恢复、审批与运行时治理）。

在组合架构中，LangChain 的 Runnable 常作为 LangGraph 节点实现；LangGraph 提供 checkpointer/thread 体系承载状态、轨迹与恢复路径，并可将 tracing/evals 串入第 9 章的评测与可观测闭环。

适用场景：

* 长任务、多工具编排、需要审批/暂停/恢复、需要回放与审计的生产 Agent；
* 需要把“状态更新路径”做成显式合约（schema + reducer），并在此基础上做回归评测与线上监控。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_40.6wr7xipin4.webp)

## 11. High-level Agent Frameworks: DeepAgents / Mastra / CrewAI / Agno

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_41.466b1zb8d.webp)

上一节的 LangChain/LangGraph 更像“搭积木”：提供组件抽象与可控运行时，但要把它们拼成一个可交付系统，仍需要补齐大量工程胶水——计划与任务分解、工作区与上下文外置、记忆与存储、可观测与评测、权限与发布流程。这些恰好对应第 8/9 章的主线：**把状态/持久化/上下文工程做成运行时能力，并把评测/可观测性做成回归与运维能力**。

这一节的高层框架（high-level frameworks）更接近“带电池的 Agent 系统骨架”：把常见的 agent harness/编排/运维能力做成默认配置与最佳实践，让项目更快从 demo 走到可交付形态；代价是更强的约定与更明确的框架边界（通常需要按其抽象组织工具、状态与执行流）。

一个实用的选型问题是：框架主要解决哪类“系统问题”——

* **长任务与工作区（coding/研究型任务）**：更像 CLI harness，把文件系统当上下文后端（DeepAgents）。
* **工作流 + 记忆 + 可观测一体化**：更像应用框架/平台原语，偏工程落地（Mastra）。
* **角色化协作与任务编排**：用“角色/流程”表达多代理分工与控制（CrewAI）。
* **私有化运行时与控制平面**：从编排走向服务化交付与部署治理（Agno）。

### 11.1 DeepAgents：长任务 harness（规划 + 文件系统 + 子代理）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_42.8vnenuv0yd.webp)

DeepAgents 的定位可以理解为“长任务 Agent 的工作台（harness）”：与其把全部能力压在单次 prompt 上，不如把任务拆成可执行步骤，并把长上下文与中间产物落到工作区，从而让多步任务更接近可交付的工程流程。([GitHub][24]; [LangChain Docs][26])

示意图（harness：todo + workspace + subagents）：

```text
                      +----------------------+
goal/spec ----------> |   Main Agent Loop    |
                      | (planner + executor) |
                      +----+-----------+-----+
                           |           |
                 write_todos()         | task()
                           |           |
                    +------+----+      v
                    |  TODOs   |  +-----------+
                    | (plan)   |  | Sub-agent |  isolated context
                    +------+----+  +-----+-----+
                           |             |
                           v             v
                      +----+-------------+------------------+
                      |           Workspace (files)          |
                      |  inputs / notes / code / artifacts   |
                      +--------------------------------------+
                           ^
                           |
          ls/read_file/edit_file/write_file (context back-end)
```

工程关键点（把第 6/8/9 章的建议“产品化”成默认能力）：

* **规划与任务分解（显式进度）**
  * `write_todos` 将 plan-and-execute 落为可读的任务清单，使“下一步做什么”不再依赖隐式对话记忆；
  * TODOs 既是对齐工具（便于人工 review/调整优先级），也是最小可用的运行时状态（完成/阻塞/风险）。
* **文件系统作为上下文后端（working set + 外置记忆）**
  * `ls/read_file/write_file/edit_file` 将长文本与中间产物外置到 workspace，把上下文工程从“塞 prompt”变为“读写可引用的工件”；
  * 以文件为载体沉淀证据、草稿与变更路径，天然有利于审计与回放（尤其适合跨文件产出与多轮迭代）。
* **子代理委派（context quarantine + 分而治之）**
  * `task` 用于派生子代理处理子问题（检索、分析、实现某个模块等），主代理只接收“可交付摘要/结论”，降低上下文污染；
  * 并行子任务在结构上更接近 DeepResearch 的编排方式：独立上下文 → 汇总对齐 → 合并到主计划。
* **与底座的组合**
  * DeepAgents 常与 LangChain/LangGraph 生态的 tracing、存储与工具体系协同；例如结合 LangGraph Store 承载跨线程长期记忆与检索回填（适合偏长期运行的项目知识沉淀）。

官方提供了 CLI 文档与 quickstarts 示例工程，便于上手实践。([LangChain Docs][25]; [GitHub][27])

适用场景：研究/编码类长任务、需要把“计划—工件—子任务—汇总”串成稳定工作流的团队；尤其适合强调 workspace 产出与可审计过程的交付型 Agent。

### 11.2 Mastra：TypeScript 生态的一体化框架（Workflows + Memory + Observability）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_43.3k8i358yac.webp)

Mastra 是偏“工程化产品”的 TS 框架，提供 workflows、agents、RAG、integrations、evals 等原语，并强调模型路由与可观测。

工程关键点：

* **Workflows**：用 workflow/step 把多步执行显式化，便于治理重试、分支与编排复杂度。
* **Observability**：启用后自动对 agent run、LLM 生成、tool call、workflow step 生成 trace，并携带 AI 语义上下文。
* **Memory**：提供 thread/消息存储与语义召回（向量）组合的记忆系统，支持不同存储后端。

适用场景：Node/TS 团队、需要“工作流 + 记忆 + 可观测 + 多模型/多 provider”一体化工程栈。

### 11.3 CrewAI：角色/任务编排（Crew / Process / Flows）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_44.lw7zn0osy.webp)

CrewAI 以“角色化 agent + 任务编排”为中心，提供 Crews 与 Flows 两条主线。

工程关键点：

* **Process（Sequential / Hierarchical）**：默认顺序执行；层级模式由 manager 协调分派与校验，并要求配置 `manager_llm` 或自定义 `manager_agent`。
* **Memory**：提供短期/长期/实体记忆等组件化能力。
* **Flows**：面向“事件驱动的工作流”，用于把 crews 与编码任务组合成更可控的自动化流程。
* **可运维能力**：如 tool hooks（拦截与控制工具执行）、replay（从最近一次 kickoff 回放特定任务）。

适用场景：强调组织结构化协作、任务分工明确、希望用“流程/角色”表达编排逻辑的团队。

### 11.4 Agno：框架 + 运行时 + 控制平面（面向私有化生产部署）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_45.41yjrqabv5.webp)

Agno 在定位上同时覆盖：multi-agent framework、FastAPI 运行时与控制平面，强调“自有云、自有数据”，并提供 AgentOS 作为生产服务形态。([GitHub][28]; [DeepWiki][29])

工程关键点（从“编排库”往“交付形态”走）：

* **Agents / Teams / Workflows**：面向 multi-agent 与流程编排的核心抽象。
* **记忆与知识**：内置记忆与知识管理能力，面向长期运行的生产系统。
* **Guardrails / Hooks / MCP**：运行时治理与扩展点（策略、拦截、工具生态）。
* **FastAPI runtime + UI 控制平面**：把 agent 作为服务运行与管理，而不仅是本地脚本。

文档中提供了案例与应用示例，包括 Deep Research Agent。([DeepWiki][30]; [docs.agno.com][31])

适用场景：更偏“生产系统交付”，关注私有化部署与服务化运行（而不仅是本地编排）。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_46.icm1x7m34.webp)

---

## 12. Legacy / Flow-based：Dify / Flowise / n8n

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_47.8adr1k0kni.webp)

上一节讨论的高层 Agent 框架更偏“以运行时为中心”：围绕 agent loop 把状态、持久化、上下文工程做成可控系统骨架。而在另一条更早的工程路径上，业界长期存在一类“流程编排/自动化”产品：用节点图（flow）表达业务逻辑，把调用模型、检索知识库、触发外部系统当作图中的节点。随着 LLM 普及，这类平台快速吸收了 RAG、tool calling 与提示词管理能力，形成了 Dify / Flowise / n8n 等常见选择。

从第 8 章的工程骨架看，Flow-based 平台往往将 Agent 的关键问题“降维”为更易治理的对象：

* **状态**：以 workflow variables / inputs / outputs 表达跨节点数据流，状态更新路径由执行图显式约束；
* **持久化**：以数据集/知识库、运行记录与版本发布等形态沉淀过程与资产，便于复现与审计；
* **上下文工程**：以知识库检索配置、节点级 prompt、以及变量拼装规则控制 working set，而不是把一切塞进对话历史。

从第 9 章的运维视角看，它们的优势通常不在“推理范式”，而在“可运营性”：提供日志、调用统计、成本与延迟等指标入口，配合可视化编排与连接器生态，让 LLM 能更快嵌入既有业务流程。

取舍也很明确：Flow-based 平台更偏确定性的流程执行，对“开放式循环 + 细粒度 checkpoint/resume + 人类介入语义 + 可回放轨迹”的支持通常不如专用 agent runtime；但当目标是**快速落地业务流程、降低编排门槛、强化连接器与平台化运维**时，Flow-based 往往是更省工程成本的选项。

下面分别概览三类代表：Dify 偏平台化应用与 LLMOps；Flowise 偏可视化组装 LangChain 组件；n8n 偏企业自动化引擎，把 LLM 节点嵌入成熟工作流体系。

### 12.1 Dify：workflow + RAG + LLMOps 平台化

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_48.6iks6nh7rh.webp)

Dify 的定位更接近“LLM 应用的平台化交付层”：以可视化 workflow 为核心，把模型调用、知识库检索、工具/插件接入、运行日志与发布运维整合到一个统一的控制台中，从而降低端到端落地门槛。

示意图（平台化：构建台 + 运行时 + LLMOps）：

```text
   +-------------------+        publish/version        +-------------------+
   |  Studio / Console | ----------------------------> |   App Runtime     |
   |  (workflow UI)    |                               | (API / Web / chat)|
   +---------+---------+                               +----+---------+----+
             |                                              |         |
             | configure                                    | run     | observe
             v                                              v         v
   +-------------------+                         +----------------+  +-------------------+
   | Workflow Graph    | ---- steps/nodes -----> | Node Executor  |  | Logs / Analytics  |
   | (variables/branch)|                         | (LLM/RAG/tools)|  | tracing/cost/debug|
   +---------+---------+                         +--------+-------+  +-------------------+
             |                                              |
             | RAG                                           | providers/tools
             v                                              v
   +-------------------+                         +------------------------+
   | Datasets / KB     | <--> embeddings/index   | Models / Plugins / APIs|
   +-------------------+                         +------------------------+
```

工程要点（workflow + RAG + LLMOps 的平台化拆解）：

* **应用形态与发布**
  * 将“对话应用 / workflow 应用”等形态产品化为可配置实体，并提供发布与运维入口，适合业务侧快速迭代上线；
  * 通过环境配置、密钥管理、权限与多租户等能力，将模型能力纳入平台治理。
* **Workflow 编排（可视化执行图）**
  * 节点化表达：LLM、检索、条件分支、变量读写、外部 API/插件调用等，降低复杂链路的表达成本；
  * 变量系统支撑“轻量状态”：对话/工作流中可将中间结果写入会话变量（conversation variables），用于跨节点传递与复用。
* **知识库与 RAG 管线**
  * 数据集/知识库模块承载文档导入、切分、向量化与检索配置，使 RAG 更接近可运营资产；
  * 适合与第 8 章的上下文工程结合：长资料在知识库中沉淀，workflow 只按需检索回填到 working set。
* **LLMOps（观测、排障与运营）**
  * 平台日志与运行记录使一次调用/一次 workflow run 可被定位与复盘，降低排障门槛；
  * 成本、延迟与失败等数据更容易沉淀为运营指标与告警策略（呼应第 9 章的可观测与评测闭环）。
* **插件与连接器生态**
  * Plugins 机制用于扩展模型提供方与外部能力，把“连接器生态 + 平台运维”作为默认优势；
  * 适合将 LLM 节点嵌入既有业务系统（CRM、工单、知识库、审批流等）。

边界与取舍（Flow-based 平台的典型限制）：

* 对长任务的“强状态机 + durable execution”支持通常弱于专用 agent runtime（checkpoint 粒度、可回放语义、人类介入编排能力更依赖平台能力边界）；
* 一旦需要高度定制的运行时控制、复杂的外部副作用治理或深度代码编排，可能需要下沉到代码框架或自建运行时。

适用场景：快速落地业务流程、RAG 应用与可运营后台；强调可视化编排、连接器生态与平台运维，对“强状态机 + 可恢复执行”的要求相对较低或可接受平台约束。

### 12.2 Flowise：可视化 LangChain 编排（从 Chatflow 到 Agentflow）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_49.5q7wox0m18.webp)

Flowise 定位为开源可视化构建器，面向“把链路拖出来”。其文档将能力扩展到 Agentflow 等更复杂编排形态。

适用场景：原型期/交付期希望用可视化表达链路；或团队需要把 LangChain 组件化能力封装给非核心开发者使用。

### 12.3 n8n：企业自动化工作流引擎（把 LLM 节点放入管道）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_50.3d5a7pmsuj.webp)

n8n 的优势在于 triggers、节点连接器与成熟的工作流引擎；并可通过 LangChain 节点（如 `@n8n/n8n-nodes-langchain`）把 LLM/向量库等能力接入自动化管道。

适用场景：企业流程自动化（工单、消息、CRM、数据同步）中嵌入 LLM；强依赖连接器生态与可视化运维。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_51.4cldkvpk08.webp)

---

## 13. CLI Agent 的实现思路：以 Claude Code 为主线

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_52.8z70lko3nr.webp)

前面几章从不同抽象层给出了“Agent 工程化”的路径：LangChain/LangGraph 提供组件与运行时原语，高层框架提供 harness，而 Flow-based 平台把 LLM 能力嵌入业务流程。CLI Agent 则是另一种非常务实的交付形态：把执行环境直接放到终端与工作目录里，使“读文件 / 改代码 / 跑命令 / 写产物”成为默认动作。

对照第 8 章的骨架，CLI Agent 的关键不在于对话形式，而在于把**工作区、工具与权限**做成显式运行时：状态可追踪、变更可审计、上下文按需读取；对照第 9 章的要求，则需要把一次 run 做成可回放的轨迹，并把测试、成本与失败模式纳入日常开发与运维闭环。

本节以 Claude Code 为主线，拆解现代“编码型 Agent”的系统结构与落地要点：核心架构、扩展能力、Sub-agent、Skills。

### 13.1 Core: Orchestrator, Plan, Tool, Agent Loop

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_53.1sfj88pldy.webp)

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_54.92qmjah6dh.webp)

这一节不讨论“某个模型有多会写代码”，而是回答一个更工程化的问题：**像 Claude Code 这类 CLI 编码型 Agent，为什么能把长任务稳定跑完？**

核心在四个原语：**Orchestrator（编排器）**、**Plan（计划）**、**Tool（工具）**、**Agent Loop（闭环循环）**。它们把第 6/7/8/9 章的抽象（显式计划、状态/持久化、上下文工程、可观测与回放）落到一个可运行的终端执行系统里。

先给一张“最小闭环”结构图（概念级，具体实现细节留到第 14 章）：

```text
User goal/spec
      |
      v
+------------------------+        +----------------------+
|      Orchestrator      |<------>|   Plan (TODOs)       |
| loop driver + LLM      |        | steps / acceptance   |
| budget + safety gates  |        | status / risks       |
+-----------+------------+        +----------------------+
            |
            | chooses tool + inputs
            v
+------------------------+        +----------------------+
|      Tool runner       |------->|  Workspace / Env     |
| registry + schema      |        |  files/shell/git/MCP |
| permissions + sandbox  |        +----------------------+
+-----------+------------+
            |
            | tool results (stdout/stderr/diff/errors)
            v
     Observations -> state update -> next decision
```

下面分别解释这四个原语各自的“职责边界”和“为什么缺一不可”。

#### 13.1.1 Orchestrator：把“模型调用”变成“可控执行”

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_55.54y92m65qc.webp)

在 Chatbot 里，LLM 是主角；在 Agent 里，LLM 更像“策略模块（Policy）”，而 **Orchestrator 才是运行时的控制面**：它驱动循环、维护状态、约束工具副作用，并把一次 run 变成可审计、可恢复的执行过程。

一个面向编码任务的 Orchestrator，通常至少负责：

* **把用户目标转成可执行任务**：明确范围、约束、交付物与验收方式（例如“改哪些文件、跑哪些测试、输出什么补丁/说明”）。
* **上下文装配（working set）**：决定本轮该读哪些文件、要不要搜索、要不要看 git diff；避免把全仓库塞进 prompt。
* **计划生命周期管理**：生成/更新计划、推进步骤状态、处理阻塞与重规划（与 Plan 强绑定，下一小节展开）。
* **工具路由与调度**：选择工具、填充参数、执行、捕获输出，并将结果转成可消费的 Observation。
* **安全门禁（safety gates）**：在关键副作用点强制“先计划/先 review 再执行”，并对高风险动作触发审批或策略拦截（与 permissions/sandbox 强绑定）。
* **预算与停止条件**：限制最大步数、工具调用次数、wall time、token 成本；当无法推进时及时停下来请求人工信息或确认（呼应第 7 章的 stop condition）。
* **失败恢复与回放数据采集**：为每一步记录 `step_id/tool_call_id`、输入/输出摘要、关键中间产物；出现错误时能定位、重试、回滚或从 checkpoint 继续。

把 Orchestrator 视为“执行系统的内核”有一个直接好处：**可以替换策略模型（Policy/LLM），但不破坏执行语义**。例如把同一个工具层/权限层/日志层复用在不同模型或不同 prompt 策略上，使迭代更像软件工程而不是“调参玄学”。

#### 13.1.2 Plan：从“漂亮回答”到“可推进的任务状态”

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_56.7i0vjtjywy.webp)

Plan 不只是“开头写一段计划”，而是**贯穿整个 run 的显式任务状态**。它解决两个核心问题：

1. **长任务漂移（plan drift）**：模型容易在对话中忘记早期约束；计划把约束、里程碑与验收条件固定下来（呼应第 6 章）。
2. **可控推进**：每一步都能回答“下一步要做什么、为什么要做、做完如何验证”，让 Orchestrator 能做 gating、预算控制和回放。

在 CLI 编码型 Agent 中，计划的最佳形态往往是“可读、可更新、可对齐”的 TODO 列表（人类也能快速 review）。一个实用的最小字段集可以是：

* `step`: 这一步要达成什么（动词开头）
* `acceptance`: 怎么算做完（可验证条件，尽量落到命令/测试/对比）
* `tools`: 预期会用哪些工具（读文件/搜索/跑命令/改文件）
* `artifacts`: 这一步会产出什么（diff、日志文件、说明文档）
* `risk`: 是否涉及副作用/权限（例如写配置、删文件、发请求）
* `status`: todo/doing/done/blocked（以及阻塞原因）

示例（概念表达，具体格式可用 Markdown TODO / YAML / JSON 皆可）：

```yaml
plan:
  - id: 1
    step: "定位问题与影响范围"
    acceptance: "能用最小复现或日志解释失败原因"
    tools: ["rg", "read_file", "shell"]
    artifacts: ["notes/analysis.md"]
    risk: "read-only"
    status: "todo"
  - id: 2
    step: "实现修复并生成补丁"
    acceptance: "关键路径逻辑正确，代码可构建"
    tools: ["edit_file"]
    artifacts: ["git diff / patch"]
    risk: "write"
    status: "todo"
  - id: 3
    step: "验证与回归"
    acceptance: "相关测试通过或提供替代验证证据"
    tools: ["shell"]
    artifacts: ["notes/verification.md"]
    risk: "exec"
    status: "todo"
```

计划在运行时需要“可演进”：

* **Plan validation（计划校验）**：在执行前检查权限/依赖/可执行性/成本。例如计划里出现“重置数据库”这类不可逆动作，应被标红并要求明确审批；出现“跑全量测试”也要做时间预算评估。
* **Plan refinement（逐步细化）**：先用粗粒度里程碑锁定方向，再在接近执行时把步骤细化成可操作的 tool plan（输入输出、预期失败模式）。
* **Replanning（基于 observation 重规划）**：工具返回的信息与预期不一致时，应允许替换策略（换方案/回退/拆分子任务），但要把变更写回计划，避免“悄悄跑偏”。

可以把计划理解为 CLI Agent 的“最小状态机”：**计划状态 + 工具结果**，共同决定下一步动作；而不是靠对话历史隐式推进。

#### 13.1.3 Tool：把外部世界纳入闭环，但要可治理

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_57.8ok6sf8vi9.webp)

Tool 是 Agent 获得 grounding 和行动能力的入口（读文件、写补丁、跑命令、查 git、访问网络/MCP）。但工具也带来真实世界的不确定性：权限、速率限制、输出波动、以及不可逆副作用。编码型 Agent 的关键不是“工具越多越好”，而是**工具必须有合约（contract）与治理（governance）**。

一个成熟的工具层通常包含三部分：

1. **Tool registry（工具注册表）**：列出工具名、参数 schema、输出类型、失败类型、以及风险分级。
2. **Permission/Sandbox（权限与隔离）**：把工具调用分级成 allow/ask/deny 或只读/可写/高危；并用沙箱减少“无意义审批”，同时把副作用控制在边界内（第 13.2 再展开具体扩展机制）。
3. **Tool runner（执行器）**：统一超时、重试、输出截断、日志落盘与幂等策略，把“工具不稳定性”收敛成可预测的返回结构。

实践中很有效的一种做法是按“副作用”给工具分层：

* **Read-only**：`ls/read_file/rg/git show` —— 可随时自动调用，主要风险是泄露敏感信息与上下文污染（需要脱敏/截断策略）。
* **Repeatable side effects（可重复副作用）**：`format/lint/test` —— 失败可重试，适合作为验收手段；风险是耗时与资源占用（需要预算）。
* **Non-reversible side effects（不可逆副作用）**：`rm/mv/写生产配置/发外部请求` —— 默认需要明确审批、双写审计或 dry-run，必要时要求“先产出 diff/计划再执行”。

另外两条“编码任务常用但容易被忽视”的工具治理原则：

* **输出要可引用**：长 stdout/stderr 不要全塞回 prompt，优先落盘到 workspace（日志文件、patch 文件），再在上下文里引用摘要与关键片段（呼应第 8 章上下文工程）。
* **验证要工具化**：尽量把 acceptance 写成可执行命令或可检查的 diff，而不是“我觉得修好了”。这会显著降低长链路的误差累积（呼应第 9 章评测与回归思路）。

#### 13.1.4 Agent Loop：观察-决策-行动-更新-停止的工程化版本

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_58.73ufsybo1s.webp)

Agent Loop 是第 7 章提到的“observe-act loop”的落地：它把一次“问答”扩展为多轮执行，并把每一轮变成可审计的 step。

一个典型 CLI 编码型 Agent 的 loop，可以拆成“外层计划推进 + 内层工具闭环”两层（概念伪代码）：

```js
state = init(goal, constraints, env)
plan  = make_or_load_plan(state)

while not stop(state, plan):
  obs = collect_observations(state, plan)     # files, diffs, tool results, user feedback

  decision = policy(obs, state, plan)         # update_plan | call_tool | ask_user | finalize

  if decision.type == update_plan:
    plan = apply_plan_update(plan, decision)  # validation + versioning
    continue

  if decision.type == call_tool:
    assert permission_ok(decision.tool)
    result = run_tool(decision.tool, decision.args)
    state, plan = reduce(state, plan, result) # record trace, advance step, summarize outputs
    continue

  if decision.type == ask_user:
    pause_for_human()
    continue

return deliverable(plan, artifacts, verification)
```

这里的关键不是“循环本身”，而是 Orchestrator 在每一轮强制执行的几类约束：

* **预算约束**：步数/时间/token/tool calls；避免陷入无效循环或成本失控。
* **循环检测**：重复读同一文件、重复跑同一命令、计划不推进等信号出现时，触发重规划或请求人工信息。
* **失败策略**：工具失败时优先“修正输入→重试”，再到“换策略→回退 checkpoint→拆分子问题”；必要时进入 reflection（对失败原因结构化总结）并写入工作区，防止反复踩坑。
* **交付导向**：每个步骤都要产出可交付的工件（diff、日志、说明、测试结果），最终组合成用户可消费的结果，而不是只输出一段“我做了什么”的叙述。

### 13.2 Extends: Plugins / Slash commands / MCP / Hooks

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_59.3gow5ffvjz.webp)

13.1 描述的是 CLI Agent 的“内核”（Orchestrator/Plan/Tool/Loop）；13.2 关注的是“扩展面”：**在不改内核语义的前提下扩工具、加治理、做复用**。

* **Slash commands（控制面）**：用显式命令管理运行时配置与边界（例如 `/permissions`、`/sandbox`、`/mcp`、`/hooks`），把“策略/权限/调试”从自然语言里剥离出来。
* **MCP（工具接入标准）**：以 MCP server 形式提供 tools/resources，CLI 作为 client 纳入 tool registry；调用仍受权限、审计与沙箱约束。
* **Plugins（可分发能力包）**：将工具、命令、默认配置与 hooks 打包，便于团队复用同一套工作流与治理规则。
* **Hooks（可插拔治理）**：在 plan 更新、tool 调用、文件写入等节点拦截/改写/自动化（例如强制先产出 diff、自动跑最小验证）。
* **Memory（可选）**：把跨会话经验做成“可检索、可引用”的输入，按需回填到 working set，而不是当作无限对话历史。
* **Permissions & sandboxing（底线）**：把“能做”与“允许做”分离，通过 allow/ask/deny 与隔离边界控制副作用。

### 13.3 Sub-agent & Skills

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_60.8z70lko3nh.webp)

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_61.26lyz3xw8s.webp)

在 13.1 的四原语里（Orchestrator/Plan/Tool/Loop），**Sub-agent** 与 **Skills** 都不属于“新的内核组件”，而是 Orchestrator 的两种常见“增强手段”：

* **Sub-agent**：把一个大任务拆成子任务，交给一个上下文隔离的执行体去跑，再把结果（摘要/证据/产物）汇总回主 loop（对应“分层编排 + context quarantine”）。
* **Skills**：把一套稳定可复用的做事方式（步骤、约束、工具偏好、输出格式）模块化封装，供主 agent 在合适时机调用（对应“可复用策略/工作流”）。

#### 13.3.1 Sub-agent：并行与隔离的执行单元

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_62.5xb4kcmrga.webp)

Sub-agent 适合解决两类问题：**并行吞吐**与**上下文污染**。

* **并行吞吐**：多个互不依赖的子问题（调研、比对、生成候选方案）可以并发跑，提高端到端速度。
* **上下文隔离（context quarantine）**：子任务的噪声、错误推断、长日志不直接进入主上下文；主 agent 只接收“可交付摘要 + 证据/产物”。

一个实用的 sub-task 输入输出合约（让 Orchestrator 更好调度与回放）：

* **输入（给子代理）**：子目标、约束/风险点、允许工具与权限边界、working set（可读文件/链接/线索）、期望交付格式与验收点。
* **输出（回主代理）**：结论摘要、关键证据（引用到文件/命令输出/链接）、产生的工件（patch/log/notes）、未决问题与下一步建议。

典型使用场景：

* **代码库快速摸底**：让子代理分别理解不同模块/目录，主代理只做汇总与决策。
* **多方案探索**：并行提出 2–3 个实现方案与风险评估，主代理选择其一落地。
* **失败定位**：子代理专注复现/定位错误与最小修复路径，避免主 loop 被长日志淹没。

需要注意的治理点（呼应 13.1）：

* **预算与停止条件**：对子代理单独设定 step/time/tool budget，避免“子任务失控”拖垮主任务。
* **权限与副作用**：对子代理的写入/执行权限应更保守；高风险动作仍由主代理走审批或二次确认。
* **汇总前验证**：子代理给的结论优先要求“可验证证据”（diff、可运行命令、可复现步骤），否则容易把幻觉在汇总阶段放大。

#### 13.3.2 Skills：可复用的策略模块（workflow/playbook）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_63.9ddgcfweih.webp)

如果 sub-agent 是“把人力拆成多个并行执行者”，skills 更像“把资深同事的做法写成 SOP”。它通常用于提升一致性与可控性：

* **稳定的步骤模板**：例如“先读哪些文件、再做哪些检查、最后如何验收”的固定顺序。
* **固定的输出结构**：例如必须输出 TODO、风险点、验收命令、回滚方案等。
* **工具偏好与约束**：例如优先用只读工具收集证据；修改代码前必须产出 diff；跑命令前先说明目的与预期结果。

从 13.1 的视角看，skills 主要影响的是：

* **Plan 的质量**：把“计划怎么写、验收怎么写”固化成可复用规范。
* **Tool 的使用方式**：把“什么情况下调用什么工具、如何处理输出/失败”固化为策略。
* **Loop 的稳定性**：减少随机游走，降低“重复读/重复跑/不推进”的概率。

#### 13.3.3 Sub-agent vs Skills：如何选择

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_64.2a5kwtqyyg.webp)

可以用一句话区分：

* **Skills**：解决“怎么做更一致、更可控”（复用策略）。
* **Sub-agent**：解决“谁来做、怎么隔离”（分工与并行）。

一个快速判断表：

| 维度 | Sub-agent | Skills |
| --- | --- | --- |
| 形态 | 运行时执行体（可并行） | 静态策略/流程模块（可复用） |
| 上下文 | 默认隔离，主代理汇总 | 通常直接作用于主代理 |
| 适合 | 多子问题、并行探索、隔离噪声 | 重复工作流、规范化输出、组织约束 |
| 风险 | 成本上升、结论不一致、汇总困难 | 过度约束、技能陈旧、与项目冲突 |

实践中最常见的组合是：完全独立、高耗时、低频的事情，适合做 Sub-agent，反之可用 Skills。值得注意的是，Sub-agent 也可以加载 Skills，所以其实二者不冲突。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_65.4jolgbbpfb.webp)

---

## 14. CC Build From Scratch: [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_66.9rjw3b4pdh.webp)

这一章不做“Claude Code 逆向分析”，也不试图复刻产品 UI，而是用一个可运行的教学项目 [learn-claude-code][32]，把第 13 章的四原语（Orchestrator/Plan/Tool/Loop）落成 **一套最小但完整的 CLI 编码型 Agent 骨架**：能读仓库、能改文件、能跑命令、能拆任务、能按需加载技能。

learn-claude-code 的设计非常适合当“从 0 到 1 的工程脚手架”：**v0→v4 五个版本，每个版本只新增一个概念**（bash 工具 → 4 工具 → Todo → Sub-agent → Skills），总共约千行代码，但把核心机制讲得非常清楚。

* 学习材料入口：
  * repo: https://github.com/shareAI-lab/learn-claude-code
  * articles（偏公众号风格）: https://github.com/shareAI-lab/learn-claude-code/tree/main/articles
  * docs（偏技术讲解，中英）: https://github.com/shareAI-lab/learn-claude-code/tree/main/docs

### 14.1 目标与最小闭环：先把“能跑起来”变成第一原则

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_67.wj1ssfwxg.webp)

如果只记住一句话：**编码型 Agent = 一个允许模型反复调用工具直到完成任务的循环**。

learn-claude-code 给出的“最小闭环”几乎可以作为所有 CLI Agent 的伪代码模板：

```python
messages = [{"role": "user", "content": user_goal}]

while True:
  r = model(messages, tools)
  if r.stop_reason != "tool_use":
    return r.text

  results = execute(r.tool_calls)  # bash/read/write/edit/...
  messages.append({"role": "user", "content": results})
```

把它对照回第 13 章四原语：

* **Orchestrator**：驱动这个 loop；决定何时停、何时读文件、何时调用什么工具、何时重规划。
* **Tool**：把“动作”从自然语言里剥离为可执行 schema（bash/read/write/edit/...）。
* **Plan**：把“下一步做什么”变成显式状态（v2 的 TodoWrite 就是最小可用形态）。
* **Agent Loop**：observe→act→observe 的闭环，靠工具结果把 hallucination 拉回现实。

这一章的写法会沿着 learn-claude-code 的版本递进来展开：每一小节都回答三件事：**新增了什么？解决了什么问题？代价/边界是什么？**

### 14.2 v0：Bash is All You Need（1 个工具也能涌现出“完整 Agent”）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_68.32igek7koj.webp)

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_69.6po0233d6a.webp)

v0 的核心洞察是：在类 Unix 环境里，**bash 是通往一切能力的 meta-interface**。读文件、写文件、搜索、运行测试、调用 git、本地脚本化——都可以通过 bash 完成。

因此 v0 只提供 1 个工具：

* `bash(command: str) -> stdout/stderr`

然后把系统提示词写得足够“工程化”（引导多用工具、少空谈、不要编造路径等），就已经能在不少任务上跑出可用效果。

#### 14.2.1 递归即子代理：用“进程隔离”换“上下文隔离”

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_70.5fl2vrldv5.webp)

v0 最精彩的地方是“子代理”实现几乎不要额外机制：**通过 bash 调用自身**：

```bash
python v0_bash_agent.py "explore src/ and summarize"
```

为什么这等价于 Sub-agent（至少在教学版里足够）：

* **进程隔离 = 历史隔离**：子进程有全新的 `history=[]`，不会把探索阶段的长输出污染主对话。
* **stdout = 汇报通道**：子代理只把最终总结打印出来，父代理把它当作工具结果读回。
* **递归 = 层级拆解**：复杂任务可以继续拆子任务（当然生产系统会对递归深度/预算做约束）。

v0 的代价也很明确：

* **安全边界非常弱**：子代理也能执行任意 bash（包括写文件/删文件）。
* **治理全靠提示词**：没有权限系统、没有 allowlist/denylist、没有审计与审批。

所以 v0 更像“证明题”：证明 Agent 的本质确实极小；但真正可用的工程系统必须继续加护栏。

### 14.3 v1：Model as Agent（4 工具 + workspace 护栏，把 v0 变成“可维护系统”）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_71.26lyz3xw8g.webp)

v1 的目标是把 v0 的“bash 万能”拆成更稳定的 4 个工具接口，原因是：

* **可控性**：读文件/写文件/编辑文件分别约束，行为更可预测。
* **可观测性**：工具层可以统一做路径检查、输出截断、超时、错误格式化。
* **更少幻觉**：模型不再需要“发明一条复杂 bash”来做精确编辑。

v1 的“四件套”基本覆盖 90% 编码任务：

| 工具 | 用途 | 工程护栏（推荐） |
| --- | --- | --- |
| `bash` | 跑命令：`rg`/`git`/`pytest`/`npm` | 超时、危险命令拦截、输出截断 |
| `read_file` | 读文件内容 | `safe_path`（不允许逃逸工作区）、可选行数上限 |
| `write_file` | 新建/整文件覆盖 | `safe_path`、自动建目录、显式返回写入摘要 |
| `edit_file` | 小范围“外科手术式”替换 | **exact match**（找不到 old_text 就报错） |

这里面最值得“抄作业”的是两类护栏：

1. **workspace 逃逸防护（safe_path）**：任何文件路径都 resolve 到 WORKDIR 下；`../` 逃逸直接拒绝。
2. **输出/成本控制**：工具结果截断（例如 50KB），避免一次 `cat` 把上下文打爆。

从第 13 章角度看，v1 让 Tool 层具备了“治理入口”：安全、超时、截断、错误处理可以在工具层统一做，而不是寄希望于模型每次都自觉。

### 14.4 v2：TodoWrite（把“计划”从模型脑内拖到台面上）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_72.96a8h0a92n.webp)

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_73.4qrtbqxuum.webp)

v1 最大的问题不是“不会写代码”，而是做长任务时容易出现 **context fade**：

* 前几轮说了要做 A→B→C，但十几个 tool call 后忘了做到哪一步；
* 执行顺序跳来跳去，用户也看不清进度；
* 完成定义不清晰，很难收敛到“可验收”。

v2 的解法极其朴素，但效果非常强：增加一个 Todo 工具（计划板），让模型 **必须在结构化状态机里推进任务**。

#### 14.4.1 Todo 的 schema：约束不是限制，而是脚手架

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_74.lw7zn0orx.webp)

learn-claude-code 的 TodoWrite 设计有几个关键约束（不是随意的）：

* **最多 20 条**：防止无限扩张的“愿望清单”。
* **最多 1 条 in_progress**：强制专注，避免并行导致的上下文漂移。
* **字段必填**：`content/status/activeForm` 三件套，保证可显示、可推进、可回放。

呈现上一般会渲染成这种“看得见的执行状态”：

```text
[ ] Refactor auth module
[>] Add unit tests <- Adding unit tests...
[ ] Update documentation
(0/3 completed)
```

对照第 13 章，“Plan 不是一段漂亮 prose”，而是一个要被 Orchestrator 强制维护的对象；TodoWrite 是最小的可运行版本。

#### 14.4.2 提醒（nag）机制：把“流程偏好”变成运行时反馈

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_75.8hgywzmq24.webp)

v2 还加了一个非常实用的工程小招：如果模型连续多轮没更新 Todo，就往上下文里注入 reminder（软约束），提示它回到“计划→执行→更新”节奏。

这类 reminder 属于低成本但高收益的 harness：不改变核心 loop，却显著减少“跑偏”。

### 14.5 v3：Task / Sub-agent（用上下文隔离解决“探索污染”）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_76.1vz55yio2z.webp)

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_77.8l0kupfsru.webp)

当任务变成“先探索大仓库 → 再做精细改动”，单一上下文会被探索阶段的海量细节占满。v3 的核心目标就是解决这类 **context pollution**。

v3 引入 Task 工具，让主代理把子任务派给“上下文隔离”的子代理：

* 主代理保留干净上下文，只接收子代理的**最终总结**；
* 子代理可以被赋予更窄的权限（工具白名单），例如 explore 只能读不能写。

#### 14.5.1 Agent 类型注册表：用“职责分离”引导模型行为

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_78.3rbpykv3oo.webp)

learn-claude-code 的教学版通常会定义三类子代理（也可以扩展）：

* `explore`：只读探索（`bash + read_file`），负责“找文件/理解结构/定位入口”。
* `plan`：只读规划（`bash + read_file`），负责“基于现状产出可执行方案/风险点”。
* `code`：全权限实现（全部工具），负责“落地修改与验证”。

这背后的设计意图不是“权限安全”这么简单，而是 **行为塑形**：当模型知道自己只有只读工具时，它更倾向于先把事实摸清楚，而不是边看边改。

#### 14.5.2 Task 的输入输出合约：子代理只交“可用结论”

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_79.13m9o822ci.webp)

一个实用的 Task 工具 schema 通常会包含：

* `description`：简短任务名（用于进度显示）
* `prompt`：对子代理的详细指令（含范围/输出格式/验收点）
* `agent_type`：选择 explore/plan/code（决定工具集与系统提示词）

执行时关键点是：**子代理使用独立 `sub_messages=[]` 跑同一个 agent loop**，跑完后只把最终文本返回主代理。

这就是第 13 章提到的 context quarantine：把“噪声/长日志/探索细节”关在子上下文里，只把“可复用的摘要与证据”带回主循环。

### 14.6 v4：Skills（把“知识”外化成可编辑文件，并按需加载）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_80.86u53u7hwh.webp)

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_81.175vlxv528.webp)

到 v3 为止，我们解决的是“能干活、能拆活”。但编码 Agent 很快会遇到另一个瓶颈：很多任务不是缺工具，而是缺 **做法**：

* 如何写一个 MCP server？
* 如何做系统化 code review？
* 如何处理 PDF/音频/图像等非文本输入？

这类“专家方法论”如果全塞进 system prompt，会把上下文撑爆；如果完全靠模型即兴发挥，会不稳定。v4 的 Skills 机制就是为此设计的：**工具管能力，技能管知识**。

#### 14.6.1 SKILL.md：一个可版本化、可分发的“知识包”

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_82.5xb4kcmrfj.webp)

最常见的 skills 目录结构：

```text
skills/
  pdf/
    SKILL.md
    references/
    scripts/
    assets/
  mcp-builder/
    SKILL.md
```

其中 `SKILL.md` 通常采用 YAML frontmatter 提供元数据（name/description），正文是详细步骤、命令模板、注意事项等。

#### 14.6.2 渐进式披露（progressive disclosure）：让 context 只为“当前任务”付费

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_83.1aphjno7rx.webp)

v4 把 skill 加载拆成三层：

* **Layer 1（始终加载）**：只加载 name + description（很短，用于路由/触发）。
* **Layer 2（触发时加载）**：加载 SKILL.md 正文（可能上千 token，但只在需要时注入）。
* **Layer 3（按需）**：再去读 references/scripts/assets（无限深，但按需取用）。

这基本是生产级 CLI Agent（包括 Claude Code/Kode 一类产品）在“技能/文档注入”上最常见的工程策略：让上下文保持 lean，同时保留无限扩展深度。

#### 14.6.3 缓存友好的注入方式：skill 内容走 tool_result，而不是 system prompt

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_84.6po0233d5o.webp)

learn-claude-code 特别强调一个容易被忽略的成本点：很多 LLM API 有 **prompt prefix cache**，但命中条件通常是“前缀完全相同”。因此：

* **不要**把动态状态/skill 内容写进 system prompt（会让缓存失效，成本可能飙升）。
* **要**把 skill 作为工具结果追加到 messages 尾部（前缀不变，缓存命中）。

这也是为什么 v4 的 Skill tool 会把 SKILL.md 内容作为 `tool_result` 返回，追加到对话历史里（而不是修改 system prompt）。

如果对这类成本差异敏感，可以把 learn-claude-code 的《上下文缓存经济学》当作必读补丁：一旦在历史中间插入/编辑/替换消息，成本与延迟都可能出现数量级变化。

### 14.7 从“教学版”到“可用版”：还差哪些工程补丁？

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_85.39lo9ztq3f.webp)

learn-claude-code 的 v0-v4 已经覆盖了 Claude Code 类 CLI Agent 的核心机制，但离“生产可用”通常还差一组工程化补丁。可以把它们理解为第 13 章 13.2（扩展面）里那些能力的“落地清单”：

1. **权限与沙箱（底线治理）**
   * `bash` 的 allow/ask/deny；对网络、写入、危险命令做分级审批；
   * workspace 逃逸、敏感文件规则、secret redaction。
2. **变更交付方式：从 write/edit 到 patch/diff**
   * 让模型产出 diff/patch，再由运行时应用（更可审计、更易 review/回滚）；
   * 默认先 `git diff` 再解释变更，形成“证据优先”的交付习惯。
3. **可观测与可回放（debuggable agent）**
   * 记录每次 tool_call 的输入/输出摘要、耗时、失败原因；
   * 保存 transcript，使一次 run 可回放、可复盘、可评测。
4. **最小验证闭环（不要只‘写完’，要‘验收’）**
   * 把“怎么验收”写进计划（Todo/Plan），并在结束前自动执行（例如 `pytest`/`npm test`/`go test`）。
5. **上下文工程（working set）**
   * 读文件前先 search；大文件按需截取；探索任务交给子代理；
   * 避免把“所有历史”当作上下文：用计划、日志、产物来承载状态。
6. **插件化与生态接入**
   * MCP servers / connectors；hooks；skills 分发与版本管理。

做到这里，基本就拥有了一套“Claude Code 风格”的可演进骨架：核心 loop 很小，但通过 Plan/Tools/Sub-agents/Skills/Permissions/Hooks 不断叠加工程约束，让它能稳定跑长任务、并且可控可审计。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_86.1zir3obqsa.webp)

---

## 15. Claude Code Alternatives: Gemini CLI, Codex, OpenCode

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_87.mkdc68gq.webp)

下面选取三个常被拿来对标 Claude Code 的 CLI 编码型 Agent：Gemini CLI、Codex CLI、OpenCode，并用同一套维度做横向对比（信息以各项目公开仓库/文档为准；推出时间按 GitHub 仓库首次公开粗略标注）。

* Claude Code: https://github.com/anthropics/claude-code
* Gemini CLI: https://github.com/google-gemini/gemini-cli
* Codex CLI: https://github.com/openai/codex
* OpenCode: https://github.com/anomalyco/opencode

| 维度 | Claude Code | Gemini CLI | Codex CLI | OpenCode |
| --- | --- | --- | --- | --- |
| 源公司/组织 | Anthropic | Google（google-gemini） | OpenAI | Anomaly（anomalyco） |
| 主要实现语言 | Shell/Python/TypeScript（Node 生态） | TypeScript（Node.js） | Rust | TypeScript |
| 开源程度 | **部分开源 / 源码可见但非开源**（LICENSE 为商业条款） | **完全开源**（Apache-2.0） | **完全开源**（Apache-2.0） | **完全开源**（MIT） |
| Model 绑定与限制 | 主要绑定 **Claude**（Anthropic 账户/API）；更换 provider 需借助第三方路由/代理 | 主要绑定 **Gemini**（Google 账号/API/Vertex）；强调 Gemini 2.5 Pro / 1M context | 主要绑定 **OpenAI**（ChatGPT 登录或 API key） | **Provider-agnostic**：Claude/OpenAI/Gemini/本地模型等（同一套交互切模型） |
| 工具与扩展面 | 终端 agent + 代码理解/写入 + git workflows；支持插件（commands/agents） | 内置 Search grounding、文件/命令/抓取工具；支持 MCP 扩展 | 本地运行；sandbox/审批；配置/skills；支持 MCP 与通知 hooks | 强 TUI + LSP；内置 build/plan agent 与 `@general` subagent；client/server 架构便于多客户端 |
| 功能丰富度（相对） | 高（产品化工作流与生态） | 中-高（“直达模型”+ 常用工具 + MCP） | 中-高（本地 agent + 安全治理 + 可配置） | 高（终端体验 + 多 provider + 多客户端） |
| 推出时间（约） | 2025-02 | 2025-04 | 2025-04 | 2025-04 |

补充：几个“设计取向”的差异（也决定了该怎么选）：

* **厂商第一方 vs 第三方**：Claude Code / Gemini CLI / Codex 属于模型厂商第一方工具，默认强绑定各自账号与模型；OpenCode 更强调 provider-agnostic，在模型快速迭代/价格波动时更容易“换发动机”。
* **扩展机制**：Claude Code 更偏“插件化工作流”（自定义命令/agents）；Gemini CLI 与 Codex 都把 MCP 作为对接外部能力的主入口（协议化、可组合）；OpenCode 既做 agent，也把终端体验（TUI/LSP）作为核心卖点。
* **运行时形态**：Codex 强调 sandbox/审批这类“副作用治理”；Gemini CLI 强调 Search grounding 与大上下文；OpenCode 的 client/server 更像把 agent runtime 做成可被不同前端驱动的服务。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_88.1aphjno7rs.webp)

## 16. $2B 的 Agent: Manus 解析

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_89.5fl2vrlduf.webp)

上一章我们把 Claude Code、Gemini CLI、Codex CLI、OpenCode 放在同一张表里对比，关注点是：当 Agent 被交付为 CLI 工具时，运行时、权限与扩展机制会决定它能跑多长、多稳、多可控。

但在真实工作流里，“终端 + 工作区”并不是唯一的默认执行环境。更多任务发生在浏览器、邮箱、IM 与各类 SaaS：跨站点、跨账号、跨格式，既要求过程可见（可接管/可审计），也要求产物可交付（文档/链接/自动化结果）。Manus 正是在这个位置上，把 Agent 从“给建议”推进到“把事做完”，并把它产品化成一个可持续运营的系统——也因此被市场用 **\$2B** 这样的叙事来标记其商业化潜力（这里的 **\$2B** 更像是对增长与护城河的预期，而不只是模型能力本身）。

本章不把 Manus 当作某个单点功能，而把它当作一套可运行、可扩展、可治理的 Agent Runtime 来拆解：任务如何建模、计划如何落地、工具如何编排、状态如何持久、失败如何治理、权限如何收束。我们将从产品定位与典型端到端任务入手，拆解 Cloud Browser / Browser Operator / Mail Manus / Collab 等能力面板，再用 Task / Plan / Tool / State / Observability 的视角归纳其架构要点，并补全 API 与生态接入方式，最后回到一个更现实的问题：为什么这种“闭环交付”的 Agent 能被卖到 $2B。

### 16.1 Manus 是什么：产品定位与典型端到端任务

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_90.102nqi8zme.webp)

Manus 是一种面向通用任务的自主 AI Agent，引入了 [从意图到行动闭环交付](https://www.baytechconsulting.com/blog/manus-ai-an-analytical-guide-to-the-autonomous-ai-agent-2025) 的全新范式。不同于传统对话式 AI（如 ChatGPT）只能提供答案，Manus 能自主计划并执行一系列操作，最终交付 [完整的工作成果](https://manus.im/tools)。其名称源自拉丁语“手”（Manus），寓意这个 AI 就像用户的 [数字双手](https://www.baytechconsulting.com/blog/manus-ai-an-analytical-guide-to-the-autonomous-ai-agent-2025)——不仅会给建议，更会亲自动手完成任务。从研究信息、编写代码到上线部署，Manus 致力于贯通任务执行的各个环节。

Manus 可以胜任跨领域的复杂任务，覆盖知识工作和数字化操作的各个方面。例如：

* 内容创作与报告：给定一个主题，Manus 能自主检索多源资料并撰写完整的[调研报告或文章](https://www.baytechconsulting.com/blog/manus-ai-an-analytical-guide-to-the-autonomous-ai-agent-2025)。又如处理长邮件或 PDF 文档，将其摘要后输出成要点或决策建议。
* 幻灯片与文档生成：用户只需提供要点，Manus 就能自动生成 PowerPoint 演示文稿，包括草拟内容、设计幻灯片版式，甚至添加[引人注目的图表](https://manus.im/zh-cn/tools)。整个过程耗时可能仅数分钟，且成果可编辑完善。
* 代码编写与应用部署：Manus 配备了代码生成和执行环境，能够根据自然语言要求编写程序、构建网页或小游戏，并将结果[部署到在线空间](https://www.helicone.ai/blog/manus-benchmark-operator-comparison)。例如，用户一句话描述游戏需求，Manus 可产出完整的浏览器游戏或克隆一个网站。
* 数据分析与可视化：Manus 可以读取用户提供的表格或数据库，通过内置的数据分析库处理数据，输出分析报告或[交互式图表](https://www.helicone.ai/blog/manus-benchmark-operator-comparison)。例如，自动生成销售数据的可视化图表并分析趋势。
* 在线事务处理：借助浏览器控制能力，Manus 能尝试代替用户执行网上事务，如[填写表单、预订服务甚至下单购买](https://medium.com/@jalajagr/inside-manus-the-anatomy-of-an-autonomous-ai-agent-b3042e5e5084)（尽管在实际使用中有时会受限于验证步骤）。这一能力展示了其从对话助手向数字代理的跃升。

简而言之，Manus 的典型工作流是：用户以自然语言描述目标，Manus 将此视作一个任务，自主拆解为子步骤计划，利用各种工具执行每一步，监控中间状态并迭代调整，最终将成果交付给用户（可参考 [Helicone 的机制拆解](https://www.helicone.ai/blog/manus-benchmark-operator-comparison)）。这种端到端的自动化，大幅减少了用户在不同应用之间来回手动操作的时间，使复杂任务的完成如同使用“一站式服务”般流畅（也可参考 [Integrations 文档](https://open.manus.im/docs/integrations/integrations)）。

### 16.2 Manus 能做什么

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_91.5j4othegk5.webp)

与 Claude Code 这种 CLI Agent 不同，Manus 是一个『通用』Agent：它不把“工作区/终端”当作唯一入口，而是把浏览器、邮箱与协作平台当作默认执行面，把任务拆成“能交付的结果”而不是“可运行的命令”。给出目标后，它会在云端/本地拉起合适的环境，读取附件与数据源，选择工具执行，并把中间产物沉淀为可追踪的文件、链接与日志。

更具体地说，这个『通用』体现在三个层面：

* **跨场景**：从调研与写作、幻灯片与文档生成，到数据分析、代码生成与部署，再到需要登录态的网页操作与跨系统事务处理；
* **跨介质**：网页、PDF/Office、邮件线程、表格、代码仓库、IM 消息等都可能是输入/输出的一部分；
* **跨触点**：不仅在网页 UI 里对话，还能通过浏览器插件、邮件、Slack 等渠道触发与交付（后面 16.4 会展开）。

同时，Manus 的“通用”也依赖于连接器/集成层：通过 [Integrations](https://open.manus.im/docs/integrations/integrations) / [Data Sources](https://open.manus.im/docs/integrations/data-sources) 以及 [Zapier](https://zapier.com/apps/manus/integrations/slack) 这类工作流平台，它可以把任务接到已有的业务系统里，比如 Notion 的知识库、Google Calendar 的日程，乃至 Stripe 这类支付与账务系统。这样交付就不止是“生成一段内容”，而是对真实系统进行创建/更新/通知等可执行动作。

为了把“通用能力”落到可用的产品形态，Manus 在能力面板上做了几类典型的执行通道（不同通道对应不同的权限边界与用户参与度）：

* [Cloud Browser](https://open.manus.im/docs/features/cloud-browser)：在隔离的云端浏览器里执行跨站点操作，界面实时可见、可接管；适合需要长链网页流程、且不想污染本地环境的任务。
* [Browser Operator](https://open.manus.im/docs/features/browser-operator)：通过浏览器扩展在本地浏览器里“借用现成登录态”执行操作，省去重复登录与验证；但也意味着需要更严格的授权与确认。
* [Mail Manus](https://open.manus.im/docs/features/mail-manus)：把任务放进邮件工作流——转发/抄送即可触发，附件天然携带；适合异步处理长文档、报表与审批类任务。
* [Collab](https://open.manus.im/docs/features/collab)：把任务与产物放到团队协作语境里，让“谁在做/做到了哪/交付了什么”可共享、可审计，便于 review 与交接。

如果把这些“产品入口与执行通道”抽象掉，Manus 的本质仍然是一套 Agent Runtime：Task 如何建模、Plan 如何演进、Tool 如何选择与编排、State 如何持久、以及 Observability 如何支撑可控与可靠——这也是下一节要用运行时视角拆解的部分。

### 16.3 Agent Runtime 视角：Task / Plan / Tool / State / Observability

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_92.4g4zilimoj.webp)

第 13/14 章已经把 Orchestrator/Plan/Tool/Loop 的通用机制讲清楚了，这里不再重复定义，而是把 Manus 的产品形态映射到运行时的五个维度（参考 [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)）。

* Task（任务）：API 层的 Task 是顶层执行单元（prompt + 附件 + 元数据），有唯一 ID/状态/时间戳等；运行时由顶层的 [Executor Agent](https://www.helicone.ai/blog/manus-benchmark-operator-comparison) 负责接住请求并推进执行（另见 [Manus API](https://open.manus.im/docs/api-reference)）。
* Plan（计划）：复杂任务会被 [Planner Agent](https://www.helicone.ai/blog/manus-benchmark-operator-comparison) 拆成 UI 可见的 to-do 列表；执行中按新信息与偏差持续插入/修改步骤，保证可对齐、可推进。
* Tool（工具）：工具层包含沙盒内置能力（浏览器/终端/Python/文件等）+ 通过 [Integrations](https://open.manus.im/docs/integrations/integrations) 接入的第三方服务；每步“选工具→执行→回填结果”把推理闭环拉回现实。
* State（状态）：每个任务运行在隔离沙盒与工作空间里，沉淀中间工件（文件/数据/代码）供多步复用与最终交付；长任务用分层记忆/多 Agent 分工缓解遗忘与噪声。
* Observability（可观测性）：用户侧暴露计划、当前步骤、工具操作与中间结果；系统侧记录结构化日志，并可通过 [Webhooks](https://open.manus.im/docs/integrations/webhooks) / [Slack 集成](https://open.manus.im/docs/integrations/slack-integration) 推送状态变化，同时配合重试与误差预算等治理策略（见 [Observability for Manus 1.5 Agents: Logs, Retries, Error Budgets](https://skywork.ai/blog/ai-agent/observability-manus-1-5-agents-best-practices/)）。

### 16.4 API 与生态接入：Projects / Tasks / Webhooks + Slack / Zapier / Data Sources

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_93.2dp6ujk1n5.webp)

除了网页 UI，Manus 也提供可程序化调用的 [Manus API](https://open.manus.im/docs/api-reference)：用 Project 管理默认配置与权限边界，用 Task 承载一次次“从输入到交付”的执行；附件与中间/最终工件以 File 形态沉淀，并可通过 [Webhooks](https://open.manus.im/docs/integrations/webhooks) 把状态变化与结果就绪推送回系统。这样 Manus 更像一个可嵌入业务流程的“交付引擎”，而不只是一个独立 App。

除了直接调用 API，Manus 还提供开箱即用的工作流集成，方便把触发与交付放进团队日常工具链（可参考 [Integrations](https://open.manus.im/docs/integrations/integrations)）：

* [Slack 集成](https://open.manus.im/docs/integrations/slack-integration)：团队安装后可在 Slack 聊天中直接 @ 调用 Manus 完成任务。Manus 能读取线程上下文和共享文件，根据指令执行任务，并把结果发回同一线程，供成员共同查看和调整。
* [Zapier 集成](https://zapier.com/apps/manus/integrations/slack)：通过 Zapier 连接大量应用，把 Manus 作为工作流中的智能节点。例如“表格新增记录 → 触发 Manus 汇总分析 → 把摘要发到 Slack”。
* [数据源集成（Data Sources）](https://open.manus.im/docs/integrations/data-sources)：在任务中直接获取实时外部数据（如财经市场、社交媒体趋势、内容平台热门信息等），开箱即用，无需用户单独管理 API 密钥。
* [MCP Connectors](https://open.manus.im/docs/integrations/integrations)：通过 OAuth 授权连接 Gmail、Notion、Stripe、HubSpot、Google 日历、GitHub、Google Drive 等应用，也支持开发者编写自定义 MCP 接入，把内部工具纳入 Manus 的行动空间。

### 16.5 为什么值钱：交付闭环、可靠性治理、分发与护城河

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_94.lw7zn0or4.webp)

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_95.491rn5wh8t.webp)

把 “\$2B” 的叙事拆成更可验证的变量，其实更像三件事：**效果（能不能把事做成）**、**收入（有没有真实付费）**、**团队（能不能在应用层持续迭代并把领先变成系统）**。

#### 16.5.1 效果：通用 Agent 断档领先

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_96.2rvmlesci3.webp)

在公开对比里，Manus 被描述为在 **GAIA benchmark**（面向真实任务的通用助手评测）上达到 SOTA，并在三档难度上超过 OpenAI Deep Research（见 [Helicone 的 benchmark 对比](https://www.helicone.ai/blog/manus-benchmark-operator-comparison)）。这类“真任务基准 + 端到端交付”的领先，是“断档领先”叙事最直接的依据：同样一句目标，系统更可能交付可用产物，而不是停在建议层。

#### 16.5.2 收入：可观 ARR（年化口径）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_97.39lo9ztq2y.webp)

相比“只有 demo、没有现金流”的产品，Manus 已经跑出可观的经常性收入量级。TechNode 报道其披露的 revenue run rate（RRR，年化口径）达到 **\$90M**，并指出其在 3 月上线付费服务（见 [Manus AI reports \$90 million annualized revenue](https://technode.com/2025/08/21/manus-ai-reports-90-million-annualized-revenue/)）。无论用 RRR 还是 ARR 来表述，这类指标的意义在于：价值主张足够强，用户愿意持续付费。

#### 16.5.3 团队：应用层能力（全球第一梯队）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_98.54y92m65oo.webp)

模型可以被追平，但把 Agent 做成“可交付、可控、可嵌入”的应用层系统更难：上下文工程、工具/连接器、沙盒与权限、可观测与可靠性治理、以及把多触点交付融进协作工作流（见 16.3/16.4）。Manus 团队在这些环节有持续的工程化输出与方法论沉淀（例如 [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) 与 [Observability for Manus 1.5 Agents: Logs, Retries, Error Budgets](https://skywork.ai/blog/ai-agent/observability-manus-1-5-agents-best-practices/)），也因此更接近“全球第一梯队”的应用层能力：能把一次领先快速产品化、规模化，并在可靠性与生态上持续复利。

综上，Manus 的估值更像是“效果领先 × 经常性收入 × 应用层团队能力”的乘积，而不是单点模型的溢价。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_99.54y92m65on.webp)

---

## 17. Manus Alternatives: agenticSeek, OpenHands, AgentGPT, UI-TARS-desktop

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_100.491rn5wh9k.webp)

如果把 Manus 理解为“通用任务的闭环交付 Agent”，那么开源/替代方案大多是在某个维度上做取舍：要么更强调本地隐私，要么更聚焦软件工程（SWE），要么专注 GUI/浏览器操作。下面给一个足够简洁的对比。

| 项目 | 形态/定位 | 强项 | 主要短板 | 适合谁 | 参考 |
| --- | --- | --- | --- | --- | --- |
| agenticSeek | 本地优先的通用 Agent（定位为 “local Manus alternative”） | 100% 本地运行与隐私、带 web browsing/coding、多 agent 规划 | 产品化与稳定性、生态/集成深度通常弱于商业产品 | 想要“Manus 方向 + 本地隐私”的个人/团队 | https://github.com/Fosowl/agenticSeek |
| OpenHands | 面向软件开发的 Agent（SDK/CLI/GUI/Cloud） | 工程化的 SWE 流程、CLI/GUI 都齐、公开 benchmark（如 SWEBench） | 主要聚焦开发任务，不是“通用办公+跨 SaaS 交付” | 把 Agent 当“开源 Devin/Jules”用来写代码/改仓库 | https://github.com/OpenHands/OpenHands |
| AgentGPT | 浏览器里配置/运行自治 Agent（偏玩法与演示） | 上手快、可视化地跑“任务→子任务”循环 | 能力边界偏早期 AutoGPT 路线，闭环交付/治理能力有限 | 想快速体验/教学演示“自治 Agent 是怎么跑的” | https://github.com/reworkd/AgentGPT |
| UI-TARS-desktop | 桌面端 GUI Agent（本地/远程 computer & browser operator） | 多模态 UI 操作能力强，适合“像人一样点点点” | 重点在 GUI/浏览器操作，不等同于通用交付型 Agent | 需要 UI 自动化/浏览器 Operator 的场景 | https://github.com/bytedance/UI-TARS-desktop |

补充一条务实信息：开源协议与自托管限制也会影响选型——`agenticSeek`/`AgentGPT` 为 GPL-3.0，`UI-TARS-desktop` 为 Apache-2.0，`OpenHands` 核心为 MIT（`enterprise/` 目录单独授权）。

一个朴素的选型法：
* 要“本地隐私的通用 Agent”→ `agenticSeek`
* 要“开源软件工程 Agent（写代码/修 bug/改仓库）”→ `OpenHands`
* 要“UI/浏览器 Operator（GUI 自动化）”→ `UI-TARS-desktop`
* 要“快速演示自治循环”→ `AgentGPT`

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_101.et047ejc6.webp)

---

## 18. 总结

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_102.9kgo7vijxh.webp)

从模型视角看，LLM 仍然是 next-token predictor：它擅长在给定上下文里“生成最可能的下一句”，但不天然具备长期记忆、稳定执行与对外部事实的强约束。Chatbot 把它包装成对话产品，而 Agent 则把它放进一个可运行的执行系统里——用工具获得 grounding，用循环与状态跑长任务，用评测与可观测把不确定性纳入治理。

本文的主线可以压缩成三句话：

1. **把推理落到现实**：从 Tool Calling/MCP，到 ReAct 的 Thought/Action/Observation，再到 DeepResearch 的分层编排与证据体系，核心都是让“结论有来源、行动可回填”，用可验证的 Observation 对抗 hallucination。
2. **把长任务做成系统**：Plan + Agent Loop 把目标拆成可推进步骤；State/Checkpoint/Resume 让执行可恢复、可回放、可审计；上下文工程用 working set、外置记忆与子任务隔离对抗 context overflow 与噪声传播。
3. **把不确定性变成可运营**：权限与 sandbox 收束副作用；最小验证闭环（tests/校验器）让“完成”有可执行定义；离线 golden tasks + tracing 把迭代变成可回归的工程流程，把上线变成可监控的系统。

落到工程实践，Agent 的“公式”大致是：

* **Policy（LLM）**：负责决策与生成候选方案；
* **Runtime（Orchestrator/Loop）**：负责循环、预算、停止条件与重规划；
* **Tools & Environment**：负责真实世界 I/O 与可验证证据；
* **State & Memory**：负责持久化、工件与跨步骤连续性；
* **Governance（permissions/observability/evals）**：负责可控、可审计与可运营。

最后回到“交付形态”：LangChain/LangGraph 提供构建原语与可恢复运行时，高层框架与 flow 平台提供更快的工程入口；CLI Agent 把“工作区 + 工具 + 权限”变成默认执行环境；而 Manus 这类产品进一步证明，护城河往往在应用层系统能力（上下文工程、工具生态、可靠性与协作交付），而不是某个单点模型。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_103.3yexu0h943.webp)

## Refs

- [1] [Function calling | OpenAI API](https://platform.openai.com/docs/guides/function-calling)
- [2] [Structured model outputs | OpenAI API](https://platform.openai.com/docs/guides/structured-outputs/key-ordering?api-mode=chat)
- [3] [Model Context Protocol (MCP) | OpenAI Developers](https://developers.openai.com/apps-sdk/concepts/mcp-server)
- [4] [Connectors and MCP servers | OpenAI API](https://platform.openai.com/docs/guides/tools-connectors-mcp)
- [5] [Completions | OpenAI API Reference](https://platform.openai.com/docs/api-reference/completions)
- [6] [Chat Completions | OpenAI API Reference](https://platform.openai.com/docs/api-reference/chat?locale=en)
- [7] [Function calling and other API updates - OpenAI](https://openai.com/index/function-calling-and-other-api-updates/)
- [8] [Assistants migration guide | OpenAI API](https://platform.openai.com/docs/assistants/migration)
- [9] [Deprecations | OpenAI API](https://platform.openai.com/docs/deprecations)
- [10] [Why we built the Responses API](https://developers.openai.com/blog/responses-api)
- [11] [Migrate to the Responses API | OpenAI API](https://platform.openai.com/docs/guides/migrate-to-responses)
- [12] [Reasoning models | OpenAI API](https://platform.openai.com/docs/guides/reasoning)
- [13] [Conversation state | OpenAI API](https://platform.openai.com/docs/guides/conversation-state)
- [14] [completions-responses-migration-pack | GitHub](https://github.com/openai/completions-responses-migration-pack)
- [15] [OpenAI Responses API: The Ultimate Developer Guide | DataCamp](https://www.datacamp.com/tutorial/openai-responses-api)
- [16] [Manus API - Manus Documentation](https://open.manus.im/docs/integrations/manus-api)
- [17] [Overview - Manus API](https://open.manus.im/docs/api-reference)
- [18] [Browser Operator - Manus Documentation](https://open.manus.im/docs/features/browser-operator)
- [19] [Meta Acquires Autonomous Agent Developer Manus AI, Marks Its Fifth Deal in 2025 | Gadgets 360](https://www.gadgets360.com/ai/news/meta-acquisition-manus-ai-agent-developer-fifth-deal-2025-10129781)
- [20] [Inside Meta's Groundbreaking Acquisition of Manus | AI Magazine](https://aimagazine.com/news/how-manus-puts-meta-ahead-in-the-agentic-ai-economy)
- [21] [Meta buys Manus for $2 billion to power high-stakes AI agent race | TechRadar](https://www.techradar.com/pro/meta-buys-manus-for-usd2-billion-to-power-high-stakes-ai-agent-race)
- [22] [What Meta’s Acquisition of Manus Means for the Future of AI Agents | All About AI](https://www.allaboutai.com/ai-news/what-meta-acquisition-of-manus-means-for-the-future-of-ai-agents)
- [23] [Manus (AI agent) | Wikipedia](https://en.wikipedia.org/wiki/Manus_%28AI_agent%29)
- [24] [GitHub - langchain-ai/deepagents](https://github.com/langchain-ai/deepagents)
- [25] [Deep Agents CLI | LangChain Docs](https://docs.langchain.com/oss/python/deepagents/cli)
- [26] [Deep Agents overview | LangChain Docs](https://docs.langchain.com/oss/python/deepagents/overview)
- [27] [GitHub - langchain-ai/deepagents-quickstarts](https://github.com/langchain-ai/deepagents-quickstarts)
- [28] [GitHub - agno-agi/agno](https://github.com/agno-agi/agno)
- [29] [agno-agi/agno-docs | DeepWiki](https://deepwiki.com/agno-agi/agno-docs)
- [30] [Examples and Applications | agno-agi/agno-docs | DeepWiki](https://deepwiki.com/agno-agi/agno-docs/5-examples-and-applications)
- [31] [Deep Research Agent - Agno](https://docs.agno.com/examples/use-cases/agents/deep_research_agent_exa)
- [32] [GitHub - shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
- [33] [Manus AI: An Analytical Guide to the Autonomous AI Agent 2025](https://www.baytechconsulting.com/blog/manus-ai-an-analytical-guide-to-the-autonomous-ai-agent-2025)
- [34] [Manus AI Agent Toolkit for Delivering Work](https://manus.im/tools)
- [35] [用于交付工作的 Manus AI Agent 工具包](https://manus.im/zh-cn/tools)
- [36] [What is Manus AI? Benchmarks & How it Compares to Operator and Computer Use](https://www.helicone.ai/blog/manus-benchmark-operator-comparison)
- [37] [Inside Manus: The Anatomy of an Autonomous AI Agent | by Jalaj Agrawal | Medium](https://medium.com/@jalajagr/inside-manus-the-anatomy-of-an-autonomous-ai-agent-b3042e5e5084)
- [38] [Integrate Manus with Your Existing Tools - Manus Documentation](https://open.manus.im/docs/integrations/integrations)
- [39] [Cloud browser - Manus Documentation](https://open.manus.im/docs/features/cloud-browser)
- [40] [Mail Manus - Manus Documentation](https://open.manus.im/docs/features/mail-manus)
- [41] [Manus Collab - Manus Documentation](https://open.manus.im/docs/features/collab)
- [42] [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- [43] [Overview - Manus API (API Reference)](https://open.manus.im/docs/api-reference)
- [44] [Get the Cheat Code on Long-Running AI Agents—Here's What Manus, Google, and Anthropic Learned After Trial and Error + 12 Prompts to Help Build Long-Running Agents Yourself](https://natesnewsletter.substack.com/p/i-read-everything-google-anthropic)
- [45] [Slack Integration - Manus Documentation](https://open.manus.im/docs/integrations/slack-integration)
- [46] [Overview - Manus API (Integrations)](https://open.manus.im/docs/integrations/integrations)
- [47] [Observability for Manus 1.5 Agents: Logs, Retries, Error Budgets](https://skywork.ai/blog/ai-agent/observability-manus-1-5-agents-best-practices/)
- [48] [Webhooks - Manus API](https://open.manus.im/docs/integrations/webhooks)
- [49] [Manus Slack Integration - Quick Connect - Zapier](https://zapier.com/apps/manus/integrations/slack)
- [50] [Data Sources - Manus Documentation](https://open.manus.im/docs/integrations/data-sources)
- [51] [Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models | arXiv](https://arxiv.org/abs/2305.04091)
- [52] [ReWOO: Decoupling Reasoning from Observations for Efficient Augmented Language Models | arXiv](https://arxiv.org/abs/2305.18323)
- [53] [Reflexion: Language Agents with Verbal Reinforcement Learning | arXiv](https://arxiv.org/abs/2303.11366)
- [54] [Model Context Protocol (MCP) - Anthropic](https://www.anthropic.com/news/model-context-protocol)
- [55] [modelcontextprotocol/specification | GitHub](https://github.com/modelcontextprotocol/specification)
