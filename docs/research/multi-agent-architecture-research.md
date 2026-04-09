# 多Agent架构研究报告

> 调研日期：2026-04-09
> 目的：为 Invest_Agent v2 项目的架构决策提供依据

---

## 一、核心概念辨析

### 1.1 Workflow vs Agent vs Multi-Agent

Anthropic 在 [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents/) 指南中做了关键区分：


| 类型              | 定义                 | 确定性 | 灵活性 |
| --------------- | ------------------ | --- | --- |
| **Workflow**    | LLM 和工具通过预定义代码路径编排 | 高   | 低   |
| **Agent**       | LLM 动态决定自己的流程和工具调用 | 低   | 高   |
| **Multi-Agent** | 多个 Agent 通过协调机制协作  | 最低  | 最高  |


**Anthropic 核心建议**：从最简单的方案开始，只在确实需要时才增加复杂度。很多场景下，优化单个 LLM 调用 + RAG 就够了。

### 1.2 Agentic System 的构成要素

一个完整的 Agentic System 通常包含：

- **LLM 推理内核**：思考和决策
- **Tool 调用**：与外部世界交互
- **记忆系统**：跨会话状态持久化
- **规划能力**：任务分解和执行排序
- **反思/验证**：自我检查和纠错

---

## 二、五大多Agent编排模式

### 2.1 层级式（Supervisor/Worker）

- **机制**：中央 Supervisor 分解任务、分派给 Worker、综合结果
- **优点**：清晰的控制流、易于审计和追溯、适合合规场景
- **缺点**：Supervisor 可能成为瓶颈、单点故障
- **适用**：复杂结构化问题、需要全局视角的任务
- **典型实现**：LangGraph Supervisor Pattern、Claude Code 的 Orchestrator-Worker

### 2.2 顺序式（Pipeline）

- **机制**：任务按预定义顺序流转，每步依赖前步结果
- **优点**：简单可预测、易于调试
- **缺点**：不灵活、无法并行、一步失败全链中断
- **适用**：文档处理、数据 ETL、内容生产流水线

### 2.3 并行式（Ensemble）

- **机制**：多个 Agent 同时工作，结果由中央组件聚合
- **优点**：高吞吐、可实现投票/辩论机制
- **缺点**：成本高（多次 LLM 调用）、聚合逻辑复杂
- **适用**：多视角分析、头脑风暴、对抗验证（Bull vs Bear）

### 2.4 事件驱动（Pub/Sub）

- **机制**：消息代理实现发布-订阅通信，复杂度从 O(n²) 降至 O(n)
- **优点**：松耦合、可扩展性好
- **缺点**：最终一致性、调试难度高
- **适用**：高频实时系统、异步工作流、主动警报

### 2.5 对等式（P2P/Mesh）

- **机制**：Agent 之间直接通信，无中央协调者
- **优点**：高弹性、无单点故障
- **缺点**：调试困难、通信复杂度 O(n²)
- **适用**：去中心化场景

---

## 三、单Agent vs 多Agent：实证决策框架

### 3.1 关键研究数据（2025-2026）


| 指标       | 单Agent | 多Agent    | 来源            |
| -------- | ------ | --------- | ------------- |
| 基准测试胜率   | 64%    | 36%       | Princeton NLP |
| 顺序推理性能   | 基准     | 下降 39-70% | Google/MIT    |
| 并行任务性能   | 基准     | 提升 80.9%  | Google/MIT    |
| Token 消耗 | 基准     | 多 56-150% | CrewAI 测量     |
| 响应延迟     | 2-4s   | 8-15s     | 行业平均          |
| 单次任务成本   | 基准     | 2-5x      | 行业平均          |
| 开发周期     | 1-3 天  | 2-4 周     | 行业平均          |
| 企业级扩展成功率 | —      | <10%      | 行业报告          |


### 3.2 选择单Agent的场景

- 顺序推理和线性工作流
- 上下文 < 30K tokens
- 工具数 < 10-15
- 对调试简单性和成本敏感

### 3.3 选择多Agent的场景

- 可并行化的子任务（多视角分析、数据采集）
- 工具数 30+ 需要专业化分工
- 需要对抗验证（Bull vs Bear 辩论）
- 单个上下文窗口放不下（200K+ tokens）
- 需要不同模型处理不同任务（成本/质量路由）

### 3.4 多Agent的主要风险

- 协调失败占故障的 37%
- 验证缺失占 21%
- 级联失败：错误放大 4.4-17.2x
- 幻觉传播：一个 Agent 的幻觉可能被其他 Agent 放大

---

## 四、主流框架对比

### 4.1 LangGraph（LangChain 生态）

- **架构**：基于有向图的编排（节点 = 动作，边 = 转换）
- **GitHub Stars**：48K（2026.3），生产部署领先约 40%
- **核心特性**：
  - 确定性控制流 + 条件路由
  - 内置状态持久化（Checkpointer）
  - LangSmith 可观测性
  - 支持 Python / TypeScript
  - 预构建 Supervisor / Swarm 库
  - Human-in-the-loop 支持
  - LangGraph Platform 托管部署（含 cron jobs）
- **2026 新特性**：Command API、Functional API（@task/@entrypoint）、langgraph-supervisor/swarm 包
- **缺点**：学习曲线陡峭、简单场景 boilerplate 多、与 LangChain 耦合
- **任务完成率**：91%（加验证节点后）
- **适合**：需要精确流程控制的生产系统

### 4.2 AutoGen（微软）

- **架构**：分层事件驱动，任务建模为 Agent 间的异步对话
- **GitHub Stars**：37K
- **核心特性**：灵活对话模式、事件驱动、支持人类参与
- **缺点**：非 Python 团队学习成本高、更偏学术研究
- **适合**：研究密集型对话模式

### 4.3 CrewAI

- **架构**：角色制团队模型，灵感来自人类组织
- **GitHub Stars**：29K
- **核心特性**：直觉化的角色/目标/任务模型、最快原型速度
- **缺点**：复杂长周期工作流能力不足、生产稳健性较弱
- **适合**：快速原型验证

### 4.4 OpenAI Agents SDK（原 Swarm）

- **架构**：基于 Handoff 的轻量级方案
- **五个原语**：Agent / Tool / Handoff / Guardrail / Tracing
- **核心特性**：极简抽象、内置追踪和评估、与 OpenAI 深度集成
- **缺点**：仅支持 OpenAI 模型、供应商锁定
- **适合**：全 OpenAI 技术栈项目

### 4.5 Claude Agent SDK（Anthropic）

- **架构**：Orchestrator-Worker 模式
- **核心理念**：
  - 父 Agent 显式选择子 Agent 的模型层级（Haiku/Sonnet/Opus）
  - 子 Agent 运行在隔离上下文中，防止"上下文腐化"
  - 基于文件系统的 Mailbox 通信（最简协议）
- **内部测试**：多Agent系统在复杂任务上优于单Agent 90%+
- **适合**：Claude 生态项目

### 4.6 框架对比总结


| 维度    | LangGraph | AutoGen | CrewAI | OpenAI SDK | Claude SDK |
| ----- | --------- | ------- | ------ | ---------- | ---------- |
| 成熟度   | ★★★★★     | ★★★★    | ★★★    | ★★★★       | ★★★★       |
| 学习曲线  | 陡峭        | 中等      | 平缓     | 平缓         | 平缓         |
| 生产就绪  | 高         | 中       | 低      | 高          | 高          |
| 模型无关  | 是         | 是       | 是      | 否(仅OpenAI) | 否(仅Claude) |
| 状态持久化 | 内置        | 需自行     | 有限     | 有限         | 有限         |
| 可观测性  | LangSmith | 有限      | 有限     | 内置         | 有限         |
| 适合场景  | 复杂生产      | 学术研究    | 快速原型   | OpenAI全栈   | Claude全栈   |


---

## 五、新兴互操作标准

### 5.1 MCP（Model Context Protocol）— Anthropic

- 标准化 Agent 如何访问**工具和数据源**
- 已成为行业事实标准
- 定义工具的 Schema、调用协议、安全模型

### 5.2 A2A（Agent-to-Agent Protocol）— Google

- 标准化 Agent 之间如何**相互通信**
- 基于 HTTP + JSON-RPC 2.0 + SSE
- Agent Card 机制实现能力发现
- 支持同步/流式/异步三种交互模式
- 标准任务状态机：submitted → working → input-required → completed/failed/cancelled
- 2026.3 达到 v1.0.0，23K+ GitHub Stars
- 50+ 技术合作伙伴（Salesforce、SAP、ServiceNow 等）

### 5.3 MCP + A2A 互补关系

- **MCP**：Agent ↔ Tool/Data（纵向集成）
- **A2A**：Agent ↔ Agent（横向协作）
- 两者共同构成完整的 AI Agent 互操作性栈

---

## 六、金融投研领域的多Agent实践

### 6.1 TradingAgents 框架（学术研究）

模仿真实交易公司的组织结构：

- **专业化角色**：基本面分析师、情绪分析师、技术分析师
- **对抗机制**：Bull Researcher vs Bear Researcher 辩论
- **风险管理**：独立的风险监控团队
- **综合决策**：Trader Agent 基于辩论结果 + 历史数据决策
- **结果**：8 个月回测 13.43% 收益 vs S&P 500 的 10.08%

### 6.2 多Agent投研系统的关键洞察

1. **对抗验证**是最有价值的多Agent模式 — 强制搜索反面证据，克服确认偏差
2. **结构化通信**（JSON Blackboard）确保可审计性
3. **可配置协作结构**可动态适应不同分析场景
4. 多Agent 改善了决策的**可解释性** — 每个 Agent 的推理过程透明可追溯
5. 关键指标改善：累计收益率、Sharpe Ratio、最大回撤

### 6.3 对 Invest_Agent 的启示

- Bull vs Bear 对抗验证模式高度匹配需求 P2（去偏差设计）和 P7（深度优于广度）
- 角色分工模式匹配投研 vs 组合管理的自然边界
- 结构化通信匹配 P8（可追溯性）
- 但 MVP 阶段不必上多Agent，单Agent + 多Tool 足以验证核心流程

---

## 七、结论与建议

### 7.1 对 Invest_Agent 的推荐策略

**渐进式架构**：单Agent 起步 → 按需演进多Agent

1. **MVP 阶段**：单 Agent + 多 Tool，自研或使用 LangGraph
2. **增长阶段**：当工具数 > 15 或上下文压力大时，引入 Supervisor + Worker
3. **成熟阶段**：Bull/Bear 对抗验证 + 事件驱动警报

### 7.2 框架推荐

- **MVP**：自研 + LLM API 直调（最简、最可控）
- **Phase 2+**：LangGraph（最成熟、状态持久化内置）
- **不推荐**：CrewAI（生产不够稳健）、AutoGen（过于学术化）

### 7.3 关键设计原则

- Tool 接口从一开始就标准化（为未来拆分到独立 Agent 预留）
- LLM 抽象层必须模型无关
- 记忆系统作为共享基础设施，不绑定特定 Agent
- 确定性计算永远用代码，不要让 LLM 算数

---

## 参考资料

1. Anthropic - Building Effective Agents (2024.12)
2. [2602.03128] Understanding Multi-Agent LLM Frameworks: A Unified Benchmark (MAFBench)
3. [2601.03328] LLM-Enabled Multi-Agent Systems: Empirical Evaluation
4. TradingAgents: Multi-Agents LLM Financial Trading Framework (arXiv:2412.20138)
5. Google A2A Protocol v1.0.0 (2026.03)
6. LangGraph Documentation - Multi-Agent Patterns (2026)
7. OpenAI Agents SDK Documentation (2026)
8. Anthropic Claude Agent SDK (2026)
9. Single vs Multi-Agent Architecture: Production Decision Framework (Towards AI, 2026.03)
10. Multi-Agent Orchestration Patterns (Zylos Research, 2026.01)