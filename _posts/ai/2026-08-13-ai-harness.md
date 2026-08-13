---
title: "AI Harness：给大模型装上躯干、感官与手脚"
description: ""
date: 2026-08-13 16:00:00 +0800
categories: [AI]
tags: [AI, Agent, LLM, Harness, MCP]
math: true
---

> **TL;DR**：模型是智能体的"大脑"，Harness 则是它的"躯干、感官、手脚和规章制度"。2026 年，这个曾经的行话已经落地为一套可以逐项采购、逐项自建的技术栈——本文不满足于列概念，而是把每个组件拆到"零件级"，给出对应的协议、产品与配置示例。

在 AI 智能体（AI Agent）与大语言模型（LLM）落地的过程中，**Harness（驾驭系统 / 驭缰工程）** 正在成为 AI 工程界最核心的技术范式之一。2026 年，Thoughtworks 的 Birgitta Böckeler 在 martinfowler.com 发表了长文《Harness Engineering for Coding Agent Users》[^1]，学界也出现了《Natural-Language Agent Harnesses》[^2] 与《Code as Agent Harness》[^3] 两篇专门研究——**Harness Engineering 已经从一句行话，长成了一门有协议、有工具、有论文的工程实践。**

简单来说，**AI 应用的 Harness 是指围绕大语言模型构建的整套运行环境：控制回路、工具连接器、记忆管理、安全沙箱与可观测性基础设施。**

如果把大模型（Model）比作智能体的"大脑"，那么 **Harness 就是智能体的"躯干、感官、手脚，以及必须遵守的规则"**。系统的通用架构公式如下：

$$\text{Agent (智能体)} = \text{Model (推理大模型)} + \text{Harness (驾驭环境)}$$

---

## 1. 为什么会出现 Harness？AI 工程范式的三次演进

要理解 Harness，需要回顾 AI 开发范式的三次重大变革：

| 阶段 | 核心技术范式 | 核心关注点 | 瓶颈与代价 |
| --- | --- | --- | --- |
| **1.0 时代** | **Prompt Engineering**（提示词工程） | 如何撰写更精准的 System / User Prompt，把模型的最佳回答"套"出来 | 模型只能"说"不能"做"，无法处理跨系统联动与复杂计算 |
| **2.0 时代** | **Context Engineering**（上下文工程） | 如何通过 RAG、滑动窗口、向量数据库，为模型注入准确的上下文知识 | 模型依然处于"被动问答"状态；长流程任务中容易丢失目标（熵增） |
| **3.0 时代** | **Harness Engineering**（驾驭工程） | 构建让 AI 能够**自主感知环境、执行工具、自我纠错、跨会话记忆与安全运行**的闭环环境 | 工程范式从"写代码 / 写 Prompt"转向"为 AI 构建并维护运行环境" |

### 模型与现实世界的"确定性矛盾"

大模型本质上是一个**概率性生成引擎（Stochastic Engine）**，而真实的软件系统与企业业务流程则要求**绝对的确定性（Deterministic System）**。直接让"裸模型"操作数据库、修改生产代码，极易引发无限循环、越权操作或破坏性错误。

可以把 Harness 看作一个"控制论调速器"（Cybernetic Governor）：它在概率性的模型推理与确定性的外部系统之间，加入约束与反馈。Böckeler 的描述是，Agent Harness 像调速器一样，**把前馈（Feed-forward）与反馈（Feedback）结合起来，将代码库持续调节到期望状态**[^1]。

从 2025 到 2026 年，这种思路逐渐以产品形态出现：Claude Code、OpenAI Codex、GitHub Copilot、Gemini CLI 等编码 Agent 都可以理解为 **Model + Harness** 的组合；Meta 也将类似的机制用于广告排序模型的自主实验（详见 §2.5）。模型能力之外，运行环境的设计会直接影响这类系统的行为。

---

## 2. AI Harness 的 6 大核心组件

不同产品的划分并不完全一致，下面的六部分可以用来理解一个较完整的 AI Harness：

```
+-----------------------------------------------------------------------+
|                             AI AGENT                                  |
|                                                                       |
|   +---------------------------------------------------------------+   |
|   |                      MODEL (Brain/LLM)                        |   |
|   +---------------------------------------------------------------+   |
|                                   | (Reasoning / Action Intent)       |
| ==================================|================================== |
|                           HARNESS LAYER                               |
|                                   v                                   |
|   +---------------------------------------------------------------+   |
|   | 1. Context & Memory       | 2. Feedforward & Sensors          |   |
|   |    - Context Compaction   |    - Structural Guides            |   |
|   |    - Long/Short Memory    |    - Linters & Verifiers          |   |
|   +---------------------------+-----------------------------------+   |
|   | 3. Execution & Sandbox    | 4. Guardrails & Permissions       |   |
|   |    - Docker / WASM / eBPF |    - Human-in-the-Loop            |   |
|   |    - MCP Tool Connectors  |    - Policy Enforcement           |   |
|   +---------------------------+-----------------------------------+   |
|   | 5. Observability & Tracing| 6. Evaluation & Regression        |   |
|   |    - Token & Cost Track   |    - SWE-Bench / Custom Evals     |   |
|   |    - Checkpoint & Resume  |    - Agent Benchmarking           |   |
|   +---------------------------------------------------------------+   |
+-----------------------------------------------------------------------+
```

### 2.1 上下文与记忆调度系统（Context & Memory Engine）

上下文窗口既是"工作台"，也是成本中心：更长的上下文意味着更高的延迟与 Token 账单，而且**长上下文中段的注意力会衰减**。因此 Harness 的第一项工作，是像操作系统管理内存一样管理上下文预算。

**短期工作记忆与上下文压缩（Context Compaction）**

* **触发机制**：Harness 实时统计 Token 用量，当接近窗口或预算阈值（例如总预算的 70%–80%）时，不再盲目"继续塞"，而是触发压缩。
* **压缩策略**：从简单粗暴的"整段摘要"，到**分层摘要（Hierarchical Summarization）**——把已完成的子任务折叠成一句结论，保留关键决策、报错与约束；更精细的做法是 **Context Editing**：不重写整个历史，只替换"变脏"的段落，保留仍有效的结构化信息。
* **工作区即状态**：在编码 Agent 里，文件系统本身就是最好的工作记忆——中间产物、TODO、已改文件列表都落在磁盘上，而不是全部塞回提示词。

**长期记忆持久化（Long-term Memory）**

跨会话记忆在 2026 年已经分化出几种成熟方案，它们解决的核心问题是：**写什么、存哪里、何时回填**。

| 方案 | 定位 | 记忆形态与写路径 | 特点 |
| --- | --- | --- | --- |
| **Mem0** | 轻量长期记忆服务 | 向量（可选知识图谱）；**写时**由 LLM 做 ADD / UPDATE / DELETE | 写路径清晰、延迟低，适合"用户偏好 / 事实"类记忆 |
| **Letta（MemGPT）** | 自管理记忆运行时 | 上下文内 memory blocks + core / archival 分层 | 模型自己决定读、写、迁移，最接近"操作系统" |
| **Zep / Graphiti** | 时序知识图谱记忆 | 图 + 向量，记录实体关系随时间变化 | 适合"谁在什么时候做了什么"这类关系型事实 |
| **LangMem** | SDK 式记忆 API | 可插拔后端 | 与 LangGraph 等编排框架集成方便 |

一个容易踩的坑：长期记忆一旦被污染（比如把一次错误的假设当成了用户偏好），会跨会话持续毒化输出。所以记忆系统同样需要**写入权限控制、时间衰减和人工纠错入口**，而不是"存了就不管"。

**有界子代理（Subagents）作为上下文防火墙**：与其把 50 个文件全部读进主上下文，不如派一个子代理在**干净、受限的新上下文**里做探索，只把结论摘要返回。这既省 Token，又把无关细节挡在主任务之外——2026 年的主流编码 Agent 普遍内置了这套机制。

### 2.2 前馈引导与反馈传感回路（Feedforward Guides & Feedback Sensors）

这是 Harness Engineering 中最精妙的设计。Harness 的控制逻辑可分为**事前引导**与**事后感知**两翼[^1]：

```
                +------------------------------+
                |    Harness Control Loop      |
                +------------------------------+
                               |
       +-----------------------+-----------------------+
       |                                               |
       v                                               v
[ Feedforward Guides ]                       [ Feedback Sensors ]
(事前引导：提高第一次就做对的概率)          (事后感知：捕获错误并指导 AI 自我修正)
  - 系统规范 (AGENTS.md / CLAUDE.md)           - 编译器与静态检查 (Linters)
  - 项目模板与代码库架构                       - 自动化单元测试 (Unit Tests)
  - 架构约束规则                               - 语义评测 (LLM-as-a-Judge)
```

**前馈引导（Guides）的载体**，在 2026 年已经标准化：

* `AGENTS.md`（Codex 生态）与 `CLAUDE.md`（Claude 生态）：仓库级持久约定，定义目录结构、命令、验证方式与禁区；支持在子目录嵌套，作用域越近越具体。Anthropic 的官方建议是**控制在 200 行以内**——太长的"规范书"会被模型选择性忽略。
* `.cursor/rules`、`SKILL.md` 等：把"何时该做什么"写成可发现、可继承的规则与流程。

**反馈传感器（Sensors）按计算方式分两类**，这正是该文「计算型 / 推理型」分类法的精髓[^1]：

* **计算型控制（Computational）**：确定性规则——编译检查、`mypy` / `pyright` 类型检查、`gofmt` 格式、`shellcheck`、单元测试。毫秒到秒级出结果，结论确定、可复现，因而常被接入自动反馈回路。
* **推理型控制（Inferential）**：语义审查——LLM-as-a-Judge 评审语气、合规性或业务逻辑。慢、贵、有方差，但能抓住"编译过了但做错了"的问题。

**Hooks：把传感器接到回路上的确定性执行器**。Claude Code、OpenAI Codex 等主流编码 Agent 都提供生命周期钩子。以 Claude Code 的 settings 为例：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "./scripts/agent-gate.sh" }
        ]
      }
    ]
  }
}
```

```bash
#!/usr/bin/env bash
# agent-gate.sh：每次文件修改后静默执行，失败就回灌诊断
# 从 stdin 读取 hook 输入（tool_name / tool_input / tool_response）
input=$(cat)

# 只跑快的那部分：lint + 相关单测
result=$(make check 2>&1)
if [ $? -ne 0 ]; then
  # 把裁剪后的诊断以 JSON 返回，harness 会注入模型的后续上下文
  jq -n --arg msg "$(printf '%s' "$result" | tail -c 2000)" \
    '{ "additionalContext": ("[Sensor] 构建失败，请先自愈：\n" + $msg) }'
fi
```

> 注：不同产品的字段名略有差异——Claude Code 的 `PreToolUse` 支持 `block` 决策，`PostToolUse` 可用 `additionalContext` / `updatedToolOutput` 回灌、用 `systemMessage` 提示用户；OpenAI Codex 的 hooks 则覆盖 `SessionStart`、`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`PermissionRequest` 等事件，可用退出码 + stderr 阻塞。**但"检测 → 裁剪 → 回灌 → 自愈"的回路结构完全一致。**

于是就有了你我都熟悉的场景——修改没通过时，Harness 不会把一坨堆栈直接甩给人类，而是整理成极简诊断让模型自我修正：

```text
[Harness Sensor Signal]
The changes you made to user_service.py broke 2 unit tests:
- test_user_login(): Expected HTTP 200, got 500.
Stack trace: TypeError: 'NoneType' object is not subscriptable at line 42.
Please self-correct the patch before presenting to the human.
```

绝大多数低级错误在到达人类审阅之前，就被 Harness 拦截并自愈。

### 2.3 工具集成与运行沙箱（Tool Interfacing & Execution Sandbox）

**MCP：工具与数据面的"USB 接口"**

模型上下文协议（MCP）基于 JSON-RPC 2.0，由 Host（智能体应用）、Client、Server 三层组成。核心原语是三个：

* **Tools**：模型可调用的函数，带 JSON Schema 描述；
* **Resources**：通过 URI 读取的数据（文件、数据库记录、API 对象）；
* **Prompts**：可复用的参数化提示模板。

当前规范（2026-07-28 版）在三个核心原语之外提供 **Elicitation**（服务端向用户索取信息），以及 **Tasks**（异步长任务：轮询、中途注入输入、持久化句柄）、**Skills over MCP**（通过 MCP 发现与消费技能）、**MCP Apps**（对话中内嵌图表 / 表单等交互 UI）等官方扩展[^4]——其中 Tasks 与 Elicitation 自 2025-11-25 版引入。传输层上，本地进程走 stdio，远程服务走 Streamable HTTP（旧版 SSE 已被取代）。

```json
{
  "name": "search_customer",
  "description": "按订单号查询客户信息，返回脱敏后的姓名与会员等级",
  "inputSchema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string", "description": "订单号" }
    },
    "required": ["order_id"]
  }
}
```

注意一个被反复强调的安全前提：**模型只看到工具名、描述和 Schema，真正的执行在 Server 端**。所以工具描述本质上是一份"不可信输入"，Schema 校验和最小权限要放在 Server 里做——规范也明确要求 Host 在调用任何工具前获得用户同意[^4]。

**执行沙箱：隔离是有层次的**

AI 生成的代码是否需要在宿主机外执行，取决于任务和信任边界。按隔离强度从低到高，常见选项如下：

| 隔离层 | 代表实现 | 隔离机制 | 适用场景 |
| --- | --- | --- | --- |
| 进程级 | seccomp / Landlock | 裁剪系统调用与文件权限 | 可信代码、本地自用 |
| 容器 | Docker / Podman | namespace + cgroup，共享内核 | 团队内部、需生态兼容 |
| 用户态内核 | gVisor（Sentry） | 拦截系统调用，应用态实现内核 | 容器之上更强的宿主防护（Modal 采用） |
| MicroVM | Firecracker / Kata / Cloud Hypervisor / libkrun | KVM 硬件虚拟化，独立 guest 内核 | 多租户、不可信代码（E2B / Vercel 采用） |
| WASM | WASI / Component Model | 字节码沙箱，无系统调用 | 纯计算、插件、边缘执行 |

容器常用于本地或团队内部环境；多租户、执行不可信代码的场景则更常采用 Firecracker 一类的 MicroVM。无论隔离层级如何，出网策略与密钥注入方式都是 Harness 的一部分：密钥通常通过运行时 secret 挂载提供，而不写入镜像或提示词。

**A2A：智能体之间的"合同"**。如果说 MCP 管"手脚怎么连工具"，A2A（Agent2Agent）则管"智能体之间怎么协作"：用 Agent Card 声明能力、用任务生命周期（提交 → 执行中 → 需输入 → 完成/失败）做异步协调。A2A v1.0 在 2026 年已是 Linux Foundation 治理下的稳定协议，与 MCP 互补[^5]——单体 Agent 内用 MCP，多 Agent 编排用 A2A。

### 2.4 安全治理与权限护栏（Guardrails & Permission Systems）

**三层权限规则**是当下编码 Agent 的常见模式：`allow`（自动放行）、`ask`（每次都问人）、`deny`（永不执行）。不少实现按 **deny → ask → allow** 的顺序求值，使禁止规则具有更高优先级。

```json
{
  "permissions": {
    "allow": ["Bash(npm test)", "Read(./src/**)"],
    "ask":   ["Bash(git push*)", "WebFetch"],
    "deny":  ["Bash(rm -rf *)", "Read(~/.ssh/**)", "Read(~/.aws/**)"]
  }
}
```

配套的还有**沙箱档位与人工审批**：OpenAI Codex 提供只读、工作区可写（默认）、全权限等 sandbox 档位，越出工作区的写操作、外网访问、危险命令都要单独审批；Claude Code 有对应的 permission modes 与 `disableBypassPermissionsMode` 开关。**最小权限原则（PoLP）** 在这里的工程表达是：Agent 只拿到"完成本任务所需的最小能力集合"，并且所有高危动作（删库、改主干、转账、发版）都走 Human-in-the-Loop 审批。

**别忘了间接提示注入（Indirect Prompt Injection）**。这是 OWASP LLM Top 10 的头号风险（LLM01:2025）：恶意内容不在你写的 Prompt 里，而是藏在网页、邮件、文档或工具返回值里，诱导 Agent 去执行额外动作。实用的纵深防御层次：

1. **工具输出即数据**：在系统提示中声明检索内容与工具输出只是数据，优先级低于系统规则；
2. **工具结果解析**：用结构化解析把"数据"与"指令"切开，丢弃注入的指令片段；
3. **出站白名单 + 权限最小化**：让被注入的 Agent 即使中招也无越权能力可动；
4. **输出过滤与审计**：对写出的内容做敏感信息 / 异常动作检测，全部操作留痕可回溯。

### 2.5 可观测性与断点续传（Observability & Checkpointing）

Agent 是一个"模型调用 + 工具调用"嵌套的分布式系统，调试必须靠**链路追踪**：每一次模型调用、每一次工具执行都是一个 span，Thought / Action / Observation 都挂在轨迹上，出问题才能定位"哪一步开始偏航"。

2026 年的主流方向已经清晰：**用 OpenTelemetry（API/SDK + OTLP 协议 + Collector）做统一的埋点与采集标准，用它仍在 Development 阶段的 GenAI 语义约定来定义 span 的属性 Schema**[^6]。不同操作类型的 span 按约定携带各自的 `gen_ai.*` 属性——模型调用与工具调用的属性集并不相同：

```text
# 模型调用 span：operation.name = "chat"
gen_ai.system              = "anthropic"
gen_ai.operation.name      = "chat"
gen_ai.request.model       = "claude-sonnet-4-5"
gen_ai.usage.input_tokens  = 12841
gen_ai.usage.output_tokens = 487

# 工具调用 span：operation.name = "execute_tool"
gen_ai.operation.name      = "execute_tool"
gen_ai.tool.name           = "bash"
gen_ai.tool.call.id        = "toolu_01Bx..."
```

上层可再对接 LangSmith、Langfuse（已提供原生 OTel 端点）、Arize Phoenix、Opik 等平台，查看 token 成本、延迟、错误率和失败轨迹。对于跨会话任务，按任务维度记录成本也有助于理解一次执行的资源消耗。

**断点续传（Checkpoint & Resume）**是长任务的硬需求。简单的做法是会话级 resume（Codex 的 `codex resume`、Claude Code 的 `--continue/--resume`）；生产级做法是**把状态外部化**——Meta 的 Ranking Engineer Agent（REA）是最好样本[^7]：

* 架构上分 **Planner**（与工程师协作制定实验计划，数据来自"历史实验洞察库 + 独立 ML 研究 Agent"的双源假设引擎）与 **Executor**（管理跨数天的异步训练任务）；
* 它**不持有**贯穿数周的上下文窗口，而是用 **Hibernate-and-Wake** 机制：训练任务启动后把状态快照持久化到外部存储并"休眠"，等作业完成再唤醒、重建上下文继续；
* 效果：广告排序模型精度 2 倍、工程人效 5 倍。

这类设计的共同点是：中间状态被放到文件、数据库或 artifact 等外部介质中，而不是完全依赖一段持续增长的上下文窗口。

### 2.6 评估与回归测试（Evaluation & Regression）

Harness 不仅运行 Agent，也评估 Agent。公开基准覆盖了不同侧面：**SWE-bench Verified**（500 个真实 GitHub issue 的代码修复）、**Terminal-Bench**（终端自主操作）、**τ-bench**（工具对话任务）、**OSWorld**（跨应用电脑使用）。一个稳定的规律是：**越接近生产现实的基准，分数越低**——前沿系统在纯代码基准上已相当高，但在 OSWorld 这类"真实环境 + 长流程"任务上仍有巨大缺口。

在实际工程中，除了公开基准，也可以通过固定模型、调整 Harness 的对照实验来观察其影响：

* 把"改 AGENTS.md / 加工具 / 调 Linter 规则 / 换沙箱"都当成一次 harness 变更；
* 用一组**自建 golden tasks**（你们仓库里最有代表性的 20–50 个真实任务）跑 A/B：任务成功率、平均步数、token 成本、人工介入次数；
* 这类评测也可以像单测一样接入 CI，作为 Harness 变更的参考信号。

模型版本、工具和规则都会变化，因此评测集也需要随着系统演进而维护。

---

## 3. Harness 工程的核心设计模式

### 模式一：基于代码约束的 Feedforward-Feedback 闭环

以现代编码 Agent（Claude Code、Codex、Cursor、Devin、Copilot）为例，闭环分三步：

1. **Feedforward**：介入项目时自动加载 `AGENTS.md` / `CLAUDE.md` / `.cursor/rules`，注入规范、目录结构与编码偏好；
2. **Action**：模型产出代码修改；
3. **Feedback**：§2.2 里的 hooks 静默跑测试与 Linter，失败就把诊断回灌给模型自愈，而不是甩给人类。

这套模式的进阶形态是 **Spec-Driven Development**：先用文档钉死"要做什么"，再让 Agent 在约束下实现。GitHub Spec Kit 给出了经典五阶段流水线（Constitution → Spec → Plan → Tasks → Implement，对应 `/constitution`、`/specify`、`/plan`、`/tasks`、`/implement`），AWS Kiro、BMAD、OpenSpec 等工具把同一思想做成了企业级变体。**规格文件 + 质量门禁，本质就是"把前馈与反馈都写进版本库"。**

### 模式二：Skill 化的可复用流程（Agent Skills）

"写好一次流程，跨会话、跨工具复用"在 2026 年有了开放标准——**Agent Skills**[^8]：每个技能是一个目录，`SKILL.md` 用 `name` + `description` 声明能力与触发条件，正文按需加载，可附带脚本与参考资料。

```markdown
---
name: deploy-check
description: 发布前执行部署安全检查清单。当用户准备发布或提到"上线"时使用。
---

# 部署前检查
1. 运行 `make test` 并确保全绿
2. 核对 `git diff main...HEAD`，确认无意外变更
3. 检查迁移脚本与回滚方案是否齐全
```

核心机制是**渐进式披露（Progressive Disclosure）**：会话开始时只加载技能的名字和描述（几乎不占上下文），模型判断需要时再读正文。一个技能可以在 Claude Code、Codex、Cursor、Copilot、Gemini CLI 等几十个宿主间通用。它和 MCP 的分工：**Skill 是"怎么做"的可复用知识，MCP 是"能做什么"的运行时能力。**

### 模式三：有界子代理与多智能体编排

让一个 Agent 单线程扛大任务，是上下文爆炸与注意力漂移的根源。2026 年的主流做法是**子代理隔离**：

* **Creator-Verifier（创作者-校验者）**：一个代理负责写，另一个带着"挑错"的目标审查——把"生成"与"验收"解耦；
* **并行专项审查**：安全、性能、风格三个子代理并行跑，各自只读自己的检查项，避免互相污染；
* **Agent Teams**：Claude Code 等产品支持多个独立会话以共享任务列表 + 点对点消息协作，配合 A2A 实现跨进程、跨产品的编排。

子代理的边界就是它的价值：**干净的新上下文 + 最小工具集 + 只返回摘要**，相当于给主代理装了一道上下文防火墙。

### 模式四：自然语言 Agent Harness（NLAH）

更进一步的问题是：Harness 的控制逻辑埋在 Controller 代码里，难检查、难迁移、难做消融实验。《Natural-Language Agent Harnesses》[^2] 提出把**运行级策略**写成可编辑的自然语言文档（NLAH），再由共享的 **Intelligent Harness Runtime（IHR）** 解释执行——把文档转成 agent 调用、交接（handoffs）、状态更新、验证门禁与产物契约。实验表明，IHR 执行的 NLAH 能达到与代码实现相近的任务表现，同时静态策略更短、模块可逐个消融分析。这与 `AGENTS.md` / `SKILL.md` 的实践互为印证：**Harness 本身正在变成一种可以被版本管理、被测试、被复用的"研究对象"。**

---

## 4. 概念边界：Harness vs. Framework vs. Prompt

| 维度 | Prompt Engineering | Agent Framework（LangChain / CrewAI 等） | Agent Harness |
| --- | --- | --- | --- |
| **定位** | 给模型的文本指令输入 | 从零开发 AI 应用的代码库 / SDK | **智能体的完整运行时与控制基础设施** |
| **工作方式** | 编写自然语言提示词 | 在 Python / TS 里调用封装好的 API | **编排上下文、沙箱、传感器、权限与反馈闭环** |
| **代表技术** | few-shot、CoT、system prompt | LangGraph、AutoGen、CrewAI | **AGENTS.md / Hooks / Skills / MCP / 沙箱 / Tracing** |
| **类比** | 乐谱 | 各种乐器组件 | **音乐厅声学、指挥台与录音设备** |
| **关注点** | "模型怎么想" | "代码怎么组装" | **"智能体在真实环境里如何可靠、安全地完成任务"** |

---

## 5. 结语

Harness 并不是又一个 Agent Framework，也不是一份固定的产品清单。它描述的是模型之外、围绕模型建立的运行环境：上下文如何被组织，工具如何被约束，执行结果如何回到下一轮推理，以及系统如何在长任务中保留状态。

不同产品在这些环节上的实现差异很大：有的将规则写在仓库文件中，有的放入运行时配置；有的依赖容器隔离，有的使用更强的虚拟化边界；有的以链路追踪回放失败过程，有的以评测集观察系统变化。MCP、A2A、Skills、Hooks、沙箱与 Tracing 也不构成一套必须同时采用的方案，它们只是分别落在这个运行环境的不同位置。

从这个角度看，Harness 关心的不是让模型"更聪明"，而是模型进入真实软件环境后，哪些信息可见、哪些动作可做、结果如何被检查，以及失败时系统如何继续运转。

---

### 参考资料

[^1]: Birgitta Böckeler, *Harness Engineering for Coding Agent Users*, martinfowler.com, 2026-04. <https://martinfowler.com/articles/harness-engineering.html>
[^2]: Linyue Pan, Lexiao Zou, Shuo Guo, Jingchen Ni, Hai-Tao Zheng, *Natural-Language Agent Harnesses*, arXiv:2603.25723, 2026-03. <https://arxiv.org/abs/2603.25723>
[^3]: Xuying Ning et al., *Code as Agent Harness: Toward Executable, Verifiable, and Stateful Agent Systems*, arXiv:2605.18747, 2026-05. <https://arxiv.org/abs/2605.18747>
[^4]: Model Context Protocol, *Specification (2026-07-28)*. <https://modelcontextprotocol.io/specification/2026-07-28>
[^5]: Agent2Agent Protocol (Linux Foundation). <https://a2a-protocol.org/>
[^6]: OpenTelemetry, *Semantic Conventions for GenAI*. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
[^7]: Meta Engineering, *Ranking Engineer Agent (REA): The Autonomous AI Agent Accelerating Meta's Ads Ranking Innovation*, 2026-03-17. <https://engineering.fb.com/2026/03/17/developer-tools/ranking-engineer-agent-rea-autonomous-ai-system-accelerating-meta-ads-ranking-innovation/>
[^8]: Agent Skills, open standard. <https://agentskills.io/>
