# Agent 记忆系统与数据存储结构深度研究

> 调研日期：2026-04-09
> 目的：为 Invest_Agent v2 的记忆系统和存储架构设计提供依据
> 前置文档：[多Agent架构研究](multi-agent-architecture-research.md)

---

## 一、核心问题

Agent 记忆系统面临一个根本矛盾：**记忆越多，上下文越大，token 成本越高，性能反而越差**。

2025 年企业 AI 的数据显示：

- 近 **65%** 的故障归因于上下文漂移和记忆丢失，而非上下文窗口耗尽
- 信息被"埋"在中间位置会导致 **30%+** 的准确率下降（"lost-in-the-middle" 效应）
- 不受控的记忆增长导致 LOCOMO 基准测试性能从 0.455 降至 0.05

因此，核心挑战不是"如何存更多"，而是**如何在正确的时机，以正确的密度，将正确的信息送入上下文**。

---

## 二、主流项目的记忆架构详解

### 2.1 Hermes Agent — 三层冻结快照架构

Hermes 是目前记忆系统设计最精细的开源 Agent 之一。

#### 2.1.1 三层记忆结构

**Layer 1: 冻结快照记忆（Frozen Snapshot Memory）**

- `MEMORY.md`（~800 tokens / 2,200 字符上限）：环境事实、项目约定、经验教训
- `USER.md`（~500 tokens / 1,375 字符上限）：用户偏好、沟通风格
- 存储位置：`~/.hermes/memories/`
- 条目用 `§`（section sign）分隔符隔开
- 会话启动时注入 system prompt，之后**冻结不变**

**Layer 2: 情景记忆（Episodic Memory）**

- 使用 **ChromaDB** 向量存储
- 记录过去任务执行的时间戳记录（尝试了什么、成功了什么、失败了什么）
- 新任务开始时进行**语义相似度搜索**
- 实现了自我改进机制 — Agent 记住**策略**而非仅仅事实

**Layer 3: 会话搜索（Session Search）**

- **SQLite FTS5** 全文索引，覆盖所有历史对话
- 结果按需由 LLM 摘要
- 提供"无限情景回忆容量"

#### 2.1.2 冻结快照的设计原理

这是 Hermes 最关键的设计决策：

1. **保护 LLM 前缀缓存（Prefix Cache）**
   - Anthropic prompt caching 要求 system prompt 前缀稳定不变
   - 冻结的记忆 = 稳定的前缀 = 缓存命中率最高
   - 输入 token 成本降低约 **75%**

2. **防止会话中途不稳定**
   - 会话中对记忆的修改**立即写盘**但**下次会话才生效**
   - Tool 返回结果可以显示修改后的状态，但 system prompt 不变
   - 避免 Agent 行为因 prompt 突变而不可预测

3. **Agent 可见容量**
   - Agent 能看到剩余容量百分比和字符数
   - 自主决定何时清理旧条目、何时添加新条目

#### 2.1.3 记忆操作工具

Memory Tool 支持三种操作：

- `add`：插入新条目
- `replace`：子串匹配替换（只需提供唯一子串即可定位条目）
- `remove`：子串匹配删除

安全措施：所有写入前进行安全扫描，阻止 prompt 注入、凭据泄露、SSH 后门等恶意内容。

#### 2.1.4 Prompt 组装顺序

Hermes 的 system prompt 按以下 10 层组装（在 `prompt_builder.py` 中实现）：

```
1. Agent 身份 (SOUL.md 或默认)
2. Tool 使用行为指导
3. Honcho 静态块（可选）
4. 自定义 system message（可选）
5. 冻结 MEMORY 快照            ← 持久记忆
6. 冻结 USER 快照               ← 用户档案
7. Skills 索引（仅名称+描述）    ← 渐进披露
8. 项目上下文文件
9. 时间戳 / 会话ID
10. 平台提示
```

关键设计点：
- 只有一种项目上下文会加载（优先级：.hermes.md → AGENTS.md → CLAUDE.md → .cursorrules）
- 所有上下文文件进行安全扫描和截断（20,000 字符上限，70/20 头/尾比例）
- Skills 索引是紧凑的元数据列表，不是完整内容

#### 2.1.5 双层上下文压缩系统

**第一层：Gateway 安全网（85% 阈值）**

- 在 Agent 处理消息之前运行
- 使用粗粒度字符估算
- 防止长期积累的会话（如 Telegram/Discord 过夜消息）导致 API 失败

**第二层：Agent 压缩器（50% 阈值，可配置）**

四阶段压缩算法：

```
Phase 1: 裁剪旧 Tool 输出
  - 无 LLM 调用，最便宜
  - 保护尾部的 Tool 结果不动
  - 旧的长输出替换为 "[Old tool output cleared to save context space]"

Phase 2: 确定边界
  - 保护头 3 条消息（system + 首次交互）
  - 尾部保护：按 token 预算向后走，或至少保护最后 20 条
  - 边界对齐：不拆分 tool_call/tool_result 组

Phase 3: 结构化摘要（LLM 调用）
  - 对中间部分生成结构化摘要
  - 模板：Goal / Constraints / Progress(Done/InProgress/Blocked) / 
          Key Decisions / Relevant Files / Next Steps / Critical Context
  - 摘要 token 预算 = 被压缩内容 × 20%，最少 2000，最多 12000

Phase 4: 组装
  - 头部 + 摘要 + 尾部
  - 清理孤立的 tool_call/tool_result 对
```

**迭代式再压缩**：后续压缩是**更新**上一次摘要，而非从头生成。已完成的工作从 "In Progress" 移至 "Done"，新进展被追加。

#### 2.1.6 Prompt Caching 策略

Anthropic 允许每次请求最多 4 个 `cache_control` 断点。Hermes 使用 "system_and_3" 策略：

```
断点 1: System prompt          (所有轮次稳定)
断点 2: 倒数第 3 条消息         ─┐
断点 3: 倒数第 2 条消息          ├─ 滚动窗口
断点 4: 最后 1 条消息           ─┘
```

TTL 可配置为 5 分钟（默认）或 1 小时（长会话）。

#### 2.1.7 会话存储 Schema

Hermes 使用 SQLite（WAL 模式）存储会话：

```
~/.hermes/state.db
├── sessions          — 会话元数据、token 计数、计费
├── messages          — 每个会话的完整消息历史
├── messages_fts      — FTS5 虚拟表（全文搜索）
└── schema_version    — 迁移版本追踪
```

Sessions 表包含：id、source、user_id、model、system_prompt、parent_session_id（会话谱系）、token 统计（input/output/cache_read/cache_write/reasoning）、计费信息。

Messages 表包含：id、session_id、role、content、tool_calls（JSON）、timestamp、token_count、finish_reason、reasoning。

FTS5 通过三个触发器保持同步：INSERT、UPDATE、DELETE 时自动更新索引。

写入竞争处理：短超时（1秒）+ 应用层随机抖动重试（20-150ms，最多 15 次）+ BEGIN IMMEDIATE 事务。

#### 2.1.8 Skills 系统 — 渐进披露的典型实现

Skills 是按需加载的知识文档，存储在 `~/.hermes/skills/`：

- System prompt 只注入 Skills **索引**（名称 + 一句话描述）
- Agent 判断当前任务匹配某个 Skill 时，调用 `skill_view(name)` 加载完整内容
- Agent 可以从经验中自主创建新 Skills
- 遵循 `SKILL.md` Markdown 格式，包含元数据、步骤、陷阱说明

---

### 2.2 Claude Code — 无数据库文件记忆 + 子Agent隔离

#### 2.2.1 三层记忆

**Auto-Memory（长期存储）**

- 每个记忆条目是独立的 Markdown 文件，带 frontmatter（Name/Description/Type）
- `MEMORY.md` 索引文件（200 行上限），超出时截断旧条目
- 每轮对话后，专门的**维护子 Agent** 提取洞察并更新记忆
- 维护子 Agent 工具受限：只能读 Bash + 操作 memory 目录

**Agent-Tool Memory（领域专用）**

- 自定义子 Agent 有自己的命名空间记忆 `.claude/agent-memory/[agent-name]/`
- 专业 Agent 积累领域经验，不污染主项目记忆

**检索机制**

- 扫描记忆文件的 frontmatter
- 发送列表给 Claude 3.5 Sonnet（256 token 限制 + 严格 JSON schema）
- 模型选出最多 5 个相关文件注入上下文
- **不是向量检索，而是 LLM 判断相关性** — 更简单，无需向量基础设施

#### 2.2.2 上下文压缩（Compaction）

三种模式：
- **微压缩**：大块 Tool 输出早期卸载
- **自动压缩**：上下文接近容量时触发
- **手动压缩**：用户在任务边界主动触发

每 5,000 tokens 更新一次 session notes（需累积 10,000 tokens + 三次 Tool 调用后才开始）。

压缩保留：最近文件、todos、继续指令 — 而非仅仅做摘要。这确保 Agent 在压缩后能延续工作动量。

---

### 2.3 MemGPT / Letta — OS 式虚拟上下文管理

#### 2.3.1 核心思想

类比操作系统的虚拟内存管理：

| 操作系统 | MemGPT |
|----------|--------|
| 主存/RAM | 主上下文（context window 内） |
| 磁盘存储 | 外部上下文（数据库/文件） |
| 虚拟内存 | 虚拟上下文（Agent 感知的"无限"记忆） |
| 页面调度 | 记忆检索/存储的 function calling |

#### 2.3.2 两层记忆

**Tier 1（主上下文内）**：
- "Human" 块：用户信息和偏好
- "Persona" 块：Agent 身份和行为指南
- 有字符/token 限制和可选描述
- 这些是**离散的功能单元**

**Tier 2（外部上下文）**：
- 对话记忆、归档记忆、外部文件
- 通过 function calling 按需检索

#### 2.3.3 关键洞察

Agent **自己决定**何时换入/换出记忆。这是最像"操作系统"的方案 — 记忆管理不是被动的 RAG 检索，而是 Agent 的**主动行为**。

---

### 2.4 Mem0 — 生产级记忆层

#### 2.4.1 架构哲学

**将事实与记忆分离**：
- 关键事实（提醒、deadline、余额）→ 关系型数据库（真源）
- 行为模式、偏好、上下文 → 向量数据库（个性化记忆）

#### 2.4.2 关键指标

- Token 减少 80%，保留核心细节
- 对比 OpenAI Memory：质量高 26%，token 少 90%
- 100,000+ 开发者在用
- SOC 2 / HIPAA 合规

#### 2.4.3 实现方式

- LLM 从对话中提取重要事实 → 嵌入 → 存入向量数据库 → 按需检索
- 记忆带时间戳、版本化、可导出
- 内置 TTL 和 size 追踪
- 支持 Kubernetes、air-gapped 部署

---

## 三、学术前沿方案

### 3.1 SimpleMem — 语义无损压缩（2026.01）

**三阶段管线**实现 **30 倍** token 压缩：

1. **语义结构化压缩（Semantic Structured Compression）**
   - 熵感知过滤，消除低信息量内容
   - 消解共指（coreference resolution）
   - 添加绝对时间戳（替代相对时间表达）
   - 产出：紧凑的多视角索引记忆单元

2. **在线语义合成（Online Semantic Synthesis）**
   - 会话内实时运行
   - 合并相关上下文为统一抽象表示
   - 消除跨消息的冗余

3. **意图感知检索（Intent-Aware Retrieval Planning）**
   - 推断搜索意图
   - 动态确定检索深度和范围
   - 根据查询复杂度调整检索策略

**结果**：
- F1 提升 26.4%（对比 Mem0）
- Token 从 16,910 降至 531（30x 压缩）
- 检索速度提升 50.2%（388.3s vs 583.4s）

### 3.2 xMemory — 层次化语义记忆（2026.02）

**四层结构**：Messages → Episodes → Semantics → Themes

**自顶向下检索**：
1. 先在 Themes 层选择相关主题
2. 展开到 Semantics 层获取语义单元
3. 仅在必要时展开到原始 Episodes/Messages

**核心优势**：
- Token 从 ~9,000 降至 ~4,700
- 解决了 RAG 的"检索坍塌"问题 — 传统 RAG 返回大量语义相似但内容重复的片段
- "解耦到聚合" — 先拆分记忆为语义组件，再按层次组织

**代价**：需要大量后台处理来检测对话边界、摘要情景、合成主题（"写税"）。

### 3.3 MEMORA — 谐波记忆表示（2026.02，微软）

**核心创新**：

- **主抽象（Primary Abstractions）**：索引具体记忆值，合并相关更新为统一条目
- **线索锚点（Cue Anchors）**：扩展检索访问路径，连接不同方面的相关记忆
- 检索策略主动利用记忆之间的**连接关系**，超越单纯的语义相似度

**理论贡献**：标准 RAG 和知识图谱记忆都是 MEMORA 的特例。

**解决的根本矛盾**：
- 具体但碎片化（存原始交互/原子事实 → 检索噪声大）
- 高效但模糊（压缩为摘要 → 丢失关键细节）
- MEMORA 在两者之间**结构性平衡**

### 3.4 AgeMem — Agent 自主记忆管理（2026.01）

将记忆操作暴露为 Tool：Agent 通过强化学习自主决定存储、检索、更新、摘要、丢弃。

三阶段渐进式强化学习训练策略。Agent 像人一样自主管理自己的记忆。

---

## 四、渐进披露（Progressive Disclosure）机制

### 4.1 核心原理

渐进披露是 LLM 时代的基础工程哲学：**在固定上下文预算内，按需加载信息，而非预加载全部**。

类比人类认知：你不会背下百科全书，而是记住目录，需要时查阅。

Hermes Agent 的创造者将其总结为："deliver the right information, at the right time, at the right density, into the context."

### 4.2 三层模型

```
┌────────────────────────────────────────────────────────────┐
│  Tier 1: 元数据层（Metadata Tier）                          │
│  占预算 1-5%                                                │
│  每条 50-200 字符                                           │
│  内容：名称、一句话描述、状态标签                              │
│  目的：创建语义索引，让 Agent 知道"有什么可用"                 │
├────────────────────────────────────────────────────────────┤
│  Tier 2: 激活层（Activation Tier）                          │
│  占预算 10-30%                                              │
│  每条 500-5,000 词                                          │
│  内容：完整文档、详细规则、完整 Thesis                         │
│  触发：Agent 判断与当前任务相关时加载                          │
├────────────────────────────────────────────────────────────┤
│  Tier 3: 按需层（On-Demand Tier）                           │
│  不占常驻预算                                                │
│  内容：源文件、历史数据、原始记录                              │
│  触发：通过 Tool 调用获取                                     │
└────────────────────────────────────────────────────────────┘
```

**效率对比**：

| 模式 | 100 个条目的 token 消耗 | 比率 |
|------|------------------------|------|
| 急切加载 | 100 × 1,000 = 100,000 | 100% |
| 渐进披露 | 100 × 50 + 1 × 1,000 = 6,000 | 6% |
| 节省 | — | **94%** |

### 4.3 深度分级检索

参考 memsearch 的实现：

- **L1 搜索**：返回 top-K 相关片段 + 相关性评分（始终的起点）
- **L2 展开**：当片段需要更多上下文时，加载完整 Markdown 段落
- **L3 原文**：需要原始对话时，加载完整历史轮次

Agent 根据查询需求**自主决定**检索深度。

### 4.4 Hermes 中的渐进披露实践

**Skills 系统**是最佳案例：
- System Prompt 只注入 Skills 索引（名称 + 一句话描述）
- Agent 判断匹配时调用 `skill_view(name)` 加载完整内容
- 等价于：动态链接（dynamic linking），而非静态链接

**Tool 描述**：
- 只有 Tool 的 Schema 描述进入上下文
- Tool 的实现代码按需调用

**记忆冻结快照**：
- 最关键的 ~1,300 tokens 常驻
- 其余所有记忆通过 Tool 按需检索

---

## 五、记忆衰减与遗忘机制

### 5.1 为什么需要遗忘

不受控的记忆增长导致：
- **检索噪声**：无关记忆稀释搜索结果
- **过时信息**：旧的市场观察可能误导决策
- **冲突记忆**：早期判断与后期修正共存
- **存储膨胀**：向量数据库性能随规模下降

### 5.2 主流遗忘策略

**FadeMem（2026.01）— 仿生遗忘**：
- 双层记忆层级 + 自适应指数衰减
- 衰减率由三因素调节：语义相关性、访问频率、时间模式
- 重要记忆无限期保留，偶然交互快速衰减
- 结果：存储减少 45%，多跳推理能力不变

**Oblivion（2026.04）— 衰减驱动激活**：
- 将记忆控制解耦为**读路径**和**写路径**
- 避免"始终在线"的检索开销 — 衰减的记忆不参与检索
- 实现层次化记忆组织

**人类启发**：
- 遵循 Ebbinghaus 遗忘曲线 — 记忆指数衰减，除非被强化
- 重要记忆（高情感价值/高频访问）衰减极慢
- 不重要的日常交互快速淡化

### 5.3 投研场景的领域特化衰减

投研记忆的衰减策略应该**领域特化**，而非统一衰减：

| 衰减等级 | 类型 | 半衰期 | 示例 |
|----------|------|--------|------|
| 永不衰减 | 投资原则、核心教训 | ∞ | "不要追涨杀跌"、IC 触发条件 |
| 极慢衰减 | Thesis 核心、重大决策 | 180 天 | AAPL thesis、卖出 BABA 的理由 |
| 正常衰减 | 市场观察、日检洞察 | 30 天 | "美联储暂停加息"、某日日检发现 |
| 快速衰减 | 临时查询、格式请求 | 7 天 | "AAPL 今天什么价"、"帮我算一下" |

---

## 六、存储架构选型

### 6.1 方案对比

| 维度 | PostgreSQL + pgvector | SQLite + ChromaDB | SQLite + 文件系统 |
|------|----------------------|--------------------|--------------------|
| 代表项目 | Mem0（生产版） | Hermes Agent | Claude Code |
| 向量检索性能 | 3,000 QPS@1M 向量 | 2,000 QPS（RAM 限制） | 无向量（LLM 判断） |
| 并发支持 | 优秀（MVCC） | 有限（WAL 单写） | 文件级锁 |
| 关系查询 | 原生 SQL + JSON 列 | 原生 SQL + FTS5 | 无 |
| 运维复杂度 | 中等（需独立服务） | 极低（嵌入式） | 极低 |
| 扩展性 | 十亿级向量 | 百万级 | 千级文件 |
| ACID 事务 | 完整支持 | 支持 | 不支持 |
| 混合查询 | 向量 + 关系 JOIN | 向量 + FTS5 分离 | N/A |
| 适用阶段 | 生产/多用户 | MVP/单用户 | 极简原型 |

### 6.2 MVP 推荐：SQLite（嵌入式）

理由：
- 单用户 MVP 不需要独立数据库服务
- SQLite 嵌入式零运维，Hermes 已验证可行
- FTS5 全文搜索覆盖会话检索需求
- WAL 模式支持基本并发
- 后期可平滑迁移到 PostgreSQL

### 6.3 Phase 2+ 推荐：PostgreSQL + pgvector

理由：
- 多用户需要并发写入
- 向量 + 关系混合查询（如"最近 30 天关于 AAPL 的 Thesis 变更"）
- SSOT 原则要求 ACID 事务保障
- pgvector 性能远超 ChromaDB（并发下 9.81s vs 23.08s）

### 6.4 向量嵌入选型

MVP 阶段使用 sqlite-vec 或内存中的轻量向量检索，避免引入 ChromaDB 等外部依赖。

---

## 七、关键设计决策总结

### 7.1 必须采用的设计

| 设计 | 来源 | 理由 |
|------|------|------|
| 冻结快照记忆 | Hermes | 保护前缀缓存，降低 75% token 成本 |
| 严格容量限制 | Hermes | 防止记忆膨胀，Agent 自主管理 |
| 结构化压缩摘要 | Hermes | 保留关键信息（Goal/Progress/Decisions） |
| 渐进披露 | Hermes/通用 | 94% token 节省 |
| 事实与记忆分离 | Mem0 | SSOT 原则落地 |
| 领域特化衰减 | FadeMem + 自定义 | 投研场景需要差异化保留策略 |

### 7.2 可选/后期的设计

| 设计 | 来源 | 考虑时机 |
|------|------|----------|
| 层次化语义记忆 | xMemory | 记忆量 > 10,000 条时 |
| 线索锚点 | MEMORA | 需要跨维度关联检索时 |
| Agent 自主记忆管理 | AgeMem | 系统足够成熟可以训练时 |
| OS 式虚拟上下文 | MemGPT | 上下文压力非常大时 |

---

## 参考资料

1. Hermes Agent 官方文档 — [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory/), [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly), [Context Compression](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching), [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture/), [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage)
2. Claude Code — Database-Free Memory Architecture (Medium, 2026.04)
3. MemGPT/Letta — [Letta Docs](https://docs.letta.com/guides/agents/architectures/memgpt), [Virtual Context Management](https://www.leoniemonigatti.com/blog/memgpt.html)
4. Mem0 — [State of AI Agent Memory 2026](https://mem0.ai/blog/state-of-ai-agent-memory-2026)
5. SimpleMem — [arXiv:2601.02553](https://arxiv.org/abs/2601.02553v3)
6. xMemory — [VentureBeat](https://venturebeat.com/orchestration/how-xmemory-cuts-token-costs-and-context-bloat-in-ai-agents/)
7. MEMORA — [arXiv:2602.03315](https://arxiv.org/abs/2602.03315), Microsoft Research
8. FadeMem — [arXiv:2601.18642](https://arxiv.org/abs/2601.18642v1)
9. Oblivion — [arXiv:2604.00131](https://arxiv.org/abs/2604.00131)
10. AgeMem — [arXiv:2601.01885](https://arxiv.org/abs/2601.01885v1)
11. Progressive Disclosure Pattern — [Agentic Engineering Book](https://jayminwest.com/agentic-engineering-book/6-patterns/7-progressive-disclosure)
12. AI Agent Context Compression Strategies — [Zylos Research](https://zylos.ai/research/2026-02-28-ai-agent-context-compression-strategies)
