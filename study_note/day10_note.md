# Day 10 学习笔记 — 评估系统深度剖析 + 代码 Walk-Through + 终极查漏补缺

> 日期：2026-03-03
>
> **阅读前提**：已完成 Day 1-9 全部学习。Day 9 完成了面试 STAR 话术打磨。
>
> **今日定位**：10 天学习计划的**收官之战**。前 9 天我们"建造了武器库"，今天要"实弹打靶"。

---

## 写在前面：为什么 Day 10 最重要？

```
Day 1-8：输入阶段（读代码、学原理、跑实验）
Day 9  ：组织阶段（把零散知识变成 STAR 故事）
Day 10 ：输出阶段（模拟面试 + 补齐最后的知识盲区）

面试是一场限时的"知识输出考试"：
  你不是要证明你"学了很多"，而是要证明你"能说清楚"。
  Day 10 的目标就是：把最后一块拼图——评估系统——补上，
  然后做三次关键代码 Walk-Through，
  最后模拟一场完整的 30 分钟面试。
```

今天有三个核心模块：
1. **评估系统深度剖析**（Day 9 提到但未深入，今天补齐）
2. **三个关键代码文件 Walk-Through**（面试官说"带我看看代码"时的应对方案）
3. **终极模拟面试 + 查漏补缺**（30 分钟不间断自检）

---

## 一、评估系统深度剖析

### 1.1 为什么评估系统是面试加分项？

```
面试官视角：
  "做了一个 RAG 系统" → 及格
  "做了 RAG + 知识图谱 + 多 Agent" → 良好
  "做了 RAG + 知识图谱 + 多 Agent + 完整评估体系" → 优秀

评估系统展示了三种能力：
  1. 你知道怎么量化"好不好" → 工程素养
  2. 你设计了多维度指标体系 → 系统思维
  3. 你能用数据说话而非感觉 → 科学态度
```

**面试中提到评估系统的时机**：
- 面试官问"效果怎么样？" → 直接展开评估框架
- 面试官问"怎么比较不同 Agent？" → 展开 CompositeEvaluator
- 面试官问"你怎么知道改进有效？" → 展开指标量化

> 👉 **[QA指路] 如果面试官问“你的评估数据集有多大？怎么生成标准答案？”，详见 `day10_note_QA.md` 的 Q9**

---

### 1.2 评估系统的整体架构（从零理解）

先看全貌，再看细节。评估系统的目录结构是这样的：

```
graphrag_agent/evaluation/
├── core/                          # 🏗️ 核心框架（地基）
│   ├── base_metric.py            # 所有指标的抽象基类
│   ├── base_evaluator.py         # 所有评估器的抽象基类
│   └── evaluation_data.py        # 数据结构定义
│
├── metrics/                       # 📏 具体指标实现（18 个指标）
│   ├── answer_metrics.py         # ExactMatch, F1Score
│   ├── llm_metrics.py            # 连贯性、一致性、全面性、LLM 综合评分
│   ├── retrieval_metrics.py      # 检索精确率、利用率、延迟、Chunk 利用率
│   ├── graph_metrics.py          # 实体覆盖率、图覆盖率、关系利用率等
│   └── deep_search_metrics.py    # 推理连贯性、深度、迭代改进、图谱利用率
│
├── evaluators/                    # 🎯 评估器（编排指标的执行）
│   ├── answer_evaluator.py       # 答案质量评估器
│   ├── retrieval_evaluator.py    # 检索性能评估器
│   └── composite_evaluator.py    # 组合评估器（总指挥）
│
├── evaluator_config/              # ⚙️ 配置管理
│   ├── evaluatorConfig.py        # 评估器配置封装
│   └── agent_evaluation_config.py # 每种 Agent 的指标配置
│
├── preprocessing/                 # 🧹 数据预处理
│   ├── reference_extractor.py    # 从回答中提取引用数据
│   └── text_cleaner.py           # 清理思考过程和引用部分
│
├── utils/                         # 🔧 工具函数
│   ├── text_utils.py             # 文本标准化、P/R/F1 计算
│   ├── eval_utils.py             # Agent 加载、依赖注入
│   └── ...
│
└── test/                          # 🧪 测试脚本
    ├── evaluate_all_agents.py    # 多 Agent 对比评估
    ├── evaluate_*_agent.py       # 单 Agent 评估（5 个脚本）
    ├── questions.json            # 测试问题集
    └── answer.json               # 标准答案集
```

**关键架构洞察**：评估系统本身也是分层设计的，和主系统的分层逻辑（图谱构建→搜索→Agent）形成了优雅的呼应。

---

### 1.3 核心类继承体系（面试必画图）

```
                    BaseMetric (ABC)
                    ├── metric_name = "base"
                    ├── calculate_metric(data) → (Dict, List)  [抽象方法]
                    ├── get_llm_fallback_score(prompt)          [三层回退的核心]
                    └── log(message)
                        │
        ┌───────────────┼───────────────────────────┐
        │               │                           │
   答案指标类        检索指标类                  深度研究指标类
   ├── ExactMatch   ├── RetrievalPrecision      ├── ReasoningCoherence
   ├── F1Score      ├── RetrievalUtilization     ├── ReasoningDepth
   ├── Response...  ├── RetrievalLatency         ├── IterativeImprovement
   ├── Factual...   ├── ChunkUtilization         └── KnowledgeGraph...
   ├── Comprehen..  ├── EntityCoverage
   └── LLMGraph...  ├── GraphCoverage
                    ├── RelationshipUtil...
                    ├── CommunityRelevance
                    └── SubgraphQuality


                    BaseEvaluator (ABC)
                    ├── evaluate(data) → Dict         [抽象方法]
                    ├── _collect_metrics()             [自动发现机制]
                    ├── save_metric_score()
                    └── format_results_table()
                        │
                ┌───────┴────────┐
                │                │
         AnswerEvaluator    GraphRAGRetrievalEvaluator
                │                │
                └────────┬───────┘
                         │
              CompositeGraphRAGEvaluator （总指挥，不继承 BaseEvaluator）
              ├── 持有 AnswerEvaluator + RetrievalEvaluator
              ├── evaluate_with_golden_answers()
              ├── compare_agents_with_golden_answers()
              └── format_comparison_table()
```

**面试话术**："评估框架采用策略模式——`BaseMetric` 定义统一接口，18 个具体指标各自实现计算逻辑。`BaseEvaluator` 通过 `_collect_metrics()` 方法自动发现所有注册的指标子类，实现了开放-封闭原则：添加新指标只需继承 `BaseMetric` 并设置 `metric_name`，无需修改任何现有代码。"

> 👉 **[QA指路] 这其中 `_collect_metrics()` 用的 `__subclasses__()` 方法有什么陷阱？详见 `day10_note_QA.md` 的 Q2**
> 👉 **[QA指路] 如果面试官让你现场写一个新的评估指标，你怎么做？详见 `day10_note_QA.md` 的 Q5**

---

### 1.4 三层评分回退机制（设计亮点）

这是评估系统最值得说的设计点之一。每个指标不是只有一种评分方式，而是有三层回退保障：

```
评分流程（以 ExactMatch 为例）：

第 1 层：规则评分
  ├── 标准化两个答案（去标点、转小写、jieba 分词）
  ├── 完全匹配 → 1.0
  └── 不匹配 → 计算 Jaccard 内容相似度
         │
         ├── 相似度 ≥ 0.7 → 直接映射为 0.7~1.0 的分数
         │
         └── 相似度 < 0.7 → 触发第 2 层

第 2 层：LLM 回退评分
  ├── 构造 Prompt："比较两个答案的内容等价性，给 0-1 分数"
  ├── 调用 self.llm.invoke(prompt)
  ├── 正则提取响应中的数字
  └── 确保在 [0, 1] 范围内
         │
         └── LLM 调用失败 → 触发第 3 层

第 3 层：默认分数
  └── 返回 default_score（通常为 0.5）
```

**为什么需要三层？**

```
只用规则：
  "旷课30学时会被退学" vs "连续旷课达到30学时的学生将被开除学籍"
  → 规则判定不匹配（F1 低），但语义上完全等价

只用 LLM：
  10000 个实体 × 每个都要调 LLM → API 费用爆炸
  LLM 偶尔抽风返回乱码 → 评分不可控

三层组合：
  规则做快速筛选 → LLM 做语义兜底 → 默认分数做容错
  既省钱又准确又鲁棒
```

**代码关键位置**：`base_metric.py:54-91` 的 `get_llm_fallback_score()` 方法。

---

### 1.5 四大评估维度详解

> 👉 **[QA指路] 评估指标多达 18 个，这些指标之间有没有冲突？你怎么做最终判断？详见 `day10_note_QA.md` 的 Q1**

#### 维度一：答案质量评估（6 个指标）

| 指标 | metric_name | 评分方式 | 核心逻辑 |
|------|------------|---------|---------|
| ExactMatch | `em` | 规则+LLM | 标准化后精确比对，不完全匹配用 Jaccard + LLM 回退 |
| F1Score | `f1` | 规则+LLM | jieba 中文分词 → 去停用词 → 计算 Precision/Recall/F1 |
| ResponseCoherence | `response_coherence` | 纯 LLM | 评估回答的逻辑清晰度、结构化程度 |
| FactualConsistency | `factual_consistency` | 纯 LLM | 检查回答是否自相矛盾，信息是否准确 |
| ComprehensiveAnswer | `answer_comprehensiveness` | 纯 LLM | 评估回答是否全面覆盖问题所有方面 |
| LLMGraphRagEvaluator | `llm_evaluation` | 纯 LLM | **4 维加权综合评分**（核心指标） |

> 👉 **[QA指路] `F1Score` 评估指标的具体计算流水线（分词、去停用词、Precision/Recall/F1、规则+LLM回退）是怎么运作的？详见 `day10_note_QA.md` 的 Q11**

**重点展开：LLMGraphRagEvaluator（llm_evaluation）**

这是最重要的综合指标，评估 4 个维度并加权：

```python
self.aspect_weights = {
    "comprehensiveness": 0.3,   # 全面性 —— 回答覆盖了多少方面
    "relativeness": 0.25,       # 相关性 —— 回答是否切题
    "empowerment": 0.25,        # 增强理解 —— 是否帮助读者做出判断
    "directness": 0.2           # 直接性 —— 是否直击要害
}
```

**为什么全面性权重最高（0.3）？** 因为 GraphRAG 的核心价值就是比传统 RAG 更全面——通过知识图谱的多跳关联发现更多相关信息。如果全面性不高，说明图谱增强没有生效。

**评估 Prompt 的设计**（`llm_metrics.py:486-522`）：
- 返回 JSON 格式 `{"comprehensiveness": 0.X, "relativeness": 0.X, ...}`
- 包含 `reasoning` 字段让 LLM 解释评分理由
- 用正则提取 JSON 块，失败则回退到默认 0.5

> 👉 **[QA指路] `LLMGraphRagEvaluator` 用 LLM 评估 LLM，这有没有循环偏差？详见 `day10_note_QA.md` 的 Q6**

---

#### 维度二：检索性能评估（4 个指标）

| 指标 | metric_name | 核心逻辑 |
|------|------------|---------|
| RetrievalPrecision | `retrieval_precision` | 检索到的实体中，有多少被回答引用了 |
| RetrievalUtilization | `retrieval_utilization` | 回答引用的实体中，有多少来自检索结果 |
| RetrievalLatency | `retrieval_latency` | 从提问到返回的耗时（秒），越低越好 |
| ChunkUtilization | `chunk_utilization` | 检索到的文本 chunk 有多少内容被答案使用（仅 NaiveAgent） |

**Precision vs Utilization 的区别（面试常问）**：

```
假设场景：
  检索到了 10 个实体：[A, B, C, D, E, F, G, H, I, J]
  回答中引用了 3 个实体：[A, B, K]（K 不在检索结果中）

RetrievalPrecision = 引用中来自检索的 / 引用总数 = 2/3 = 0.67
  → 衡量"检索结果有没有被用上"

RetrievalUtilization = 被引用的检索结果 / 检索总数 = 2/10 = 0.2
  → 衡量"检索了那么多，到底用了多少"

启示：
  Precision 低 → 检索结果不够相关，LLM 不得不"脑补"
  Utilization 低 → 检索数量过多，大量无用信息
  两者都高 → 检索精准且高效
```

---

#### 维度三：图谱质量评估（5 个指标）

| 指标 | metric_name | 核心逻辑 |
|------|------------|---------|
| EntityCoverage | `entity_coverage` | 检索到的实体和问题关键词的匹配度 |
| GraphCoverage | `graph_coverage` | **三维组合**：结构分 + 相关性分 + 连通性分 |
| RelationshipUtilization | `relationship_utilization` | **三维组合**：数量分 + 质量分 + 相关性分 |
| CommunityRelevance | `community_relevance` | 检索到的社区摘要和问题的语义相关度 |
| SubgraphQuality | `subgraph_quality` | 提取子图的密度（边/最大可能边）和连通性 |

**重点展开：GraphCoverage（graph_coverage）**

这是最复杂的图谱指标，计算三个子分数的加权平均：

```
GraphCoverage 三维分解：

1. Structure Score（结构分）
   ├── 基于检索到的实体数量和关系数量
   ├── 有实体有关系 → 高分
   └── 只有实体没关系 → 中低分

2. Relevance Score（相关性分）
   ├── 从问题中提取关键词
   ├── 在实体名称、描述、关系描述中搜索关键词
   └── 匹配率越高 → 分数越高

3. Connectivity Score（连通性分）
   ├── 检查实体之间是否有路径连接
   ├── 在 Neo4j 中执行 shortestPath 查询
   └── 连通比例越高 → 分数越高
   （这个分数直接反映了图谱的"推理跳板"能力）
```

---

#### 维度四：深度研究专项评估（4 个指标）

这些指标只对 DeepResearchAgent 和 FusionAgent 生效：

| 指标 | metric_name | 核心逻辑 |
|------|------------|---------|
| ReasoningCoherence | `reasoning_coherence` | 从 `<think>` 标签中提取推理过程，评估逻辑连贯性 |
| ReasoningDepth | `reasoning_depth` | 分析搜索查询数量、思考层次、信息提取质量 |
| IterativeImprovement | `iterative_improvement` | 对比第一轮和最后一轮的查询策略和信息质量 |
| KnowledgeGraphUtilization | `knowledge_graph_utilization` | 回答中是否引用了实体、关系、社区等图谱信息 |

**ReasoningCoherence 的规则评分设计**（`deep_search_metrics.py:59-68`）：

```python
structure_score = 0.6                              # 有思考过程就给 0.6 基础分
if has_queries:
    structure_score += 0.1 * min(3, len(search_queries))  # 每个查询 +0.1，最多 +0.3
if has_structure and len(paragraphs) > 3:
    structure_score += 0.1                          # 有明确段落结构 +0.1
# 最终取 max(规则分, LLM分)
```

**设计思想**：规则分给"下限保障"（有结构就不会太低），LLM分给"上限突破"（逻辑真的好就应该更高），取两者最大值。

---

### 1.6 CompositeEvaluator：评估系统的总指挥

`CompositeGraphRAGEvaluator` 是整个评估系统的入口，理解它就理解了评估的全流程。

#### 核心流程图

```
CompositeGraphRAGEvaluator.evaluate_with_golden_answers(agent_name, questions, golden_answers)
│
├── 1. 加载 Agent 实例
│
├── 2. 逐题处理循环
│   ├── 计时开始
│   ├── agent.ask(question) → 获取原始回答
│   ├── clean_thinking_process() → 移除 <think> 标签（Deep Agent）
│   ├── clean_references() → 移除 #### 引用数据 部分
│   ├── extract_references_from_answer() → 提取实体/关系/chunk ID
│   ├── 创建 AnswerEvaluationSample（含 question, golden_answer, system_answer）
│   ├── 创建 RetrievalEvaluationSample（含 question, retrieval_time, entities）
│   └── 计时结束
│
├── 3. 执行 AnswerEvaluator.evaluate(answer_data)
│   ├── 遍历配置的答案指标（em, f1, response_coherence, ...）
│   ├── 对每个指标调用 metric.calculate_metric(data)
│   ├── 汇总每个样本的分数 → 求平均
│   └── 输出 {"em": 0.57, "f1": 0.75, ...}
│
├── 4. 执行 GraphRAGRetrievalEvaluator.evaluate(retrieval_data)
│   ├── 遍历配置的检索指标
│   ├── 部分指标需要查 Neo4j（entity_coverage, community_relevance）
│   └── 输出 {"retrieval_precision": 0.37, ...}
│
├── 5. 合并结果 results = {**answer_results, **retrieval_results}
│
└── 6. 保存到 JSON 文件
```

#### 多 Agent 对比功能

```python
# compare_agents_with_golden_answers() 的逻辑：
results = {}
for agent_name, agent in self.agents.items():  # naive, graph, hybrid, deep, fusion
    if agent:
        agent_results = self.evaluate_with_golden_answers(agent_name, questions, golden_answers)
        results[agent_name] = agent_results

# 输出对比表格（format_comparison_table）：
# | 指标              | naive  | graph  | hybrid | deep   | fusion |
# |-------------------|--------|--------|--------|--------|--------|
# | **答案质量指标**  |        |        |        |        |        |
# | em                | 0.2667 | 0.5667 | 0.4333 | 0.6000 | 0.7000 |
# | f1                | 0.4300 | 0.5667 | 0.5091 | 0.7500 | 0.8200 |
# | **检索性能指标**  |        |        |        |        |        |
# | retrieval_latency | 3.2    | 8.1    | 5.4    | 28.7   | 62.3   |
```

---

### 1.7 每种 Agent 的指标配置差异

不同 Agent 使用不同的指标集合，这是因为它们的能力维度不同：

```
NaiveAgent 的指标（最少）:
  答案：em, f1, response_coherence, factual_consistency, answer_comprehensiveness, llm_evaluation
  检索：retrieval_precision, retrieval_utilization, retrieval_latency, chunk_utilization
  → 没有图谱指标（因为它不用图谱）
  → 用 chunk_utilization 替代（因为它只检索文本 chunk）

GraphAgent / HybridAgent 的指标:
  答案：同上 6 个
  检索：+ entity_coverage, graph_coverage, relationship_utilization,
        community_relevance, subgraph_quality
  → 增加了 5 个图谱质量指标

FusionAgent 的指标:
  答案 + 检索：同 Graph/Hybrid
  推理：+ reasoning_coherence, reasoning_depth, iterative_improvement
  → 增加了 3 个推理指标

DeepAgent 的指标（最多）:
  答案 + 检索 + 推理：同 Fusion
  深度：+ knowledge_graph_utilization
  → 增加了 1 个专属指标
```

**面试话术**："我们为不同 Agent 配置了差异化的指标集——NaiveRAG 只评估文本检索维度，不评估图谱指标，因为它根本不使用知识图谱。DeepAgent 则有最完整的指标集，包括推理深度和图谱利用率等专属指标。这种差异化配置避免了用错指标导致的不公平对比。"

> 👉 **[QA指路] 为什么 NaiveAgent 用 `chunk_utilization` 而不是 `entity_coverage`？详见 `day10_note_QA.md` 的 Q7**

---

### 1.8 数据预处理：reference_extractor 的鲁棒设计

评估的前提是能从 Agent 回答中提取出引用数据。但 LLM 生成的引用格式千变万化，`reference_extractor.py` 必须处理各种情况：

```
LLM 可能生成的引用格式（实际遇到的）：

格式 1：#### 引用数据 {"Entities": [1, 2, 3], "Relationships": [4, 5]}
格式 2：引用数据: {'entities': ['uuid-123', 'uuid-456']}
格式 3：<引用数据> {"data": {"Entity": [1,2]}} </引用数据>
格式 4：引用: {Entities = [1, 2, 3]}   ← 这不是合法 JSON！
格式 5：参考: {"Entities": "1, 2, 3"}  ← 逗号分隔字符串

提取器的多层容错策略：
  1. 正则匹配 7 种引用标记格式
  2. 直接 JSON 解析
  3. 单引号→双引号修复后解析
  4. 提取嵌套 data 字段
  5. 暴力清理（去非 ASCII + 补双引号）
  6. 全部失败 → 退回到正则直接提取数字
```

**面试话术**："引用提取器采用了 6 层容错策略，从精确 JSON 解析逐步退化到正则模式匹配。这是因为 LLM 生成格式不可控——我们在测试中遇到过单引号、缺少引号键名、嵌套结构等各种情况。宁可多写几层 fallback，也不能因为格式解析失败导致整个评估流程中断。"

> 👉 **[QA指路] `reference_extractor` 处理多种格式的设计是否过度工程？详见 `day10_note_QA.md` 的 Q4**

---

### 1.9 评估系统的 STAR 故事

**S（5 秒）**：系统有 5 种 Agent，但没有统一的评估方式来量化"到底哪个更好"。

**T（5 秒）**：设计一套多维度、可扩展的评估框架，支持公平的 Agent 对标比较。

**A（90 秒）**：

> 我设计了一个 4 维度、18 指标的评估框架。
>
> **架构上**，采用策略模式——`BaseMetric` 定义统一的 `calculate_metric()` 接口，18 个具体指标各自实现。`BaseEvaluator` 通过 Python 的 `__subclasses__()` 方法自动发现所有已注册的指标类，实现了**开放-封闭原则**：添加新指标只需继承基类、设置 `metric_name`，不用改任何已有代码。
>
> **评分上**，每个指标内置了三层回退机制：规则评分做快速筛选，LLM 评分做语义兜底，默认分数做最终容错。比如 F1 指标，先用 jieba 分词做传统 F1 计算，如果规则分偏低再调用 LLM 评估内容等价性，取两者较高值。这样既控制了 API 成本，又保证了准确性。
>
> **编排上**，`CompositeEvaluator` 作为总指挥，同时管理答案评估器和检索评估器，支持单 Agent 评估和多 Agent 对标。每种 Agent 还配置了差异化的指标集——NaiveRAG 不评估图谱指标，Deep Agent 有专属的推理深度指标。

**R（10 秒）**：框架覆盖了答案质量（6 指标）、检索性能（4 指标）、图谱质量（5 指标）、深度推理（4 指标）四个维度。实测数据显示 FusionAgent 综合评分 0.93，DeepAgent F1 最高 0.75，NaiveRAG 仅 0.43。

---

## 二、三个关键代码 Walk-Through

### 为什么准备代码 Walk-Through？

```
面试官说"带我看看代码"时，他在考察：
  1. 你是不是真的写过/深入读过代码（而非只会说概念）
  2. 你能不能用代码解释设计决策
  3. 你的代码阅读能力和表达能力

应对策略：
  不要逐行读代码（太慢，面试官会不耐烦）
  要像导游一样——"这个文件做什么"→"核心方法是哪个"→"关键的几行代码"→"这里的设计决策是什么"

  总时间控制在 3-5 分钟
```

---

### Walk-Through 1：`agents/base.py` — Agent 基类架构

**导游式讲解（2 分钟版）**：

> "这个文件是所有 Agent 的基类，大约 300 行。我重点讲三个关键部分。"

**第一站：`__init__` 方法（第 21-74 行）**

> "构造函数做了四件事：
>
> 第一，初始化双 LLM 实例——`self.llm` 用于普通推理，`self.stream_llm` 用于流式输出。分开的原因是流式 LLM 需要不同的配置参数。
>
> 第二，初始化双层缓存——会话缓存用 `ContextAwareCacheKeyStrategy`，键中包含 `thread_id`，保证多轮对话的上下文隔离。全局缓存用 `GlobalCacheKeyStrategy`，键只基于 query 文本，跨会话复用。两层的容量也不同：会话缓存 200 内存 + 2000 磁盘，全局缓存 500 内存 + 5000 磁盘。
>
> 第三，调用 `_setup_tools()` —— 这是抽象方法，每种 Agent 返回不同的工具集。比如 NaiveAgent 只返回向量搜索工具，GraphAgent 返回本地搜索和全局搜索两个工具。
>
> 第四，调用 `_setup_graph()` —— 构建 LangGraph 状态图。"

**第二站：`_setup_graph` 方法（第 81-113 行）**

> "这是 LangGraph 工作流的核心。我用伪代码讲结构：
>
> ```
> StateGraph(AgentState)
>   ├── 节点 "agent" → _agent_node（LLM 决策：要不要调工具）
>   ├── 节点 "retrieve" → ToolNode(self.tools)（执行工具调用）
>   ├── 节点 "generate" → _generate_node（生成最终答案）
>   │
>   ├── START → agent
>   ├── agent →[tools_condition]→ retrieve 或 END
>   ├── retrieve →[子类自定义]→ generate 或其他
>   └── generate → END
> ```
>
> 关键设计：`_add_retrieval_edges(workflow)` 这个钩子方法让子类可以自定义检索后的行为。比如 GraphAgent 在 retrieve 和 generate 之间插入了 `_grade_documents` 节点做质量评估。
>
> 编译时传入 `MemorySaver` 做 checkpointer，这就是 LangGraph 的对话记忆机制。"

**第三站：双 LLM 的设计决策**

> "为什么要分两个 LLM 实例？因为流式和非流式有不同的超参需求。非流式用于工具调用决策——需要完整的 JSON 输出，temperature 要低。流式用于最终回答生成——需要 token 级输出，可以允许更多创造性。虽然当前是伪流式，但这个双实例架构为未来真流式升级预留了接口。"

---

### Walk-Through 2：`evaluation/evaluators/composite_evaluator.py` — 评估编排

**导游式讲解（2 分钟版）**：

> "这个文件是评估系统的总指挥，大约 700 行。核心是 `evaluate_with_golden_answers()` 方法。"

**第一站：`__init__` 中的指标分流（第 40-98 行）**

> "构造函数做了一件巧妙的事——把传入的指标列表自动分流成答案指标和检索指标两组。分流规则很简单：以 `retrieval_` 开头的或者图谱相关的（entity_coverage、graph_coverage 等）归检索评估器，其余归答案评估器。
>
> 每组用各自的 config 实例化独立的 evaluator。这样答案评估和检索评估完全解耦，可以独立运行。"

**第二站：`evaluate_with_golden_answers`（第 125-262 行）**

> "核心评估流程。我讲三个关键步骤：
>
> 第一步，逐题调用 `agent.ask(question)` 获取回答，同时计时。注意这里有 try-catch 包裹——如果某个问题 Agent 报错了，不会中断整个评估，而是记录错误信息继续下一题。
> 
> 👉 **[QA指路] 评估时如果 Agent 某题报错了，你怎么处理？这会影响平均分吗？详见 `day10_note_QA.md` 的 Q8**
>
> 第二步，数据预处理。对 Deep Agent 的回答调 `clean_thinking_process()` 移除 `<think>` 标签；对所有回答调 `clean_references()` 移除引用数据部分；再用 `extract_references_from_answer()` 提取实体和关系 ID。这个三步清理保证了评估时比较的是'干净的答案内容'，而不是带着格式噪音的原始输出。
>
> 第三步，分别执行两个评估器，合并结果。`{**answer_results, **retrieval_results}` 这个字典合并非常 Pythonic，同时把两组结果扁平化成一个字典方便保存和展示。"

**第三站：`compare_agents_with_golden_answers`（第 264-297 行）**

> "多 Agent 对比就是简单地循环调用 `evaluate_with_golden_answers`，但输出的 `format_comparison_table` 设计很巧——它把指标按类别分组（答案质量、LLM 评估、检索性能），每组有一个加粗的标题行，生成的 Markdown 表格一目了然。"

---

### Walk-Through 3：`evaluation/metrics/answer_metrics.py` — F1 评分实现

**导游式讲解（2 分钟版）**：

> "这个文件实现了两个最基础的评估指标：ExactMatch 和 F1Score。我重点讲 F1Score，因为它展示了三层回退机制的完整实现。"

**第一站：中文分词 + 停用词过滤（第 202-211 行）**

> ```python
> import jieba
> pred_tokens = list(jieba.cut(pred_text))
> golden_tokens = list(jieba.cut(golden_text))
>
> stopwords = {'的', '了', '和', '在', '是', '为', '以', '与', '或', '且'}
> pred_tokens = [t for t in pred_tokens if len(t) > 1 and t not in stopwords]
> ```
>
> "传统 F1 是基于 token 匹配的，对英文直接按空格拆就行。但中文没有天然分隔符，所以用 jieba 分词。停用词表只有 10 个——这是故意的，因为领域文本中很多常用词其实有意义（比如'学生'、'奖学金'不能当停用词），所以只过滤最无信息量的虚词。"

**第二站：标准 F1 计算（第 226-233 行）**

> ```python
> common_tokens = set(pred_tokens) & set(golden_tokens)
> precision = len(common_tokens) / len(pred_tokens)
> recall = len(common_tokens) / len(golden_tokens)
> rule_f1 = 2 * precision * recall / (precision + recall)
> ```
>
> "这就是经典的 F1 公式。注意用的是 `set` 交集——同一个词出现多次只算一次。这意味着 F1 衡量的是'词汇覆盖度'而非'文本重复度'。"
>
> 👉 **[QA指路] 为什么 F1 用 set 交集而不是 multiset（Counter）？这有什么影响？详见 `day10_note_QA.md` 的 Q3**

**第三站：LLM 回退取较高值（第 244-272 行）**

> ```python
> if self.llm:
>     llm_f1 = self.get_llm_fallback_score(prompt, default_score=0.5)
>     if llm_f1 > rule_f1:
>         f1 = llm_f1   # LLM 分数更高 → 采用 LLM
>     else:
>         f1 = rule_f1   # 规则分数更高 → 保留规则
> ```
>
> "这里的 `max` 策略很关键——如果规则算出 F1 = 0.3（因为表达方式完全不同），但 LLM 判定内容等价给了 0.85，取 0.85。反过来，如果 LLM 抽风给了 0.2，但规则算出 0.7，取 0.7。这保证了每种评分方式都只能'帮忙'不能'捣乱'。"

---

## 三、终极模拟面试

### 3.1 新增面试题（评估系统专题）

以下是面试中可能被追问的评估系统相关问题，配合 Day 9 的 Q1-Q15 形成完整题库：

---

**Q16："你的评估框架是怎么设计的？有多少指标？"**

> 4 个维度、18 个指标的评估框架。
>
> 答案质量 6 个：ExactMatch、F1、回答连贯性、事实一致性、全面性、LLM 4 维综合评分。
>
> 检索性能 4 个：检索精确率、检索利用率、检索延迟、Chunk 利用率。
>
> 图谱质量 5 个：实体覆盖率、图覆盖率（结构+相关+连通三维）、关系利用率、社区相关性、子图质量。
>
> 深度研究 4 个：推理连贯性、推理深度、迭代改进度、图谱知识利用率。
>
> 架构上采用策略模式，`BaseMetric` 定义统一接口，评估器通过 `__subclasses__()` 自动发现所有注册的指标类。每个指标内置三层回退：规则评分→LLM 回退→默认分数。

---

**Q17："F1 指标为什么用 jieba 分词？如果不用会怎样？"**

> 中文没有天然的词边界。如果不用 jieba，只能按字符拆分——"国家奖学金"会被拆成 5 个单字符 token。这会导致两个问题：
>
> 1. **粒度太细**："国家奖学金"和"国家励志奖学金"按字符的 F1 会很高（共享 5 个字），但它们是不同概念。按词分的话，"国家奖学金"和"国家励志奖学金"是两个不同的 token，能正确反映差异。
>
> 2. **停用词无法过滤**：按字符的话"的"、"了"、"是"都是单字符 token，和有意义的"学"、"生"混在一起，无法区分。用 jieba 分词后可以精确移除停用词。

---

**Q18："三层回退机制中，为什么 F1 取 max 而不是加权平均？"**

> 加权平均会"拉低"好分数。假设规则 F1 = 0.8（词汇匹配很好），LLM 评了 0.3（LLM 抽风了），如果取平均变成 0.55——这不公平，因为规则评分已经很准了。
>
> 取 max 的逻辑是：**每种评分方式只可能低估，不会高估**。规则评分容易因为表达方式不同而低估（"被开除"vs"退学"），LLM 评分偶尔因为 hallucination 而低估。取 max 意味着"只要有一种方式认为你好，就算你好"。
>
> 当然这也有风险——如果 LLM 误判给了高分，会造成虚高。但在实际测试中，LLM 误给高分的概率远低于规则误给低分的概率，所以 max 策略总体更合理。

---

**Q19："CompositeEvaluator 为什么不继承 BaseEvaluator？"**

> 因为职责不同。`BaseEvaluator` 是"执行者"——它管理一组同类指标，对每个指标调 `calculate_metric()`。`CompositeEvaluator` 是"编排者"——它持有两个不同类型的 evaluator（答案+检索），协调它们的执行顺序，合并结果，管理 Agent 实例。
>
> 如果让 CompositeEvaluator 继承 BaseEvaluator，它需要实现 `evaluate(data)` 抽象方法，但它的输入不是单一的 data 对象，而是同时需要 AnswerEvaluationData 和 RetrievalEvaluationData。强行继承反而破坏了接口语义。
>
> 这是**组合优于继承**原则的实际应用——CompositeEvaluator 通过持有两个 evaluator 的引用来协作，而不是通过继承获取能力。

---

**Q20："如果让你改进评估系统，你会怎么做？"**

> 三个方向：
>
> 1. **增加对抗性测试**：当前只评估正常问题。应该加入对抗性样本——比如知识图谱中不存在的问题、需要拒答的问题（"如何作弊"）。评估系统能不能识别 Agent 该拒答时拒答了。
>
> 2. **自动化回归测试**：当前每次改代码后要手动跑评估。应该把评估集成到 CI/CD 流程中——每次 commit 自动跑一轮精简版评估，如果某个指标下降超过阈值自动阻断。
>
> 3. **指标冲突检测**：18 个指标可能互相矛盾——全面性高的答案往往直接性低（因为说太多了），检索延迟低的 Agent 往往精确率也低（因为搜得少）。应该增加一个指标间的相关性分析，帮助使用者理解 trade-off 而不是盲目追求每个指标都高。

---

### 3.2 三个关键代码定位速查表

面试中被问到"代码在哪？"时，秒定位：

| 技术点 | 文件路径 | 关键行号/方法 |
|--------|---------|--------------|
| LangGraph 状态图构建 | `agents/base.py` | `_setup_graph()` |
| 双层缓存初始化 | `agents/base.py` | `__init__()` 第 44-66 行 |
| 本地搜索邻居扩展 | `search/local_search.py` | `db_query()` + Cypher 查询 |
| 全局搜索 Map-Reduce | `search/global_search.py` | `search()` 方法 |
| 实体消歧三步管道 | `graph/processing/` | disambiguation 相关文件 |
| Plan-Execute-Report | `agents/multi_agent/` | planner/ executor/ reporter/ |
| CompositeEvaluator | `evaluation/evaluators/composite_evaluator.py` | `evaluate_with_golden_answers()` |
| F1 三层回退 | `evaluation/metrics/answer_metrics.py` | `F1Score.calculate_metric()` |
| 指标自动发现 | `evaluation/core/base_evaluator.py` | `_collect_metrics()` |
| Agent 差异化指标 | `evaluation/evaluator_config/agent_evaluation_config.py` | `AGENT_EVALUATION_CONFIG` |
| 引用数据提取 | `evaluation/preprocessing/reference_extractor.py` | `extract_references_from_answer()` |
| `messages[-3]` 硬编码 | `agents/base.py` | `_generate_node()` |

---

### 3.3 完整 30 分钟模拟面试脚本

按照真实面试节奏设计，逐分钟对照练习：

```
0:00 - 2:00  面试官："请介绍一下你做的项目。"
             → 使用 Day 9 的 2 分钟话术

2:00 - 5:00  面试官追问基础概念（预测 3 个方向之一）：
             方向A："GraphRAG 和传统 RAG 区别？" → Q1
             方向B："知识图谱怎么构建的？" → Story 1
             方向C："为什么选 LangGraph？" → Q2

5:00 - 12:00 深挖某个技术点（预测 2 个热门方向）：
             热门1："实体消歧的三步管道详细讲讲" → Story 1 + Q7（消歧阈值）
             热门2："多 Agent 协作怎么实现的？" → Story 3 + 追问 DAG 有环

12:00 - 18:00 切换到另一个技术点：
             "评估系统怎么设计的？" → 新 Story（1.9 节）+ Q16
             "缓存为什么分两层？" → Story 4 + Q4
             "流式输出怎么实现的？" → Q6

18:00 - 25:00 考察工程素养和反思能力：
             "项目有什么不足？" → Story 5（选 3 个最有冲击力的）
             "如果让你改进评估系统？" → Q20
             "如果从零设计会有什么不同？" → Q13

25:00 - 28:00 快速连环问（考反应速度）：
             "F1 为什么用 jieba？" → Q17
             "CompositeEvaluator 为什么不继承 BaseEvaluator？" → Q19
             "Leiden 和 SLLPA 区别？" → Q8

28:00 - 30:00 你反问面试官：
             推荐问题 1："团队目前用的是什么 RAG 方案？在检索增强上有遇到什么挑战吗？"
             推荐问题 2："团队对 Agent 技术栈的选型方向是怎样的？比如 LangGraph vs 自研框架。"
```

---

### 3.4 易错点 & 高频纠正

在模拟过程中，注意以下最常见的表达错误：

| 错误表达 | 正确表达 | 为什么 |
|---------|---------|-------|
| "用了 18 个评估指标" | "设计了 4 维度 18 指标的评估框架" | 强调维度划分，显示体系化思维 |
| "F1 就是精确率和召回率的调和平均" | "F1 基于 jieba 分词后的 token 集合匹配，并结合 LLM 语义回退" | 突出中文特色和三层回退 |
| "CompositeEvaluator 管理评估" | "CompositeEvaluator 编排答案评估器和检索评估器的协同执行" | 精确描述职责 |
| "评估效果挺好" | "FusionAgent 综合评分 0.93，F1 从 NaiveRAG 的 0.43 提升到 DeepResearch 的 0.75" | 用数据替代感觉 |
| "有三层回退" | "规则评分做快速筛选、LLM 评分做语义兜底、默认分数做容错保底" | 说清每层的作用 |

> 👉 **[QA指路] 面试终极杀手锏：从用户提问到评估打分，画出完整的数据流！详见 `day10_note_QA.md` 的 Q10（⭐⭐⭐强烈推荐）**

---

## 四、10 天学习的知识图谱总览

```
Day 1-2: 项目总览与环境搭建
  └── 理解三层架构：图谱构建 → 搜索检索 → Agent 推理

Day 3: Agent 基类与 LangGraph
  └── BaseAgent、StateGraph、ToolNode、MemorySaver、双 LLM

Day 4: 搜索策略
  └── LocalSearch（邻居扩展）、GlobalSearch（Map-Reduce）、DualPathSearcher 缺陷

Day 5: 图谱构建与实体质量
  └── LLM 提取、KNN+WCC 聚类、三阶段消歧、实体对齐、一致性校验

Day 6: 缓存与服务层
  └── 三级缓存、会话隔离、请求链路 6 层、伪流式 SSE

Day 7: 社区检测与全局搜索
  └── Leiden（模块度优化）、SLLPA（标签传播）、社区摘要生成

Day 8: 多 Agent 与 FusionAgent
  └── Plan-Execute-Report、任务 DAG、Map-Reduce 长文档、ConsistencyChecker

Day 9: 面试 STAR 话术
  └── 2 分钟项目介绍、5 个 STAR 故事、15 个高频追问

Day 10: 评估系统 + 代码 Walk-Through + 终极查漏
  └── 4 维度 18 指标、三层回退、CompositeEvaluator、策略模式
  └── 3 个关键文件 Walk-Through
  └── 新增 5 个面试追问（Q16-Q20）
```

---

## 五、今日打卡清单

- [ ] 能画出评估系统的类继承体系图（BaseMetric → 18 个子类）
- [ ] 能解释三层回退机制（规则 → LLM → 默认）
- [ ] 能说出 4 个评估维度各包含哪些指标
- [ ] 能讲清 Precision vs Utilization 的区别
- [ ] 能完成 3 个代码 Walk-Through（每个 2-3 分钟）
- [ ] 能回答 Q16-Q20 中的任意 3 个（不看笔记）
- [ ] 完成一次 30 分钟不间断模拟面试
- [ ] 能不看笔记用 2 分钟流利介绍项目（终极测试）

---

## 六、写在最后：10 天学习心法总结

```
你现在拥有的：
  ✓ 一个可以讲 2 小时不重复的项目理解
  ✓ 5 个 STAR 故事 + 20 个追问答案
  ✓ 10 个独立发现的架构问题和改进方案
  ✓ 3 个可以逐行 Walk-Through 的关键代码文件
  ✓ 全套评估体系的深度理解

面试的本质不是"背答案"，而是"展示思维方式"：
  技术选型 → 为什么选这个而不是那个？（trade-off 思维）
  架构设计 → 为什么这样分层？（抽象思维）
  质量保证 → 怎么知道做得好不好？（量化思维）
  持续改进 → 有什么不足？怎么改？（反思思维）

这四种思维方式，比任何具体的技术点都重要。
```

---

*学习日期：2026-03-03 | 项目路径：`/home/wkt/project/graph-rag-agent`*
