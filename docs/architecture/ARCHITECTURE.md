# Invest_Agent v2 — 系统架构设计

> 版本：0.2-draft
> 日期：2026-04-09
> 状态：待审阅
> 前置文档：[REQUIREMENTS.md](../../REQUIREMENTS.md)、[多Agent架构研究](../research/multi-agent-architecture-research.md)、[记忆系统研究](../research/agent-memory-system-research.md)

---

## 一、架构总览

### 1.1 架构定位

基于调研结论，采用 **渐进式架构** 策略：

- **MVP**：单 Agent + 多 Tool，自研内核
- **Phase 2**：按需拆分为 Supervisor + 专业 Worker Agent
- **Phase 3**：对抗验证 + 事件驱动

本文档以 MVP 架构为主体，同时在接口设计上为后续演进预留空间。

### 1.2 系统全景

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend Layer                         │
│  ┌─────────┐  ┌──────────────┐  ┌─────────┐  ┌──────────┐  │
│  │   CLI   │  │   Web UI     │  │ Lark Bot│  │ Future.. │  │
│  └────┬────┘  └──────┬───────┘  └────┬────┘  └─────┬────┘  │
│       └──────────────┼───────────────┼──────────────┘       │
│                      ▼               ▼                      │
│              ┌───────────────────────────┐                  │
│              │    Frontend Gateway API   │                  │
│              └───────────┬───────────────┘                  │
└──────────────────────────┼──────────────────────────────────┘
                           │ (统一消息协议)
┌──────────────────────────┼──────────────────────────────────┐
│                    Agent Core Layer                          │
│              ┌───────────▼───────────────┐                  │
│              │     Conversation Manager  │                  │
│              │   (会话管理 / 上下文组装)  │                  │
│              └───────────┬───────────────┘                  │
│                          ▼                                  │
│              ┌───────────────────────────┐                  │
│              │       Core Agent          │                  │
│              │  ┌─────────────────────┐  │                  │
│              │  │   Planning Module   │  │                  │
│              │  │  (任务分解/排序)    │  │                  │
│              │  ├─────────────────────┤  │                  │
│              │  │  Execution Engine   │  │                  │
│              │  │  (Tool 调度/执行)   │  │                  │
│              │  ├─────────────────────┤  │                  │
│              │  │  Reflection Module  │  │                  │
│              │  │  (结果验证/自纠)    │  │                  │
│              │  └─────────────────────┘  │                  │
│              └───────────┬───────────────┘                  │
│                          │                                  │
│     ┌────────────────────┼────────────────────┐             │
│     ▼                    ▼                    ▼             │
│ ┌────────┐        ┌───────────┐        ┌───────────┐       │
│ │  LLM   │        │  Memory   │        │   Tool    │       │
│ │ Router │        │  System   │        │ Registry  │       │
│ └────────┘        └───────────┘        └───────────┘       │
└─────────────────────────────────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────┐
│                    Tool Layer                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │Portfolio │ │ Research │ │  Market  │ │  Rules   │       │
│  │  Tools   │ │  Tools   │ │Data Tools│ │  Engine  │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────┐
│                 Infrastructure Layer                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Database │ │LLM Provid│ │Data Sourc│ │  Config  │       │
│  │          │ │   ers    │ │   es     │ │  Store   │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 技术栈选型

| 层面 | 选型 | 理由 |
|------|------|------|
| 语言 | **Python 3.12+** | 生态最丰富（LLM SDK、yfinance、JQData）、开发效率高 |
| 包管理 | **uv** | 快速、现代、替代 pip+venv |
| 类型系统 | **Pydantic v2** | 数据验证、序列化、Schema 生成 |
| 异步 | **asyncio + httpx** | Agent 和 Tool 天然异步 |
| 数据库（MVP） | **SQLite + aiosqlite** | 嵌入式零运维；WAL 模式支持基本并发；Hermes 已验证可行 |
| 数据库（Phase 2+） | **PostgreSQL + SQLAlchemy 2.0** | 多用户并发；ACID 事务；JSON 列支持半结构化 |
| 全文搜索 | **SQLite FTS5** | 会话历史检索；嵌入式；Phase 2+ 迁移 PostgreSQL full-text |
| 向量存储（MVP） | **sqlite-vec** | 轻量嵌入式向量检索；无需外部服务 |
| 向量存储（Phase 2+） | **pgvector** | 高性能混合查询；与 PG 共用减少运维 |
| API 框架 | **FastAPI** | 前端 Gateway；自动 OpenAPI 文档 |
| 配置 | **YAML + Pydantic Settings** | 可读性好、类型安全 |
| 测试 | **pytest + pytest-asyncio** | 标准选择 |
| Lint/Format | **ruff** | 极快的 Python linter/formatter |

---

## 二、项目目录结构

```
invest_agent/
├── pyproject.toml                 # 项目元数据与依赖
├── README.md
├── docs/
│   ├── architecture/              # 架构文档
│   └── research/                  # 调研文档
│
├── config/                        # 配置文件
│   ├── default.yaml               # 默认配置
│   ├── llm_models.yaml            # LLM 模型配置
│   └── rules/                     # 投资规则配置
│       └── default_rules.yaml
│
├── src/
│   └── invest_agent/
│       ├── __init__.py
│       │
│       ├── agent/                 # Agent 核心
│       │   ├── __init__.py
│       │   ├── core.py            # Core Agent 主循环
│       │   ├── planner.py         # 任务规划模块
│       │   ├── executor.py        # Tool 执行引擎
│       │   ├── reflector.py       # 反思与验证
│       │   └── conversation.py    # 会话管理
│       │
│       ├── llm/                   # LLM 抽象层
│       │   ├── __init__.py
│       │   ├── base.py            # LLM 基类接口
│       │   ├── router.py          # 模型路由
│       │   ├── providers/
│       │   │   ├── __init__.py
│       │   │   ├── anthropic.py   # Claude 适配器
│       │   │   ├── openai.py      # GPT 适配器
│       │   │   └── gemini.py      # Gemini 适配器
│       │   └── prompts/           # Prompt 模板
│       │       ├── system.py
│       │       └── templates/
│       │
│       ├── tools/                 # Tool 注册与实现
│       │   ├── __init__.py
│       │   ├── registry.py        # Tool 注册表
│       │   ├── base.py            # Tool 基类
│       │   ├── portfolio/         # 组合管理 Tools
│       │   │   ├── __init__.py
│       │   │   ├── query.py       # 持仓查询
│       │   │   ├── mutate.py      # 持仓变更
│       │   │   └── health.py      # 组合健康检查
│       │   ├── market/            # 市场数据 Tools
│       │   │   ├── __init__.py
│       │   │   ├── price.py       # 价格刷新
│       │   │   └── fundamentals.py
│       │   ├── research/          # 投研 Tools
│       │   │   ├── __init__.py
│       │   │   ├── thesis.py      # Thesis 管理
│       │   │   └── analysis.py    # 分析框架
│       │   └── rules/             # 规则引擎 Tools
│       │       ├── __init__.py
│       │       └── checker.py     # 规则检查
│       │
│       ├── memory/                # 记忆系统
│       │   ├── __init__.py
│       │   ├── manager.py         # 记忆管理器（生命周期编排）
│       │   ├── frozen.py          # 冻结快照（USER_PROFILE / AGENT_STATE）
│       │   ├── index_builder.py   # 索引层动态生成（持仓/Thesis/决策摘要）
│       │   ├── store.py           # 持久化存储（SQLite / 未来 PostgreSQL）
│       │   ├── retriever.py       # 混合检索（向量+FTS5+结构化过滤）
│       │   ├── decay.py           # 衰减引擎（领域特化衰减策略）
│       │   ├── compressor.py      # 记忆压缩（同类合并/摘要）
│       │   └── types.py           # 记忆类型与衰减策略定义
│       │
│       ├── data/                  # 数据源抽象
│       │   ├── __init__.py
│       │   ├── base.py            # 数据源基类
│       │   ├── adapters/
│       │   │   ├── __init__.py
│       │   │   ├── yfinance.py
│       │   │   └── jqdata.py
│       │   └── cache.py           # 数据缓存层
│       │
│       ├── models/                # 数据模型 (Pydantic)
│       │   ├── __init__.py
│       │   ├── portfolio.py       # 持仓/组合模型
│       │   ├── thesis.py          # Thesis 模型
│       │   ├── decision.py        # 决策记录模型
│       │   ├── market.py          # 市场数据模型
│       │   └── config.py          # 配置模型
│       │
│       ├── storage/               # 存储层
│       │   ├── __init__.py
│       │   ├── database.py        # DB 连接与迁移
│       │   ├── repositories/      # Repository Pattern
│       │   │   ├── __init__.py
│       │   │   ├── portfolio.py
│       │   │   ├── thesis.py
│       │   │   └── decision.py
│       │   └── migrations/        # Alembic 迁移
│       │
│       ├── gateway/               # 前端 Gateway
│       │   ├── __init__.py
│       │   ├── app.py             # FastAPI 应用
│       │   ├── routes/
│       │   │   ├── __init__.py
│       │   │   ├── chat.py        # 对话接口
│       │   │   └── webhook.py     # 飞书等 Webhook
│       │   └── protocol.py        # 统一消息协议
│       │
│       ├── archive/               # 归档系统
│       │   ├── __init__.py
│       │   ├── archiver.py        # 自动归档
│       │   └── query.py           # 归档查询
│       │
│       └── utils/                 # 公共工具
│           ├── __init__.py
│           ├── calc.py            # 确定性计算（PnL/CAGR/权重等）
│           ├── formatting.py      # 输出格式化
│           └── logging.py         # 日志配置
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
│
└── scripts/
    ├── setup_db.py                # 数据库初始化
    └── daily_check.py             # 日检定时脚本
```

---

## 三、核心模块详细设计

### 3.1 Agent Core — 主循环

Agent 的核心是一个 ReAct 循环（Reasoning + Acting），包含规划、执行、反思三个阶段：

```
用户输入
  │
  ▼
┌──────────────────────────────┐
│  Conversation Manager        │
│  - 加载会话上下文             │
│  - 检索相关记忆               │
│  - 组装 System Prompt         │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  Planning Module              │◄──────────┐
│  - 理解用户意图               │           │
│  - 分解为子任务               │           │
│  - 选择需要的 Tools           │           │
└──────────┬───────────────────┘           │
           │                               │
           ▼                               │
┌──────────────────────────────┐           │
│  Execution Engine             │           │
│  - 按序/并行调用 Tools        │           │ (需要重新规划)
│  - 收集 Tool 结果             │           │
│  - 关键操作请求人类确认       │           │
└──────────┬───────────────────┘           │
           │                               │
           ▼                               │
┌──────────────────────────────┐           │
│  Reflection Module            │───────────┘
│  - 验证结果完整性             │
│  - 检查是否回答了用户问题     │
│  - 决定是否需要补充步骤       │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  Response Generation          │
│  - 生成用户可读的回复         │
│  - 持久化记忆                 │
│  - 归档决策/分析              │
└──────────────────────────────┘
```

**核心接口设计**：

```python
class CoreAgent:
    """Agent 主循环，协调 Planning/Execution/Reflection"""

    async def run(self, user_message: str, session_id: str) -> AgentResponse:
        """处理一次用户消息的完整流程"""
        ...

    async def _plan(self, context: ConversationContext) -> ExecutionPlan:
        """LLM 规划：分析意图、分解任务、选择 Tools"""
        ...

    async def _execute(self, plan: ExecutionPlan) -> list[ToolResult]:
        """执行规划：调用 Tools、收集结果"""
        ...

    async def _reflect(
        self, context: ConversationContext, results: list[ToolResult]
    ) -> ReflectionResult:
        """反思：验证结果、决定是否需要补充"""
        ...
```

### 3.2 LLM 抽象层

#### 3.2.1 设计目标

- 业务代码不直接调用任何特定 LLM SDK
- 支持任务级别的模型路由
- 统一的流式/非流式接口
- 结构化输出（JSON mode / Tool calling）

#### 3.2.2 接口设计

```python
class LLMProvider(ABC):
    """LLM 提供商基类"""

    @abstractmethod
    async def complete(self, request: CompletionRequest) -> CompletionResponse:
        """非流式补全"""
        ...

    @abstractmethod
    async def stream(self, request: CompletionRequest) -> AsyncIterator[StreamChunk]:
        """流式补全"""
        ...

    @abstractmethod
    async def complete_with_tools(
        self, request: CompletionRequest, tools: list[ToolSchema]
    ) -> ToolCallingResponse:
        """带 Tool Calling 的补全"""
        ...


class LLMRouter:
    """模型路由器：根据任务类型选择最优模型"""

    def route(self, task: TaskType, context: RoutingContext) -> LLMProvider:
        """
        路由策略示例：
        - DEEP_ANALYSIS → Claude Opus / GPT-4o
        - SUMMARIZATION → Claude Sonnet / GPT-4o-mini
        - SIMPLE_QUERY  → Claude Haiku / GPT-4o-mini
        - DATA_EXTRACT  → Gemini Flash
        """
        ...
```

#### 3.2.3 模型配置示例（`config/llm_models.yaml`）

```yaml
models:
  deep_analysis:
    provider: anthropic
    model: claude-sonnet-4-20250514
    max_tokens: 8192
    temperature: 0.3
    fallback: openai/gpt-4o

  summarization:
    provider: anthropic
    model: claude-haiku-4-20250414
    max_tokens: 2048
    temperature: 0.2

  simple_query:
    provider: openai
    model: gpt-4o-mini
    max_tokens: 1024
    temperature: 0.1

routing_rules:
  - task_pattern: "thesis_*"
    model: deep_analysis
  - task_pattern: "daily_check_*"
    model: summarization
  - task_pattern: "price_query"
    model: simple_query
```

### 3.3 Tool 系统

#### 3.3.1 设计原则

- **标准化接口**：每个 Tool 自描述（名称、描述、参数 Schema、返回 Schema）
- **自动注册**：通过装饰器或类继承自动注册到 Registry
- **确定性优先**：数值计算在 Tool 内部用代码完成，不交给 LLM
- **可组合**：复杂操作由 Agent 编排多个 Tool 完成，而非在 Tool 内硬编码流程
- **为拆分预留**：Tool 接口标准化后，未来可平滑迁移到独立 Agent

#### 3.3.2 Tool 基类与注册

```python
class ToolSchema(BaseModel):
    """Tool 的自描述 Schema，用于 LLM 的 Tool Calling"""
    name: str
    description: str
    parameters: dict  # JSON Schema
    returns: dict     # JSON Schema
    requires_confirmation: bool = False  # 是否需要人类确认

class BaseTool(ABC):
    """所有 Tool 的基类"""

    @abstractmethod
    def schema(self) -> ToolSchema:
        """返回 Tool 的 Schema 描述"""
        ...

    @abstractmethod
    async def execute(self, params: dict) -> ToolResult:
        """执行 Tool，返回结构化结果"""
        ...


class ToolRegistry:
    """Tool 注册表：管理所有可用 Tools"""

    def register(self, tool: BaseTool) -> None: ...
    def get(self, name: str) -> BaseTool: ...
    def list_schemas(self) -> list[ToolSchema]: ...
    def list_schemas_for_context(self, context: str) -> list[ToolSchema]:
        """根据上下文返回相关 Tools 的子集，避免 Tool 过多导致注意力稀释"""
        ...
```

#### 3.3.3 MVP Tool 清单

| Tool | 功能 | 计算类型 | 确认 |
|------|------|----------|------|
| `portfolio_query` | 查询持仓/组合概况 | 确定性 | 否 |
| `portfolio_add_position` | 添加持仓 | 确定性 | **是** |
| `portfolio_update_position` | 更新持仓 | 确定性 | **是** |
| `portfolio_remove_position` | 删除持仓 | 确定性 | **是** |
| `price_refresh` | 刷新市场价格 | 确定性 | 否 |
| `price_history` | 获取历史价格 | 确定性 | 否 |
| `rules_check` | 运行规则检查 | 确定性 | 否 |
| `thesis_get` | 获取 Thesis | 读取 | 否 |
| `thesis_create` | 创建 Thesis | LLM辅助 | **是** |
| `thesis_update` | 更新 Thesis | LLM辅助 | **是** |
| `daily_check` | 执行日检流程 | 混合 | 否 |
| `calc_portfolio_metrics` | 计算组合指标 | 确定性 | 否 |
| `memory_search` | 搜索相关记忆 | 检索 | 否 |
| `memory_store` | 存储新记忆 | 写入 | 否 |

### 3.4 记忆系统

> 设计参考：[记忆系统研究](../research/agent-memory-system-research.md)，重点借鉴 Hermes Agent 的三层冻结快照架构。

#### 3.4.1 四层记忆架构

融合 Hermes 的冻结快照 + Claude Code 的轻量检索 + SimpleMem 的语义压缩：

```
┌────────────────────────────────────────────────────────────────────┐
│  Layer 1: 冻结层（Frozen Layer）                                    │
│  ~1,300 tokens，会话启动一次性注入 system prompt                     │
│  会话期间不可变 → 保护 LLM 前缀缓存（节省 ~75% 输入 token 成本）      │
│                                                                    │
│  ┌─────────────────────────────┐ ┌──────────────────────────────┐  │
│  │ USER_PROFILE (~500 tokens)  │ │ AGENT_STATE (~800 tokens)    │  │
│  │ 投资偏好 / 风险容忍度        │ │ 核心教训 Top 5               │  │
│  │ 沟通风格 / 关键约束          │ │ 当前关注焦点                 │  │
│  │ 容量限制明确告知 Agent       │ │ 上次会话概要                 │  │
│  └─────────────────────────────┘ └──────────────────────────────┘  │
├────────────────────────────────────────────────────────────────────┤
│  Layer 2: 索引层（Index Layer）                                     │
│  ~500-1,000 tokens，会话开始时从 DB 动态生成                         │
│  压缩摘要视图，让 Agent 有全局视野无需调用 Tool                       │
│                                                                    │
│  "持仓: AAPL(22%) MSFT(18%) GOOGL(15%) ... 共 10 只, 现金 8%"     │
│  "Thesis: AAPL[有效] MSFT[有效] BABA[观察] ... 共 10 条"           │
│  "最近决策: 2026-04-05 加仓AAPL / 2026-03-28 减仓BABA"            │
│  "规则: 4 条启用, 0 违规"                                           │
├────────────────────────────────────────────────────────────────────┤
│  Layer 3: 检索层（Retrieval Layer）                                 │
│  按需加载，通过 Tool 调用获取                                        │
│                                                                    │
│  memory_search(query, type, time_range) → 混合检索                 │
│  thesis_get(symbol) → 完整 Thesis 文档                              │
│  decision_history(symbol, limit) → 决策记录                         │
│  session_search(query) → FTS5 全文搜索历史对话                      │
├────────────────────────────────────────────────────────────────────┤
│  Layer 4: 归档层（Archive Layer）                                   │
│  冷存储，仅通过检索层间接访问                                        │
│                                                                    │
│  完整 Thesis 历史版本 / 所有决策记录 / 日检报告 / 对话历史           │
└────────────────────────────────────────────────────────────────────┘
```

#### 3.4.2 冻结快照机制（参考 Hermes）

冻结快照是整个记忆系统**最关键的设计决策**：

1. **写入时机**：会话中通过 memory Tool 修改的内容**立即写盘**，但**下次会话才注入 system prompt**
2. **缓存友好**：system prompt 前缀在整个会话期间不变，Anthropic prompt caching 命中率最高
3. **容量可见**：Agent 能看到剩余容量和百分比，自主管理空间（添加/替换/删除条目）
4. **安全扫描**：所有记忆写入前检查 prompt 注入、凭据泄露等恶意模式

```python
class FrozenMemory:
    """冻结快照记忆 — 参考 Hermes MEMORY.md/USER.md"""

    USER_PROFILE_LIMIT = 1375     # 字符 (~500 tokens)
    AGENT_STATE_LIMIT = 2200      # 字符 (~800 tokens)
    SEPARATOR = "§"               # 条目分隔符

    def snapshot(self) -> str:
        """生成冻结快照，注入 system prompt（仅会话启动时调用一次）"""
        ...

    def update(self, action: str, target: str, content: str) -> str:
        """修改记忆：add/replace/remove（立即写盘，下次会话生效）"""
        ...

    def capacity_info(self) -> dict:
        """返回各区域的已用/剩余容量，供 Agent 参考"""
        ...
```

#### 3.4.3 渐进披露策略

核心原则：**不预加载全部信息，按需按层加载**。效率对比：

| 模式 | 100 条记忆的 token 消耗 | 节省率 |
|------|------------------------|--------|
| 急切加载 | 100 × 1,000 = 100,000 | — |
| 渐进披露 | 100 × 50（元数据）+ 1 × 1,000（激活） = 6,000 | **94%** |

具体到各数据域：

| 数据域 | 元数据层（常驻） | 激活层（按需） | 按需层（Tool） |
|--------|-----------------|---------------|---------------|
| 持仓 | "AAPL 22% / MSFT 18% ..." | 单个持仓完整详情 | 历史交易记录 |
| Thesis | "AAPL[有效] 一句话摘要" | 完整 Thesis 文档 | Thesis 所有历史版本 |
| 规则 | "4 条启用, 0 违规" | 具体规则定义 | 历史违规记录 |
| 决策 | "最近 5 条摘要" | 单条完整决策记录 | 关联的分析过程 |
| 日检 | "最近: 2026-04-09, 2 警告" | 完整日检报告 | 历史日检趋势 |
| 教训 | "核心: 追涨杀跌 / 集中度" | 完整复盘记录 | 当时市场环境 |

#### 3.4.4 记忆类型与数据模型

```python
class MemoryType(str, Enum):
    USER_PROFILE = "user_profile"         # 用户风格/偏好（冻结层）
    AGENT_STATE = "agent_state"           # Agent 状态/教训（冻结层）
    DECISION = "decision"                 # 决策记录
    CONVERSATION = "conversation"         # 对话上下文
    MARKET_INSIGHT = "market_insight"     # 市场洞察
    MISTAKE = "mistake"                   # 错误教训
    THESIS_EVOLUTION = "thesis_evolution" # Thesis 演变

class DecayPolicy(str, Enum):
    NEVER = "never"           # 永不衰减：投资原则、核心教训、IC 条件
    SLOW = "slow"             # 半衰期 180 天：Thesis 核心、重大决策
    NORMAL = "normal"         # 半衰期 30 天：市场观察、日检洞察
    FAST = "fast"             # 半衰期 7 天：临时查询、格式请求

class Memory(BaseModel):
    id: str
    type: MemoryType
    content: str
    metadata: dict                    # 结构化元数据（标的、日期、标签等）
    embedding: list[float] | None     # 向量嵌入（检索层记忆）
    importance: float                 # 重要性评分 (0.0 - 1.0)
    decay_policy: DecayPolicy         # 衰减策略
    created_at: datetime
    last_accessed: datetime
    access_count: int = 0
    decayed_importance: float | None  # 衰减后的有效重要性（计算字段）
```

#### 3.4.5 检索策略 — 混合检索 + 深度分级

```
用户查询 / Agent 内部检索
  │
  ▼
┌──────────────────────────────────────┐
│ 1. 意图分析                           │
│    判断检索需求：                      │
│    - 需要哪些记忆类型？               │
│    - 需要多大的时间范围？              │
│    - 需要多深的检索深度？              │
└──────────┬───────────────────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌─────────┐ ┌─────────┐
│ 向量检索 │ │ FTS5   │
│ top-K   │ │ 关键词  │
└────┬────┘ └────┬────┘
     └─────┬─────┘
           ▼
┌──────────────────────────────────────┐
│ 2. 融合排序                           │
│    - 语义相似度 × 权重                │
│    - 时效性加权（近期优先）            │
│    - 有效重要性（衰减后）             │
│    - 结构化过滤（类型、标的）          │
│    - 去重                             │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│ 3. 深度分级返回                       │
│    L1: 摘要片段 + 相关性评分（默认）   │
│    L2: 完整段落（Agent 请求展开时）    │
│    L3: 原始记录（需要精确细节时）      │
└──────────────────────────────────────┘
```

#### 3.4.6 记忆生命周期与衰减

```
创建 ──→ 活跃 ──→ 衰减 ──→ 压缩 ──→ 归档
  │        │        │        │        │
  │   被引用时回升   │   同类合并    移至冷存储
  │                  │   LLM 摘要
  │            重要性低于阈值
  │
Agent 对话/决策/分析后自动提取
```

**领域特化衰减规则**：

```yaml
decay_policies:
  never:
    types: [investment_principles, critical_mistakes, thesis_invalidation_conditions]

  slow:
    half_life_days: 180
    types: [thesis_core_logic, major_decisions, market_structure_insights]

  normal:
    half_life_days: 30
    types: [market_observations, daily_check_insights, conversation_context]

  fast:
    half_life_days: 7
    types: [price_queries, formatting_requests, temporary_calculations]
```

**衰减公式**：

```python
def effective_importance(memory: Memory, now: datetime) -> float:
    if memory.decay_policy == DecayPolicy.NEVER:
        return memory.importance

    half_life = HALF_LIFE_DAYS[memory.decay_policy]
    days_since_access = (now - memory.last_accessed).days
    decay_factor = 0.5 ** (days_since_access / half_life)

    # 访问频率加成：高频访问的记忆衰减更慢
    frequency_bonus = min(memory.access_count / 10, 0.3)

    return memory.importance * decay_factor + frequency_bonus
```

#### 3.4.7 记忆 Tool 接口

```python
# 暴露给 Agent 的记忆操作 Tools
memory_frozen_update   # 修改冻结层（add/replace/remove，下次会话生效）
memory_search          # 混合检索（向量+FTS5+结构化过滤）
memory_store           # 存储新记忆到检索层
session_search         # 全文搜索历史对话
```

### 3.5 数据源抽象层

#### 3.5.1 适配器模式

```python
class DataSourceAdapter(ABC):
    """数据源适配器基类"""

    @abstractmethod
    async def get_price(self, symbol: str) -> PriceData: ...

    @abstractmethod
    async def get_history(
        self, symbol: str, start: date, end: date
    ) -> list[PriceData]: ...

    @abstractmethod
    async def get_fundamentals(self, symbol: str) -> FundamentalsData: ...

    @abstractmethod
    def supports(self, market: Market) -> bool:
        """声明此适配器支持哪些市场"""
        ...


class DataSourceManager:
    """数据源管理器：根据市场自动选择适配器"""

    def __init__(self, adapters: list[DataSourceAdapter]):
        self._adapters = adapters

    async def get_price(self, symbol: str) -> PriceData:
        adapter = self._resolve_adapter(symbol)
        return await adapter.get_price(symbol)

    def _resolve_adapter(self, symbol: str) -> DataSourceAdapter:
        market = detect_market(symbol)
        for adapter in self._adapters:
            if adapter.supports(market):
                return adapter
        raise NoAdapterError(f"No adapter for {symbol}")
```

#### 3.5.2 数据缓存

```
请求 → 检查缓存（TTL 策略）
           │
     ┌─────┼─────┐
     ▼           ▼
  命中          未命中 → 调用适配器 → 写入缓存 → 返回
     │
     ▼
   返回
```

- 价格数据：TTL 5 分钟（交易时间）/ 24 小时（非交易时间）
- 基本面数据：TTL 24 小时
- 历史数据：永久缓存（不可变数据）

### 3.6 规则引擎

#### 3.6.1 规则定义（`config/rules/default_rules.yaml`）

```yaml
rules:
  - name: max_position_concentration
    description: 单个标的不超过总组合的 25%
    type: threshold
    metric: position_weight
    operator: lte
    value: 0.25
    severity: critical
    action: block  # block | warn | info

  - name: min_cash_ratio
    description: 现金比例不低于 5%
    type: threshold
    metric: cash_ratio
    operator: gte
    value: 0.05
    severity: warning
    action: warn

  - name: max_sector_concentration
    description: 单个行业不超过 40%
    type: threshold
    metric: sector_weight
    operator: lte
    value: 0.40
    severity: warning
    action: warn

  - name: thesis_required
    description: 每个持仓必须有关联的 Thesis
    type: existence
    check: thesis_exists
    severity: critical
    action: block
```

#### 3.6.2 规则检查流程

```python
class RuleEngine:
    """确定性规则引擎 — 纯代码计算，不涉及 LLM"""

    def check_all(self, portfolio: Portfolio) -> list[RuleResult]:
        """运行所有规则，返回结果列表"""
        ...

    def check_action(
        self, action: ProposedAction, portfolio: Portfolio
    ) -> ActionDecision:
        """检查某个拟议操作是否违反规则"""
        ...
```

### 3.7 前端 Gateway

#### 3.7.1 统一消息协议

所有前端通过统一协议与 Agent 通信，实现前端无关性：

```python
class UserMessage(BaseModel):
    session_id: str
    content: str
    attachments: list[Attachment] = []
    metadata: dict = {}  # 前端特定元数据

class AgentResponse(BaseModel):
    session_id: str
    content: str                        # 主要回复文本
    tool_calls: list[ToolCallRecord]    # 使用了哪些 Tool
    confirmations: list[Confirmation]   # 需要用户确认的操作
    data: dict | None = None            # 结构化数据（图表、表格等）
    memories_stored: int = 0            # 本次存储的记忆数
```

#### 3.7.2 Gateway 路由

```python
# FastAPI Gateway
@router.post("/chat")
async def chat(message: UserMessage) -> AgentResponse:
    """通用对话接口 — Web UI / CLI 使用"""
    ...

@router.post("/webhook/lark")
async def lark_webhook(event: LarkEvent) -> LarkResponse:
    """飞书 Bot Webhook — 转换为统一消息格式后调用 Agent"""
    ...

@router.post("/chat/{session_id}/confirm")
async def confirm(session_id: str, confirmation: ConfirmationResult):
    """用户确认操作"""
    ...

@router.get("/chat/{session_id}/stream")
async def stream(session_id: str):
    """SSE 流式响应"""
    ...
```

### 3.8 归档系统

#### 3.8.1 归档内容

| 归档类型 | 触发时机 | 内容 |
|----------|----------|------|
| 决策记录 | 买/卖/加仓/减仓操作 | 操作详情、理由、当时的Thesis、规则检查结果 |
| 分析报告 | 完成投研分析 | Thesis内容、数据快照、关键发现 |
| 日检报告 | 每次日检 | 组合状态、规则结果、异常提醒、建议 |
| Thesis变更 | Thesis更新 | 变更前后对比、变更原因、触发事件 |

#### 3.8.2 归档存储

- 结构化数据 → 数据库（可查询、可聚合）
- 原始文本/报告 → JSON 列或文件系统
- 所有归档带时间戳、关联实体ID、操作者标识

### 3.9 上下文管理与压缩

> 参考 Hermes Agent 的双层压缩系统和 Anthropic prompt caching 策略。

#### 3.9.1 上下文预算分配（200K 窗口示例）

```
总预算: 200,000 tokens

├─ System Prompt (冻结, 可缓存)
│   ├─ Agent 身份 + 行为指导:        ~2,000 tokens
│   ├─ Tool Schema (14 Tools):       ~3,000 tokens
│   ├─ USER_PROFILE (冻结):           ~500 tokens
│   ├─ AGENT_STATE (冻结):            ~800 tokens
│   └─ 小计:                         ~6,300 tokens (3.2%)
│
├─ 索引层 (动态, 会话开始):           ~800 tokens (0.4%)
│
├─ 对话历史 (压缩后):               ~40,000 tokens (20%)
│
├─ Tool 调用结果 (按需):         ~10,000-30,000 tokens (5-15%)
│
├─ LLM 推理 + 输出:                ~10,000 tokens (5%)
│
└─ 安全余量:                       ~110,000+ tokens (55%+)
```

#### 3.9.2 Prompt Caching 策略

system prompt 的冻结设计天然兼容 Anthropic prompt caching：

- **断点 1**：System prompt（所有轮次稳定，~6,300 tokens）
- **断点 2-4**：最后 3 条消息的滚动窗口

关键约束：
- system prompt 在会话期间**不可变**（记忆修改下次会话生效）
- 压缩时仅在首次压缩追加一行注释，不重写 system prompt
- 避免在中间消息插入/删除，防止缓存失效

#### 3.9.3 上下文压缩算法

采用 Hermes 验证过的四阶段压缩方案：

```
触发条件: 上下文使用 >= 50% 窗口大小

Phase 1: 裁剪旧 Tool 输出（无 LLM 调用）
  - 保护尾部区域的 Tool 结果不动
  - 超过 200 字符的旧 Tool 输出替换为占位符
  - 最便宜的预处理步骤

Phase 2: 确定压缩边界
  - 保护区: 头 3 条消息 + 尾部 token 预算（或最少 20 条）
  - 中间区: 将被摘要的消息范围
  - 边界对齐: 不拆分 tool_call / tool_result 组

Phase 3: 结构化摘要（LLM 调用）
  - 摘要模板:
    ## Goal
    ## Constraints & Preferences
    ## Progress (Done / In Progress / Blocked)
    ## Key Decisions
    ## Relevant Files / Data
    ## Next Steps
    ## Critical Context
  - Token 预算: 被压缩内容 × 20%, 最少 2,000, 最多 12,000
  - 迭代式: 后续压缩是更新上一次摘要，而非重新生成

Phase 4: 组装压缩后消息列表
  - 头部 + 摘要消息 + 尾部
  - 清理孤立的 tool_call/tool_result 对

安全网: 85% 阈值时触发粗粒度安全压缩，防止 API 失败
```

#### 3.9.4 Conversation Manager — Prompt 组装

参考 Hermes 的 10 层 prompt 组装策略：

```python
class ConversationManager:
    """会话管理器 — 负责上下文组装和压缩"""

    def build_system_prompt(self, session: Session) -> str:
        """组装 system prompt（会话启动时调用一次，之后冻结）"""
        layers = [
            self._agent_identity(),           # Agent 身份与行为指导
            self._tool_schemas(),              # Tool 描述（渐进披露：仅 Schema）
            self._frozen_user_profile(),       # 冻结 USER_PROFILE
            self._frozen_agent_state(),        # 冻结 AGENT_STATE
            self._dynamic_index(),             # 索引层（持仓/Thesis/决策摘要）
            self._timestamp_and_session(),     # 时间戳和会话 ID
        ]
        return "\n\n".join(filter(None, layers))

    async def maybe_compress(
        self, messages: list[Message], token_count: int
    ) -> list[Message]:
        """当上下文压力超过阈值时触发压缩"""
        ...
```

---

## 四、数据架构

### 4.1 数据库 Schema 概览

MVP 使用 SQLite，表结构设计兼容后续迁移到 PostgreSQL。

#### 4.1.1 业务数据表

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  portfolios  │     │    positions     │     │     theses       │
│──────────────│     │──────────────────│     │──────────────────│
│ id           │     │ id               │     │ id               │
│ name         │◄────│ portfolio_id     │     │ position_id      │
│ user_id      │     │ symbol           │     │ version          │
│ cash_balance │     │ shares           │     │ status           │
│ created_at   │     │ avg_cost         │     │ content (JSON)   │
│ updated_at   │     │ current_price    │     │ invalidation_cond│
└──────────────┘     │ sector           │     │ created_at       │
                     │ market           │     │ updated_at       │
                     │ added_at         │     └──────────────────┘
                     │ updated_at       │
                     └──────────────────┘

┌──────────────────┐     ┌──────────────────┐
│   decisions      │     │   daily_checks   │
│──────────────────│     │──────────────────│
│ id               │     │ id               │
│ portfolio_id     │     │ portfolio_id     │
│ position_id      │     │ executed_at      │
│ action_type      │     │ status_snapshot  │
│ reason           │     │ rule_results     │
│ thesis_snapshot  │     │ recommendations  │
│ rule_check_result│     │ report (JSON)    │
│ executed_at      │     └──────────────────┘
│ confirmed_by     │
└──────────────────┘
```

#### 4.1.2 记忆系统表

```
┌───────────────────────────┐
│  frozen_memories           │ ← 冻结层 (参考 Hermes MEMORY.md/USER.md)
│───────────────────────────│
│ id                         │
│ category (user_profile /   │
│           agent_state)     │
│ content TEXT               │ ← 完整内容（有字符上限）
│ char_limit INTEGER         │ ← 容量上限
│ updated_at                 │
└───────────────────────────┘

┌───────────────────────────┐
│  memories                  │ ← 检索层
│───────────────────────────│
│ id                         │
│ type (MemoryType)          │
│ content TEXT               │
│ metadata (JSON)            │ ← 标的、标签、关联实体等
│ embedding BLOB             │ ← 向量嵌入 (sqlite-vec)
│ importance REAL            │ ← 原始重要性 (0-1)
│ decay_policy TEXT          │ ← never/slow/normal/fast
│ created_at REAL            │
│ last_accessed REAL         │
│ access_count INTEGER       │
└───────────────────────────┘

┌───────────────────────────┐
│  sessions                  │ ← 会话管理 (参考 Hermes)
│───────────────────────────│
│ id TEXT PRIMARY KEY        │
│ parent_session_id TEXT     │ ← 会话谱系（压缩触发分裂）
│ model TEXT                 │
│ system_prompt TEXT         │
│ started_at REAL            │
│ ended_at REAL              │
│ input_tokens INTEGER       │
│ output_tokens INTEGER      │
│ cache_read_tokens INTEGER  │
│ estimated_cost_usd REAL    │
└───────────────────────────┘

┌───────────────────────────┐
│  messages                  │ ← 消息历史
│───────────────────────────│
│ id INTEGER PRIMARY KEY     │
│ session_id TEXT             │
│ role TEXT                  │
│ content TEXT               │
│ tool_calls TEXT (JSON)     │
│ tool_call_id TEXT          │
│ timestamp REAL             │
│ token_count INTEGER        │
└───────────────────────────┘

┌───────────────────────────┐
│  messages_fts              │ ← FTS5 全文搜索虚拟表
│───────────────────────────│
│ (content)                  │ ← 通过触发器与 messages 同步
└───────────────────────────┘
```

### 4.2 SSOT 原则落地

| 数据 | 权威来源 | 派生视图 |
|------|----------|----------|
| 持仓状态 | `positions` 表 | API 查询结果、日检报告中的持仓快照、索引层摘要 |
| Thesis | `theses` 表（版本化） | 决策记录中的 thesis_snapshot、索引层摘要 |
| 投资规则 | YAML 配置文件 | 规则检查结果 |
| 市场价格 | 数据源适配器（实时） | `positions.current_price`（缓存） |
| 决策历史 | `decisions` 表 | 记忆系统中的决策记忆、索引层摘要 |
| 用户偏好 | `frozen_memories` 表 (user_profile) | 冻结快照注入 system prompt |
| Agent 状态 | `frozen_memories` 表 (agent_state) | 冻结快照注入 system prompt |
| 对话历史 | `sessions` + `messages` 表 | FTS5 搜索结果、压缩摘要 |

---

## 五、关键流程设计

### 5.1 日检流程（Daily Check）

```
定时触发 / 手动触发 "日检"
  │
  ▼
┌─────────────────────────────┐
│ 1. price_refresh            │  ← 确定性：调用数据源刷新所有价格
│    更新 positions.current_   │
│    price                     │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 2. calc_portfolio_metrics   │  ← 确定性：计算权重、PnL、集中度等
│    生成组合快照              │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 3. rules_check              │  ← 确定性：检查所有规则
│    生成规则结果列表          │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 4. LLM 分析                 │  ← LLM：基于快照+规则结果，生成
│    - 状态评估                │     自然语言评估和建议
│    - Thesis 检查             │
│    - 行动建议                │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 5. 归档                     │  ← 确定性：存储日检报告
│    存储到 daily_checks       │
└─────────────────────────────┘
```

### 5.2 投研分析流程（Coverage Initiation）

```
用户: "帮我研究一下 AAPL"
  │
  ▼
┌─────────────────────────────┐
│ 1. 数据收集                  │  ← Tool 调用：价格、基本面、
│    market data + fundamentals│     财报等多源数据
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 2. 初步分析                  │  ← LLM（强模型）：业务理解、
│    LLM deep analysis         │     竞争优势、风险识别
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 3. 自我质疑                  │  ← LLM：主动找反面证据，
│    self-challenge             │     挑战初步结论 (P2 去偏差)
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 4. 修正与综合                │  ← LLM：整合正反观点，
│    revise & synthesize       │     形成平衡结论
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 5. Thesis 结构化             │  ← LLM + 代码：生成结构化 Thesis
│    generate structured thesis│     包含 Invalidation Conditions
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 6. 用户确认                  │  ← Human-in-the-loop
│    present for review        │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 7. 归档                     │  ← 存储 Thesis + 分析过程
└─────────────────────────────┘
```

### 5.3 决策支持流程

```
用户: "我想买入 TSLA，你觉得怎么样？"
  │
  ▼
┌──────────────────────────────┐
│ 1. 意图识别                   │ ← Agent 理解这是决策支持请求
└──────────┬───────────────────┘
           ▼
┌──────────────────────────────┐
│ 2. 前置检查 (确定性)          │
│  - Thesis 存在吗？            │ → 不存在则建议先研究
│  - 当前价格多少？             │
│  - 拟议操作的组合影响？        │ → 计算买入后的权重、集中度
└──────────┬───────────────────┘
           ▼
┌──────────────────────────────┐
│ 3. 规则检查 (确定性)          │
│  - 是否违反集中度上限？        │
│  - 现金是否足够？             │
│  - 行业集中度是否合规？        │
└──────────┬───────────────────┘
           ▼
┌──────────────────────────────┐
│ 4. LLM 综合分析               │
│  - 回顾 Thesis 当前状态       │
│  - 检索相关记忆（过去决策、教训）│
│  - 生成综合建议               │
│  - 注意：不暴露 avg_cost (P2) │
└──────────┬───────────────────┘
           ▼
┌──────────────────────────────┐
│ 5. 呈现 + 等待确认            │ ← 用户决定是否执行
└──────────────────────────────┘
```

---

## 六、多Agent演进路径

### 6.1 演进触发条件

当以下任一条件满足时，考虑拆分为多 Agent：

1. **工具膨胀**：单 Agent 工具数 > 15，出现注意力稀释
2. **上下文溢出**：单次任务上下文经常接近窗口上限
3. **专业化需求**：不同任务最优模型差异明显且频繁切换
4. **并行需求**：日检中多标的分析需要并行加速
5. **对抗验证**：投研质量要求引入 Bull/Bear 辩论

### 6.2 Phase 2 拆分方案

```
                  ┌──────────────────┐
                  │  Supervisor Agent │
                  │  (Claude Sonnet)  │
                  └────────┬─────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│Research Agent│  │Portfolio     │  │Daily Check   │
│(Claude Opus) │  │  Agent       │  │  Agent       │
│              │  │(Sonnet/Haiku)│  │(Haiku+Code)  │
│- thesis_*    │  │- portfolio_* │  │- price_*     │
│- analysis_*  │  │- calc_*      │  │- rules_*     │
│- memory_*    │  │- decision_*  │  │- daily_*     │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 6.3 平滑迁移设计

为了从单 Agent 平滑过渡到多 Agent，MVP 架构需要预留：

1. **Tool 分组标签**：每个 Tool 标注所属领域（research / portfolio / daily_check），未来按分组拆分
2. **标准化 Tool 接口**：确保 Tool 可以不修改地被不同 Agent 调用
3. **共享记忆层**：记忆系统独立于 Agent，多 Agent 共享访问
4. **共享数据层**：所有 Agent 通过 Repository 访问数据库，无直接 DB 耦合
5. **消息协议**：Agent 间通信使用与前端相同的消息协议

---

## 七、非功能性设计

### 7.1 可观测性

| 层面 | 方案 |
|------|------|
| 日志 | 结构化日志（JSON），分级输出 |
| 追踪 | 每次对话生成 trace_id，贯穿所有 Tool 调用 |
| LLM 调用追踪 | 记录每次 LLM 调用的 prompt/response/token 用量/延迟 |
| 指标 | Tool 调用次数、成功率、延迟分布 |

### 7.2 错误处理

```
Tool 调用失败
  │
  ├─ 可重试错误（网络超时、限流）→ 指数退避重试（最多 3 次）
  │
  ├─ 数据源不可用 → 降级到备用数据源 / 返回缓存数据
  │
  ├─ LLM 调用失败 → 降级到 fallback 模型
  │
  └─ 不可恢复错误 → 记录错误、通知用户、归档错误上下文
```

### 7.3 安全设计

- **买卖操作必须人类确认**（P6）
- `avg_cost` 不进入 LLM prompt（P2）
- 敏感信息（API Key 等）通过环境变量注入，不进入代码/配置
- 数据库连接使用最小权限

### 7.4 性能预期（MVP）

| 指标 | 目标 |
|------|------|
| 简单查询响应 | < 3s |
| 日检完成（10 标的） | < 60s |
| 深度研究分析 | < 5 min |
| 价格刷新（全组合） | < 10s |

---

## 八、MVP 实施路线图

### Phase 0：基础骨架（Week 1-2）

- [ ] 项目初始化：pyproject.toml、目录结构、基础配置
- [ ] Pydantic Models：Portfolio、Position、Thesis、Decision、Memory
- [ ] 数据库：SQLite Schema + Repository 层（结构兼容 PostgreSQL 迁移）
- [ ] LLM 抽象层：基类 + 至少 2 个 Provider（Anthropic + OpenAI）

### Phase 1：Agent 内核（Week 3-4）

- [ ] Core Agent 主循环（Plan → Execute → Reflect）
- [ ] Tool 基类 + Registry + 自动注册
- [ ] MVP Tools：portfolio_query、price_refresh、rules_check
- [ ] 会话管理（上下文组装）

### Phase 2：业务功能（Week 5-6）

- [ ] 完整组合管理 Tools
- [ ] 规则引擎
- [ ] 日检流程
- [ ] Thesis 管理 Tools

### Phase 3：记忆与前端（Week 7-8）

- [ ] 冻结快照记忆（USER_PROFILE + AGENT_STATE）
- [ ] 索引层动态生成（持仓/Thesis/决策摘要视图）
- [ ] 检索层（混合检索 + FTS5 会话搜索）
- [ ] 上下文压缩模块（四阶段算法）
- [ ] 记忆衰减引擎
- [ ] CLI 前端
- [ ] FastAPI Gateway

### Phase 4：打磨与验证（Week 9-10）

- [ ] 端到端测试
- [ ] Prompt 调优
- [ ] 性能优化
- [ ] 日检自动化（cron）

---

## 九、待决策项

以下问题需要在实施过程中进一步讨论确定：

1. **部署模型**：本地开发 + 云部署？还是纯本地？
2. ~~**PostgreSQL 托管**~~ → MVP 已决定用 SQLite。Phase 2 迁移时再讨论 PG 托管方案
3. **CLI 框架选择**：Rich + Typer？还是自定义 REPL？
4. **Web UI 技术栈**：Next.js？还是 MVP 阶段暂不做？
5. **日检定时方案**：OS cron？还是 APScheduler 进程内？
6. **LLM Prompt 管理**：代码内模板？还是独立的 Prompt 仓库？
7. **A 股数据源**：JQData 还是其他替代方案？
8. **向量嵌入模型**：OpenAI text-embedding-3-small？还是本地模型（sentence-transformers）？
9. **冻结记忆格式**：Markdown 文件（参考 Hermes）？还是数据库行？当前设计用数据库行，因为更适合程序化操作
10. **SQLite → PostgreSQL 迁移时机**：多用户需求出现时？还是记忆量超过阈值时？
