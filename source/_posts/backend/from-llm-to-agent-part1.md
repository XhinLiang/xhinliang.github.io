---
title: 从 LLM 到 Agent（上）：原理、局限与演进脉络
date: 2026-02-25 11:39:16
tags: [Agent,LLM,MCP,ReAct,DeepResearch,AI]
categories: Backend
toc: true
---

## 前言

2025 年结束，Agent 这个概念在所有科技行业从业者中每天都会听到 10 次以上。那到底什么是 Agent，为什么有了 LLM 不够，还要做 Agent，我们是怎么从 LLM 演进到 Agent 的？

本文尝试从下面两个方面解释清楚上面的问题：
* 理解 LLM/Chatbot 的结构性局限：为什么“会答”不等于“会做事”
* 搞清 Tool Calling / MCP / ReAct / DeepResearch 的演化脉络：推理如何与证据、行动与产物形成闭环

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_1.mkdc68kf.webp)

本文 @coauthor: ChatGPT DeepResearch, Codex, Manus, AnyGen, Nano Banana 

## 0. 背景简介

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_2.9ddgcfweku.webp)

近两年，LLM 的“能对话”已被验证，但把它变成“能完成复杂任务的执行系统”仍是工程难题。本文试图从模型原理出发，解释 Chatbot 的结构性局限，并梳理从 Tool Calling、MCP 到 Agent 框架与运行时的演化脉络，最终落到工程实践的关键能力：状态、持久化、上下文工程、可观测与可控执行。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_3.39lo9ztq6d.webp)

## 1. LLM 的工作原理：token 预测

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_4.5trimmtosk.webp)

大语言模型的核心是 **next-token prediction**：给定上下文 token 序列，输出下一个 token 的概率分布，并以此自回归生成后续文本。该机制决定了两个关键事实：

1. 模型输出本质上是**条件概率采样**的结果；
2. 模型“理解/推理”能力在接口层面表现为对概率分布的调制，而非显式执行符号规则。

推理阶段的关键组成：

* **Context window**：模型可“看到”的输入长度上限；上下文越长并不必然越好，过长会导致注意力稀释、相关信息被淹没。
* **KV cache**：自回归生成中复用注意力的 key/value，以降低重复计算成本并提升吞吐。
* **Prompt / System prompt**：通过指令与约束塑造角色、风格与边界；系统提示词在对齐层面通常优先级更高。
* **Instruction tuning**：在预训练之后进行监督微调（SFT）以提升指令遵循；再通过 **RLHF / DPO** 等方法进行偏好对齐（alignment），降低有害输出并改善可用性。
* **幻觉（hallucination）与不确定性**：模型会生成“高似然但不保证真实”的内容；其不确定性来自数据覆盖、上下文缺失以及采样策略，概率分布并不等价于可校准的事实置信度。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_5.4g4zilimrn.webp)

## 2. Chatbot 的局限性

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_6.45i5pg3ema.webp)

仅靠一次或少数几次模型调用，Chatbot 在长流程任务上存在结构性瓶颈，典型体现在：

* **Hallucination / grounding**：模型缺少对外部事实的强约束；没有检索与校验时，容易用“合理文本”填补空白。
* **Non-determinism**：温度采样、并行计算与外部工具结果都会带来波动；对生产流程而言，这意味着输出不稳定与回归困难。
* **Context overflow / context saturation**：上下文有限且“越长越贵”；当历史对话、文档与中间产物不断累积，模型会出现注意力分散、遗漏关键约束等问题。
* **Long-term memory 缺失**：模型调用天然是“无持久记忆”的；跨天/跨会话任务需要外部记忆体（数据库、向量库、文件系统等）。
* **Long-horizon error accumulation**：多步骤任务中，早期小错误会被后续步骤放大；纯文本链式推理缺少强校验机制。
* **Tool reliability 与 I/O 边界**：真实世界 I/O 存在权限、网络、速率限制、格式变更与失败重试；模型本身无法保证这些边界条件被正确处理。
* **Statefulness vs stateless**：模型单次调用是无状态的；而任务执行需要状态机（进度、依赖、产物、失败恢复）。
* **规划能力/执行能力割裂**：模型可以写出漂亮计划，但在执行时容易忽略约束、误用工具或无法闭环验证。
* **观察-行动闭环（observe-act loop）缺失**：没有“执行→观察结果→修正策略”的闭环时，模型难以稳定完成需要反馈迭代的任务。

举个例子：在服务器上配置 Nginx 时，若依赖 Chatbot 协助，通常需要手动把现有配置复制粘贴到对话里，再让模型给出下一步命令；执行后把输出贴回，模型再继续判断。整个过程是“人手工搬运上下文 + 执行 + 反馈”的反复循环。

Agent 的价值在于把上述瓶颈外置为工程可控的系统能力：

* 用工具获得**grounding**（检索/数据库/计算/运行代码/读写文件）；
* 用状态与循环控制长任务（step budget、检查点、恢复）；
* 用人类审核兜底高风险动作（human-in-the-loop）；
* 用日志与评测把不确定性纳入治理（可观测性与回归测试）。

继续用 Nginx 的例子：若使用 Agent，它可以主动读取配置文件与服务状态，先给出修改计划并请求人工审批；随后按计划执行命令、落盘、测试与 reload。中间若报错（语法校验失败、端口冲突），它能基于真实输出定位问题并迭代修复。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_7.3k8i358ybl.webp)

## 3. Tool Calling

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_8.3yexu0h96o.webp)

我们回到 Chatbot。在 2022 年的 Chatbot 中，Chatbot 不知道当前时间，不知道现在的天气，也很难计算 123456 * 987654，也很难数清楚 "strawberry" 里有几个 r。
为了解决这个问题，业界就想出了 Tool Calling 的方案，这个方案初期是由 OpenAI 实现的 Function Calling，后续集成到各种 LLM 基础模型中，最后演变成现在的 MCP 方案。

### 3.1 Function Call: 模型感知能力

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_9.5c1gy1sb7j.webp)

OpenAI 于 2023 年发布 Function Calling 能力：开发者以 JSON schema 描述函数签名，模型可输出可执行的参数结构，由系统调用真实工具（数据查询、业务 API、计算等），再将工具结果回填继续生成。([OpenAI][7])

**Function Calling** 把“动作”从文本里抽离出来，成为可校验、可执行的结构化输出：模型决定“是否调用、调用哪个工具、参数是什么”，系统负责执行与回填结果，再由模型整合为最终响应。([OpenAI Platform][1])

```python
tools = [{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "查询某城市天气（示例工具）",
    "parameters": {}, # ...
  },
}]

r = client.chat.completions.create(
  model="gpt-4.1-mini",
  messages=[{"role": "user", "content": "查一下新加坡天气。"}],
  tools=tools,
)

tc = r.choices[0].message.tool_calls[0]
args = parse_json(tc.function.arguments)  # => {"city":"新加坡"}

tool_result = {"city": args["city"], "temp_c": 31}

r2 = client.chat.completions.create(
  model="gpt-4.1-mini",
  messages=[
    {"role": "user", "content": "查一下新加坡天气。"},
    {"role": "tool", "tool_call_id": tc.id, "content": to_json(tool_result)},
  ],
)
print(r2.choices[0].message.content)
```

通过 Function Call，Chatbot 初步具备了跟外界交互的能力，此时只要 Prompt 写得还不错，上述的时间、天气等问题可以得到很好地解决。

### 3.2 MCP：把工具生态标准化

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_10.5c1gy1sb8e.webp)

在 OpenAI 的 Function Call 取得初步成功后，各家 LLM 和 LLM 框架迅速跟进，支持了类似 Function Call 的能力。
在各家 LLM 基础模型中，Function Call 的实现思路一致，但接口各有差别。这就导致工具生态被“各家 SDK 语义”割裂：schema 表达、调用返回格式、流式事件、错误处理都不统一，开发者需要为不同厂商重复适配。

2024 年 11 月，Anthropic 结合自己对 Function Call 的理解，设计并开源了 MCP 的方案。这个方案把“模型 ↔ 工具/资源”的交互抽象为统一协议，通过标准化的 server、工具定义与资源访问，降低多模型/多工具集成成本，并让工具生态具备可移植性。

需要区分的是：MCP 是 Anthropic 提出的协议；OpenAI 等平台对 MCP 的支持，属于“在自身工具体系中接入/托管 MCP server”的实现方式（如 remote MCP servers / connectors）。([Anthropic][54]; [GitHub][55]; [OpenAI Developers][3])

**MCP（Model Context Protocol）**旨在标准化“模型如何连接外部工具与资源”。MCP server 对外暴露可调用工具（含 schema），并返回结构化结果；在 OpenAI 平台中，开发者可通过 **remote MCP servers** 或 OpenAI 维护的 **connectors** 赋予模型新的能力。([OpenAI Developers][3])
在工具治理上，MCP/connector 体系强调：工具调用可设置为自动允许或需要显式审批，以满足安全与合规要求。([OpenAI Platform][4])

下面用“查询当地天气”的极简示例说明 MCP 的用法：MCP server 暴露 `get_weather` 工具，模型决定是否调用，运行时负责执行并把结果回填。

```python
from mcp import Client
from openai import OpenAI

mcp = Client("http://localhost:8080")   # 本地天气 MCP server
tools = mcp.list_tools()                # => [{"name": "get_weather", ...}]
client = OpenAI()

r = client.responses.create(
  model="gpt-4.1-mini",
  input="查一下我当地天气",
  tools=tools,                           # 让模型可用 MCP 工具
)
tc = r.output[0]                         # 运行时解析出 tool call
weather = mcp.call_tool(tc.name, tc.arguments)  # 实际调用 MCP server
final = client.responses.create(
  model="gpt-4.1-mini",
  input=[{"role": "tool", "content": weather}],
)
print(final.output_text)
```

### 3.3 CoT & Thinking Model

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_11.4g4zilimsd.webp)

在 LLM 的早期阶段，提升推理能力的常见做法是用 CoT（Chain-of-Thought）提示词“扶一把”：在问题前加一句 “Think step by step / 逐步思考”，或给出带推理过程的示例，让模型把中间步骤展开，从而显著提升数学、逻辑与多步问题的正确率。这个阶段的改进主要依赖提示工程与输出格式控制，效果可观但稳定性有限。

后来 OpenAI 干脆把这个 CoT 的能力内置到 o1 模型中，然后把这个模型称为 Thinking Model 或者 Reasoning Model。

Thinking Model 使得模型不只是“会答”，而是把更多计算预算花在推理过程上，通过更长的内部思考链与过程监督（process supervision）提升复杂任务的正确率与稳健性。由此衍生出“Thinking Model”的基本概念：在推理时显式分配思考预算，模型倾向于先规划再验证，必要时调用工具完成外部校验，但对外仅输出精炼结论与可审计的中间结果。

这套思路很快被行业吸收并扩散。DeepSeek-R1 以“推理优先”的训练路径与蒸馏版本打开了开源推理模型的供给，随后各家厂商陆续发布带 “Reasoning/Think” 标签的模型族系，强调在数学、编程、规划与多步工具调用上的可靠性。整体趋势是：**从参数规模扩展转向“推理时算力扩展”**，以更高的推理预算换取更高的任务成功率；同时也带来延迟与成本的权衡，因此生产系统常会引入推理路由与预算控制。

下面是当前 OpenAI API 的标准调用示例：

```python
r = client.responses.create(
  model="gpt-5.1",
  input="用 3 条 bullet 总结一下 OpenAI Responses API 的核心变化，并给出一个使用场景。",
  reasoning={"effort": "medium"},
  tools=[{"type": "web_search"}],  # 需要时模型会自己决定是否调用
)
```

此时不难发现，Thinking Model 虽然最终输出结果还是跟 Chatbot 类似的文本，但其已经基本具备了 Agent 的基本形态和能力。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_12.6f168xo53w.webp)

## 4. 从 ReAct 到 DeepResearch：Agent 思维范式的演化

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_13.3gow5ffvmj.webp)

如果说 Thinking Model 让模型“更会思考”，那么 ReAct 让模型“会做事”，而 DeepResearch 则把这套能力扩展成“可交付的研究系统”。这条演化路径的核心，是让推理与证据、行动与产物形成闭环。

### 4.1 ReAct：把推理与行动打通的最小闭环

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_14.5fl2vrldy2.webp)

ReAct（Yao et al., 2022）将推理（Reasoning）与行动（Acting）交织：在每轮中显式组织 **Thought / Action / Observation**，用工具获得外部证据，再基于观察继续推进。它强调“先行动获取事实，再基于事实推理”，工程价值主要体现在：

* 将“推理链”拆成可执行片段，天然适配工具调用与错误恢复；
* 以 Observation 作为对抗 hallucination 的基本单元；
* 以轨迹（trajectory）形式沉淀过程数据，便于评测与回放。

ReAct 的短板也很明确：它仍是单体循环，容易被长上下文拖垮；当任务跨度变大、证据来源增多时，需要更强的编排与证据治理能力，这正是 DeepResearch 的切入点。

### 4.2 DeepResearch

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_15.3k8i358yc8.webp)

与 Thinking Model / ReAct 相比，DeepResearch 更接近“分层研究系统”，而非单一对话。它不是模型能力的单点提升，而是一套面向研究交付的运行时方法论，典型流程包括：

* **问题分解与研究计划**：明确范围、问题边界与验收标准，再拆成可并行的子问题。
* **分层编排与并行执行**：用 orchestrator 下发任务，由 sub-agent 进行检索、实验或数据整理。
* **Context quarantine**：对子代理上下文做隔离，避免噪声与错误传播；汇总阶段做证据对齐与一致性检查。
* **Source attribution**：将“结论—证据—来源”绑定，形成可审计的引用链。
* **File-system as memory**：用工作区承载中间产物与数据集，降低对单次上下文的依赖。
* **Report synthesis**：以“报告 + 附件数据”为目标输出，而非聊天式回复。

对比 Non-Thinking Model、Thinking Model、ReAct、DeepResearch：

| 维度 | Non-Thinking Model | Thinking Model | ReAct | DeepResearch |
| --- | --- | --- | --- | --- |
| 目标 | 基础对话与文本生成 | 更强的单体推理 | 可靠执行与工具闭环 | 研究级交付与证据体系 |
| 结构 | 单轮生成 | 单模型思考 | 思考-行动-观察循环 | 分层编排 + 多子代理 |
| 证据 | 无显式证据 | 可选调用工具 | Observation 驱动 | 结论-证据-来源绑定 |
| 上下文 | 单上下文、短对话 | 单上下文增长 | 逐步追加 | 隔离 + 汇总 |
| 产物 | 文本回答 | 精炼答案 | 过程结论 + 工具结果 | 报告 + 附件数据 |
| 适用场景 | 写作/FAQ | 复杂推理题 | 多步执行任务 | 长周期研究/情报汇总 |

相关实现可参考 Agno 的 Deep Research Agent 示例。([docs.agno.com][31])

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_16.8ok6sf8vkn.webp)

## 5. 混沌初期的试探：AutoGPT 

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_17.6bhkb7v2du.webp)

从实现层面看，上面介绍的 DeepResearch 已具备工业级 Agent 的关键结构。但从用户体验看，它的交付形态偏单一（主要是一份报告）。日常工作往往需要“能动手”的系统，因此我们关注更狭义的通用 Agent：除了产出文字，还能改文件、跑命令、调用工具完成任务。

为了简化表述，后文的“Agent”默认指通用 Agent。

在 2023 年，随着 ChatGPT 引爆公众关注，社区很快冒出了一个『奇葩』项目 AutoGPT。它把 LLM 包装成一个“自驱执行体”：设定目标、生成计划、调用工具、写入记忆，再继续循环。示例里常见的能力包括搜索、写文、发 Twitter 等，给人的直观感受是“模型开始自己动手了”。

但 AutoGPT 的本质不是算法突破，而是一种工程式拼装：Prompt 结构化 + 工具调用 + 简易记忆。它让人第一次看到“Agent Loop”的雏形，却也暴露了早期的混沌：计划质量不稳定、状态与记忆缺乏约束、任务容易发散并陷入错误循环，成本与结果不可预期。

即便如此，AutoGPT 仍是重要的里程碑：它点燃了开源 Agent 的浪潮，让“自主循环”成为可讨论的工程形态，也让人们开始认真思考状态管理、预算控制、工具可靠性等后续必须解决的问题。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_18.96a8h0a95d.webp)

## 6. Plan-and-Solve / Reflection：把“隐式思考”变成“显式控制”

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_19.96a8h0a95c.webp)

在 ReAct 架构之外，人们在 Agent 开发层面还有一些别的探索，我们以 Plan-and-Solve 和 Plan-and-Execute 举例。

值得说明的是，Plan-and-Solve / Plan-and-Execute / ReAct 并不是互斥的关系，在业界实践中经常是互补的关系。工程师们经常会在 Agent Loop （后文中会提到）中混合使用这几种架构，以优化不同的任务的 Agent 效果。

### 6.1 Plan-and-Solve / Plan-and-Execute

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_20.6bhkb7v2dp.webp)

Plan-and-Solve 这类“先规划、后求解”的套路，最早以 prompting 形式被系统化提出，用来改善 zero-shot CoT 的稳定性与正确率。([arxiv.org][51]) 在 Agent 工程里，同样思路常被扩展为 Plan-and-Execute：计划不只描述推理步骤，还要显式包含工具、输入输出与验收方式，便于执行器逐步落地并回填观察；更结构化的分层实现可参考将“推理计划”与“工具观察”解耦的 ReWOO。([arxiv.org][52])
Plan-and-Solve 通过显式计划降低长任务漂移：

* **Decomposition**：先分解为可验收的子目标与里程碑（milestone）；
* **Plan validation**：对计划做约束检查（权限、依赖、成本预算、可执行性）；
* **Executor loop**：按步骤执行，必要时查询工具、写入工作区；
* **Step budget / iteration limit**：限制最大步数与成本，避免失控；
* **Backtracking**：在关键失败点回溯到上一个可用 checkpoint，重规划或切换策略；
* **Tool plan**：将“用哪些工具、输入输出、预期失败模式”写入计划，使执行器具备可操作性。

### 6.2 Reflection / Reflexion（自我纠错与记忆）

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_21.2ksepz6764.webp)

Reflection（反思）是一类通用的自我纠错策略；而 Reflexion 则更强调把“失败—归因—改进”写入可检索记忆，在下一轮尝试中显式触发策略更新与重试。该思路在 Reflexion 论文中被系统化提出。([arxiv.org][53])
Reflection 强调把失败经验转为可复用策略：

* **Self-critique**：对结果与过程做自检（是否满足需求、是否引用充分、是否越权/越界）；
* **Retry with feedback**：将错误信息与失败原因反馈给模型，触发有条件重试；
* **Critic model**：用独立评审模型或规则系统做质量门禁；
* **Post-mortem & episodic memory**：将失败案例沉淀为可检索的“情景—原因—修复”记录，形成面向任务域的策略记忆。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_22.3d5a7pmswb.webp)

## 7. Agent 的核心概念：Agent Loop

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_23.3ns40v211m.webp)

截至目前（2026 年初），Agent 的开发已经变得非常成熟，衍生出很多很多成熟的框架和落地实践。

在业界经验中，一个可运行的 Agent 至少包含：**状态（State）**、**环境（Environment）**、**策略（Policy）**与**停止条件（Stop condition）**，并在预算约束下循环迭代。

最小闭环可表述为：

* **Observation**：读取环境与工作区（用户输入、文件、工具结果、事件日志）。
* **Decision**：选择下一步动作（直接回答 / 调工具 / 更新计划 / 请求人工确认）。
* **Action**：执行工具调用或写入产物。
* **State update**：更新进度、上下文工作集与记忆摘要。
* **Stop**：达成目标、超出预算、遇到不可恢复错误或需要人工介入时停止。

以 Claude Code 举例，工程落地时常见的最小组件集通常是：

* **Tool registry**：工具清单、schema、权限与路由规则；
* **Scratchpad / workspace**：中间推理结构与产物存放区（含可审计日志）；
* **Iteration limit / budget**：token、工具调用次数、时间、成本上限；
* **Checkpoint / resume**：关键状态可持久化，支持中断恢复与回放。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_24.4g4zilimrs.webp)

## 8. Agent 工程核心：状态、持久化、上下文工程

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_25.54y92m65s7.webp)

上一节把 Agent loop 拆成了“观察-决策-行动-更新-停止”。要把这个循环变成可交付的工程系统，核心不在模型有多聪明，而在运行时是否**可恢复、可审计、可控成本**。这背后有三块最关键的工程骨架：

* **状态**：明确“当前任务到哪一步了”，避免隐式对话漂移；
* **持久化**：让执行过程可回放、可恢复，防止一次失败就全盘重来；
* **上下文工程**：保证有限 token 内放对的信息，其余外置。

### 8.1 状态模型：把“隐式对话”变成“显式运行时”

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_26.et047ejeu.webp)

Agent 工程的首要问题不是“能否回答”，而是“能否稳定执行”。可操作的抽象需要把系统状态显式化，并保证可验证的更新路径。常见的状态划分：

* **对话状态（Conversation state）**：用户目标、约束、历史交互与已确认的决策。
* **任务状态（Task state）**：计划/里程碑、待办、已完成步骤、失败原因与重试记录。
* **环境状态（Environment state）**：工作目录、文件变更、外部系统（API/DB）副作用、权限与工具可用性。

把状态落实到 schema / state machine 能带来三类收益：

1. **可恢复**：长任务中断后可从确定的节点继续；
2. **可审计**：工具调用与变更路径可追溯；
3. **可控**：每一步的输入、输出、预算与停止条件可被约束。

实践中还需要两条额外约束：

* **状态变更要可验证**：状态更新尽量通过结构化 action 完成，而不是自由文本；必要时加校验规则（如“未完成的步骤不能被标记完成”）。
* **状态版本化**：长周期任务会演进，schema 需要可迁移，避免“旧状态无法恢复”。

状态 schema 示例（JSON 结构）：

```json
{
  "version": "1.0",
  "conversation": {
    "goal": "更新 Nginx 配置并 reload",
    "constraints": ["不改动 upstream", "需要审批 reload"],
    "decisions": ["已确认使用 /etc/nginx/nginx.conf"]
  },
  "task": {
    "plan": ["读取配置", "生成补丁", "语法校验", "reload"],
    "current_step": 2,
    "status": "running",
    "errors": []
  },
  "environment": {
    "cwd": "/etc/nginx",
    "dirty_files": ["nginx.conf"],
    "tool_permissions": ["read_file", "write_file", "shell"]
  }
}
```

### 8.2 Durable execution：Checkpoint、Resume 与事件日志

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_27.1e93hdhake.webp)

长任务的失败往往来自“累积误差 + 外部 I/O 不确定性”。工程上需要把执行过程当作**可持久化的事务**：

* **Checkpointing**：在关键步骤保存状态快照（输入、工具参数、工具结果、下一步节点）。
* **Resume / Replay**：支持从某个 checkpoint 继续，或回放历史步骤复现问题（time-travel debugging）。
* **Event log（事件溯源）**：以“事件”记录状态变更，便于回放与审计；必要时可做幂等键（idempotency key）避免重复副作用。

在图/状态机式运行时里，checkpoint 往往以“每个 super-step 生成快照”的方式实现，从而天然支持人类介入、回放与容错。进一步提升稳定性还需要：

* **副作用分类**：把工具调用分成只读/可重复/不可逆三类，对不可逆动作强制审批或双写审计。
* **结果重放优先**：恢复时优先使用历史工具结果，避免因外部环境变化导致“同一步不同结果”。
* **补偿动作**：对可逆的副作用设计补偿路径（例如撤销改动、回滚配置），让失败不必从头来。

checkpoint 记录示例（事件 + 快照）：

```json
{
  "checkpoint_id": "ckpt_2025-01-12T10:14:03Z",
  "step": "syntax_check",
  "state_ref": "state_v1.0",
  "input": {"file": "/etc/nginx/nginx.conf"},
  "tool_call": {"name": "shell", "args": ["nginx", "-t"]},
  "output": {"ok": true, "stdout": "syntax is ok"},
  "next_step": "reload",
  "timestamp": "2025-01-12T10:14:03Z"
}
```

### 8.3 上下文工程：Working set 管理比“塞满上下文”更重要

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_28.77e1qo4qt8.webp)

上下文工程的目标是：在固定 token 预算内，维持正确性与任务连续性。有效方法不是“塞满上下文”，而是**分层与取舍**。常见策略：

* **Working set**：把当前决策必要信息维持在短上下文；其余内容外置。
* **Summarization / compression**：对历史对话、检索结果、日志做结构化压缩；明确“保留事实/结论/待办/风险”。
* **文件系统作为记忆**：将长文本、代码片段、表格与中间产物写入 workspace，再按需读回（避免 context overflow）。
* **Context quarantine**：对子任务使用隔离上下文（子代理/子图），主代理只接收“可交付摘要 + 引用路径”。

更工程化的做法通常会显式设定“上下文预算”：

* **固定槽位**：系统规则、任务目标、当前计划、关键证据四类信息优先保留。
* **引用而非复制**：长文档只保留摘要与路径/行号引用，模型需要时再读回。
* **准入门槛**：只有通过“相关性 + 可靠性”校验的信息才进入工作集，避免垃圾上下文污染决策。

### 8.4 Memory、缓存与成本控制

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_29.7ps8rsdyz.webp)

* **短期记忆**：最近 N 轮消息、当前待办与里程碑。
* **长期记忆**：向量库/语义检索、跨会话偏好与项目知识；需要“写入门槛”（避免把噪声固化）。
* **缓存**：对稳定工具结果（如依赖解析、目录树）做缓存；对不可缓存工具（实时数据）标记 TTL。
* **成本预算**：同时约束 token 与 tool call 次数；在并发场景下要设置 max_concurrency，防止工具风暴与费用失控。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_30.et047ejei.webp)

---

## 9. Agent 扩展迭代：效果评估、可观测性

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_31.mkdc68jd.webp)

前面几章讨论了 Agent 的运行时骨架：工具、循环、状态、持久化与上下文。把这些能力拼起来，确实能“跑起来”；但在真实生产里更难的是第二步：**如何证明它变好了、出了问题如何复现、上线后如何监控与止血**。这就落到两个问题：

* **效果评估（Evaluation）**：把“好不好”变成可重复的指标与回归门禁；
* **可观测性（Observability）**：把一次 run 变成可追踪、可回放、可定位的轨迹数据。

### 9.1 “效果”到底是什么：结果、过程与边界

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_32.8l0kupfstz.webp)

Agent 的“效果”不是单轮回答质量，而是一个端到端执行系统的综合表现。更可操作的定义通常包含三层：

* **结果层（Outcome）**：最终交付物是否满足验收标准（正确性、完整性、格式、业务约束）。
* **过程层（Process）**：是否走了合理路径（工具是否用对、是否出现无效循环、是否能自我纠错与恢复）。
* **边界层（Safety/Policy）**：是否遵守权限与合规边界（敏感操作是否审批、敏感信息是否泄露、不可逆副作用是否可控）。

工程上建议在需求阶段就写清“验收条件”，否则评估只能停留在主观印象。对可自动验证的任务，优先把验收条件写成测试/校验器；对开放式任务，再考虑 rubric 或人工抽检。

### 9.2 指标体系：从“成功率”到“可运营”

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_33.232d1e4tkj.webp)

一套能落地的指标通常会同时覆盖质量、效率与风险，并能分解到 task/step/tool 三个层级：

* **任务级（Task-level）**
  * **成功率**：按验收条件通过的比例（success rate），最好区分 hard fail / soft fail。
  * **质量分**：rubric 打分或自动评分（例如信息覆盖、格式一致性、引用完整性）。
  * **交付时间**：端到端 wall time、以及用户等待时间（含人类审批等待）。
  * **成本**：tokens、tool call 次数、外部 API 费用、以及峰值并发。
  * **人工介入率**：需要人工修正/补充上下文/重复审批的比例。
* **步骤/工具级（Step/Tool-level）**
  * **工具成功率**：按工具类型统计错误率、超时率、重试次数与平均延迟。
  * **恢复能力**：失败后能否通过重试/回溯/checkpoint 恢复（recovery rate）。
  * **无效循环**：重复读同一文件、反复调用同一工具、计划不推进等（loop rate）。
* **安全与合规（Safety/Policy）**
  * **高风险动作占比**：不可逆工具调用、写生产配置、删库等（需结合权限模型）。
  * **越权/越界事件**：被拦截的调用、敏感信息命中、policy violation 次数。

这些指标的价值在于：一旦能够稳定采集，就能做趋势、对比、回归与告警，把“模型不确定性”纳入工程治理。

### 9.3 离线评测：Golden tasks、回放与回归门禁

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_34.1vz55yio4r.webp)

Agent 的离线评测最好像软件测试一样分层组织，而不是只跑几条 demo：

* **Golden task suite（固定任务集）**：把真实任务抽象成可复现样本（输入、初始文件/数据、工具集合、验收标准）。
* **Record/Replay（录制/回放）**：将外部 I/O（搜索/API/DB）结果录制为 fixture，回放时优先使用历史结果，避免评测被外部变化污染。
* **多次运行看分布**：同一任务建议跑多次（不同 seed/温度/并发），关注成功率与方差，而不是单次结果。
* **自动验收优先**：能用测试、lint、schema 校验、diff 校验解决的，不要用主观评审；开放题再用 rubric 或 LLM-as-judge（并保留抽检）。
* **回归门禁（gating）**：把关键指标阈值写进 CI（成功率不降、成本不升、风险事件不增），让“改 prompt/改工具/改策略”都可回归。

评测的终点不是“跑出一个分数”，而是形成稳定的反馈闭环：每一次线上事故与失败 case，都能沉淀进 task suite，变成下一次的回归用例。

### 9.4 可观测性：把一次 run 变成可调试的系统

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_35.64ecfs8wx6.webp)

可观测性的目标是：**任何失败都能被复现与定位**。对 Agent 而言，日志不只记录错误栈，更要记录“决策轨迹”：

* **Trace（链路追踪）**：用 `run_id/thread_id/step_id/tool_call_id` 串起一次执行中的模型调用、工具调用、状态更新与 checkpoint。
* **事件日志 + 快照**：记录关键事件（Observation/Decision/Action/ToolResult/StateUpdate）以及必要的状态快照，支持 time-travel debugging。
* **输入输出治理**：对 prompt、工具参数、工具结果做脱敏/截断/哈希（既能排障，又不泄露敏感信息）。
* **指标面板与告警**：把 9.2 的指标落到 dashboard（成功率、成本、延迟、工具错误、循环率），并为突发退化设置告警与自动降级策略。

实践中一个常见的经验是：**把“可回放”当作第一原则**。只要 run 能 replay，很多“偶发问题”就会变成可定位的确定性问题；而这依赖运行时记录足够多的结构化轨迹数据。

![](https://cdn.jsdmirror.com/gh/XhinLiang/picx-images-hosting@master/20260114/image_w1376_h768_image_36.232d1e4tk8.webp)

## 结语

读到这里，你大概已经意识到：从 LLM 到 Agent，并不是“把 Chatbot 接上几个工具”这么简单。真正的分水岭在于：我们是否把推理放进一个**可执行、可验证、可回放**的闭环里——让模型从“会答”，走向“能交付”。

如果你准备把 Agent 落到真实业务里，不妨先把问题问得更工程一点：

* 这个任务的**验收标准**是什么？哪些可以自动校验，哪些必须人工把关？
* 失败是常态：需要怎样的**重试/回滚/checkpoint**，才能让长流程不崩盘？
* 上线以后怎么运营：**指标、日志、告警、权限边界**是否足够清晰？

当这三件事有了答案，Tool Calling / MCP / ReAct / DeepResearch 才会从“概念名词”变成“系统能力”；否则，Agent 只是在更大的上下文里，讲出更像样的故事。

回到文章开头的疑问：为什么有了 LLM 还要做 Agent？因为“会答”解决的是信息表达，“会做事”解决的是任务交付。而 Agent 的本质，就是把交付做成系统：每一步都能执行、能验证、能复盘。

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