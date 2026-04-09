# Invest-Kernel

面向长期价值投资者的 AI 投研与组合管理助手。

> 不是交易系统，不是量化策略引擎，不是 K 线技术分析工具。
> 是一个能记住你、理解你、帮你深度思考的投资伙伴。

## 愿景

投资中最大的敌人不是市场，而是认知偏差。Invest-Kernel 的目标是构建一个 **有记忆、会反思、能去偏差** 的 AI 投研系统：

- **Thesis-Driven**：每笔持仓必须有明确投资论点，没有 thesis 的仓位不应存在
- **深度优于广度**：宁可深入研究 10 家公司，也不浅层覆盖 100 家
- **去偏差设计**：成本价不进入决策 prompt、主动搜索反面证据、Invalidation Conditions 为首要约束
- **渐进式成长**：系统随使用不断积累记忆、学习风格、沉淀经验，而非每次从零开始

终极目标是可产品化的多用户平台，MVP 阶段先服务单用户，架构上为扩展留好空间。

## 架构

### Agent-First

Agent 是系统的唯一入口。用户通过自然语言交互，Agent 自主决定调用哪些 Tool、按什么顺序执行。同一套内核可接入 Web UI、飞书 Bot、CLI 等任意前端。

```
Frontend Layer          CLI / Web UI / Lark Bot / ...
        │
        ▼
Agent Core Layer        Conversation → Planning → Execution → Reflection
        │
        ├── LLM Router          任务路由到最优模型
        ├── Memory System       四层记忆架构
        └── Tool Registry       标准化工具注册
        │
Tool Layer              Portfolio / Research / Market Data / Rules Engine
        │
Infrastructure          SQLite → PostgreSQL / LLM Providers / Data Sources
```

### 核心设计原则

| 原则 | 含义 |
|------|------|
| 确定性与 LLM 分离 | 所有数值计算由代码完成，LLM 只负责分析和建议 |
| 单一真源 (SSOT) | 每类数据只有一个权威来源，派生视图从真源生成 |
| 可插拔 | 数据源、LLM、前端均可替换，投资规则和分析框架可配置 |
| Agent 自主 + 人类监督 | Agent 自主规划和执行，但买卖操作必须人类确认 |
| 可追溯 | 所有决策、分析、建议均有归档和审计轨迹 |

### 四层记忆系统

借鉴 Hermes Agent 的冻结快照架构，结合 Claude prompt caching 优化：

```
┌─ 冻结层 (Frozen)      ~1,300 tokens，会话启动注入，保护前缀缓存
│   用户画像 + Agent 状态（核心教训、关注焦点）
│
├─ 索引层 (Index)        ~800 tokens，动态生成的压缩摘要
│   持仓概况 / Thesis 状态 / 最近决策 / 规则摘要
│
├─ 检索层 (Retrieval)    按需加载，混合检索（向量 + 全文 + 结构化过滤）
│   记忆搜索 / 完整 Thesis / 决策历史 / 对话搜索
│
└─ 归档层 (Archive)      冷存储，仅通过检索层间接访问
    Thesis 历史版本 / 全部决策记录 / 日检报告 / 对话历史
```

记忆具有领域特化的衰减策略——投资原则永不遗忘，临时查询 7 天后淡化。

### 渐进式演进

MVP 采用单 Agent + 多 Tool 架构，当工具膨胀或需要并行分析时，平滑拆分为 Supervisor + 专业 Worker Agent（Research / Portfolio / Daily Check），最终引入对抗验证（Bull/Bear 辩论）。

## 技术栈

| 层面 | 选型 |
|------|------|
| 语言 | Python 3.12+ |
| 包管理 | uv |
| 类型系统 | Pydantic v2 |
| 异步 | asyncio + httpx |
| 数据库 | SQLite (MVP) → PostgreSQL |
| 向量存储 | sqlite-vec (MVP) → pgvector |
| API | FastAPI |
| 测试 | pytest + pytest-asyncio |
| Lint | ruff |

## 项目状态

当前处于架构设计阶段，核心文档：

```
docs/
├── architecture/
│   └── ARCHITECTURE.md          # 系统架构设计
└── research/
    ├── multi-agent-architecture-research.md
    └── agent-memory-system-research.md
```

## License

Private — All rights reserved.
