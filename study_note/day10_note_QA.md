# Day 10 QA 深度追问 — 评估系统 + 代码细节 + 综合串联

> 配合 `day10_note.md` 使用。本文档包含面试官可能的深层追问和"反套路"问题。
>
> 每个问题标注了难度：⭐ 基础 | ⭐⭐ 进阶 | ⭐⭐⭐ 高阶

---

## Q1：你的评估指标有 18 个，这些指标之间有没有冲突？你怎么做最终判断？ ⭐⭐⭐

**追问背景**：面试官考察你是否理解"指标越多不代表越好"，以及多目标优化中的 trade-off 思维。

> 确实存在冲突，而且这是**有意为之**的。举两个典型冲突：
>
> **冲突 1：全面性 vs 直接性。** FusionAgent 生成 5000 字报告，全面性（comprehensiveness）得分很高，但直接性（directness）偏低——面试官问"旷课退不退学"，你写了 5 页，直接性当然低。NaiveRAG 相反——简短直接，但不全面。
>
> **冲突 2：检索延迟 vs 检索精确率。** DeepAgent 多轮迭代搜索，精确率高但延迟 30 秒。NaiveRAG 一次向量检索，延迟 3 秒但精确率低。
>
> **怎么做最终判断？** 不追求所有指标都高，而是根据使用场景选择优先级：
> - 简单事实查询 → 优先直接性和延迟 → 用 NaiveRAG
> - 复杂分析 → 优先全面性和推理深度 → 用 FusionAgent
> - 日常问答 → 平衡各项 → 用 HybridAgent
>
> LLM 综合评分（llm_evaluation）本身就是加权平均（全面性 30%、相关性 25%、增强理解 25%、直接性 20%），权重的设定反映了我们认为"全面性比直接性更重要"的产品判断。如果部署场景是客服秒回，应该提高直接性权重；如果是研究报告，应该提高全面性权重。

---

## Q2：BaseEvaluator 的 `_collect_metrics()` 用了 `__subclasses__()`，这个方法有什么陷阱？ ⭐⭐⭐

**追问背景**：面试官考察 Python 元编程知识和对"魔法方法"的理解深度。

> `__subclasses__()` 只返回**直接子类**，不返回孙子类。当前代码用了递归版本 `find_descendants()` 解决了这个问题：
>
> ```python
> def find_descendants(base_class, subclasses=None):
>     if subclasses is None:
>         subclasses = set()
>     direct_subclasses = base_class.__subclasses__()
>     for subclass in direct_subclasses:
>         if subclass not in subclasses:
>             subclasses.add(subclass)
>             find_descendants(subclass, subclasses)
>     return subclasses
> ```
>
> 但还有两个潜在陷阱：
>
> **陷阱 1：模块未导入则不可见。** `__subclasses__()` 只能发现已经被 Python 解释器加载的类。如果某个指标文件没有被 import，它的子类就不会出现。当前项目在 `evaluation/metrics/__init__.py` 中统一 import 所有指标模块来解决这个问题。
>
> **陷阱 2：热重载风险。** 如果在运行时动态导入新的指标模块（比如插件系统），`_collect_metrics()` 只在 `__init__` 时调用一次，新指标不会被感知。解决方案是懒加载——每次 evaluate 时重新收集，或者提供 `refresh_metrics()` 方法。

---

## Q3：为什么 F1 用 set 交集而不是 multiset（Counter）？这有什么影响？ ⭐⭐

**追问背景**：面试官考察对 F1 计算细节的理解和对 NLP 评估惯例的了解。

> 用 `set` 意味着每个词只算一次，不管它出现了多少次。
>
> **影响举例**：
> - 标准答案："国家奖学金的申请条件是成绩优秀"
> - 系统答案："国家奖学金是国家奖学金制度中最重要的国家奖学金"
> - 用 set：system = {"国家奖学金", "制度", "重要"}, golden = {"国家奖学金", "申请", "条件", "成绩", "优秀"}。交集 = {"国家奖学金"}，F1 = 2×(1/3×1/5)/(1/3+1/5) ≈ 0.17
> - 用 Counter：system 中 "国家奖学金" 出现 3 次，会膨胀 precision。
>
> **用 set 的合理性**：评估的是"信息点覆盖率"而非"词频匹配"。系统答案重复说"国家奖学金"三次不代表它更好，反而可能是废话。SQuAD 等经典 QA benchmark 也用 set-based F1。
>
> **如果要改进**：可以用 TF-IDF 加权的 F1——高 IDF 的词（如"旷课"）匹配了应该加分更多，低 IDF 的词（如"学生"）匹配了加分更少。但这需要额外的语料库统计，增加了工程复杂度。

---

## Q4：reference_extractor 处理 6 种格式的设计是否过度工程？ ⭐⭐

**追问背景**：面试官考察你对"过度工程 vs 防御性编程"的判断力。

> **不是过度工程，而是防御性编程。** 原因有两个：
>
> 1. **LLM 输出不可控**。我们用 Prompt 要求 LLM 输出 JSON 格式的引用数据，但实际测试中发现 DeepSeek 和 GPT-4o 在不同 temperature 下会产生完全不同的格式——有时用双引号，有时用单引号，有时键名不加引号，有时嵌套一层 `data` 字段。这些都是**真实遇到的格式**，不是假想的。
>
> 2. **评估链路的脆弱性**。如果引用提取失败，后续所有检索性能指标（precision、utilization、entity_coverage 等）都会变成 0——不是因为 Agent 检索不好，而是因为我们没能正确提取引用数据。一个提取器的 bug 会导致整个评估结论错误。
>
> **但可以优化**：当前是硬编码 7 种正则模式。更好的做法是：
> - 在 Agent 的 Prompt 中加入**严格的输出格式约束**（few-shot examples + JSON schema）
> - 同时保留 fallback 提取器作为安全网
> - 记录每次 fallback 被触发的日志，用于优化 Prompt

---

## Q5：如果面试官让你现场写一个新的评估指标，你怎么做？ ⭐⭐

**追问背景**：面试官考察你对框架扩展点的理解和编码能力。

> 假设要加一个 **AnswerConciseness**（回答简洁度）指标：
>
> ```python
> # 文件：evaluation/metrics/answer_metrics.py（在已有文件中追加）
>
> class AnswerConciseness(BaseMetric):
>     """回答简洁度评估——过长的回答扣分"""
>
>     metric_name = "answer_conciseness"  # 唯一标识
>
>     def calculate_metric(self, data) -> Tuple[Dict[str, float], List[float]]:
>         scores = []
>         for sample in data.samples:
>             answer_len = len(sample.system_answer)
>             golden_len = len(sample.golden_answer)
>
>             # 规则评分：答案长度是标准答案的 1-2 倍为最优
>             ratio = answer_len / max(golden_len, 1)
>             if 0.8 <= ratio <= 2.0:
>                 score = 1.0
>             elif ratio > 2.0:
>                 score = max(0.2, 1.0 - (ratio - 2.0) * 0.2)  # 超长扣分
>             else:
>                 score = max(0.3, ratio / 0.8)  # 过短扣分
>
>             scores.append(score)
>
>         avg = sum(scores) / len(scores) if scores else 0.0
>         return {"answer_conciseness": avg}, scores
> ```
>
> **只需要这一步**。因为 `_collect_metrics()` 会自动通过 `__subclasses__()` 发现这个新类，然后在 `agent_evaluation_config.py` 的对应 Agent 配置中加入 `"answer_conciseness"` 即可启用。**不需要修改任何评估器代码。**

---

## Q6：LLMGraphRagEvaluator 用 LLM 评估 LLM，这有没有循环偏差？ ⭐⭐⭐

**追问背景**：这是 LLM-as-a-Judge 领域的核心争议，面试官考察你对 AI 评估范式的理解深度。

> 确实存在偏差，这是**LLM-as-a-Judge 范式的固有缺陷**。具体有三种偏差：
>
> 1. **自我偏好偏差**（Self-preference bias）：如果用 GPT-4o 生成答案，又用 GPT-4o 评分，GPT-4o 倾向于给自己风格的答案打高分。
>
> 2. **位置偏差**（Position bias）：在对比评估中，放在 Prompt 前面的答案更容易获得高分。
>
> 3. **冗长偏差**（Verbosity bias）：LLM 倾向于给更长的答案打更高分，即使长度≠质量。
>
> **项目中的缓解措施**：
> - 三层回退机制不完全依赖 LLM——规则评分作为"锚点"，LLM 只在规则分不确定时做补充
> - LLM 评分 Prompt 中明确了评分标准（0.8-1.0 高分、0.4-0.7 中分、0.0-0.3 低分），减少了随意打分
> - 最终取 `max(规则分, LLM分)` 而非纯用 LLM 分
>
> **如果要进一步改进**：
> - 用不同模型做交叉评估（GPT-4o 生成 + Claude 评分）
> - 引入人工评估做抽样校准，计算 LLM 评分和人工评分的相关性
> - 采用 pairwise comparison（两个答案对比）替代 pointwise scoring（单个答案打分），前者更稳定

---

## Q7：为什么 NaiveAgent 用 chunk_utilization 而不是 entity_coverage？ ⭐⭐

**追问背景**：面试官考察你对不同 Agent 检索方式差异的理解。

> NaiveAgent 不使用知识图谱，它的检索结果是**文本 chunk**（文档片段），不是实体。所以评估它的检索性能时，应该看"检索到的 chunk 有多少内容被答案使用了"（chunk_utilization），而不是"检索到的实体和问题匹配度"（entity_coverage）。
>
> 如果强行用 entity_coverage 评估 NaiveAgent，结果一定是 0——因为 NaiveAgent 的回答中根本没有实体引用数据，全是纯文本。这就像用"图谱密度"评估一个不使用图谱的系统，指标本身不适用。
>
> 反过来，GraphAgent 不用 chunk_utilization，因为它的检索结果是实体和关系，不是 chunk。**差异化指标配置确保了"用对的尺子量对的东西"。**

---

## Q8：评估时如果 Agent 某题报错了，你怎么处理？这会影响平均分吗？ ⭐⭐

**追问背景**：面试官考察你对评估鲁棒性和边界情况的思考。

> 看 `composite_evaluator.py:171-226` 的 try-catch 逻辑：
>
> ```python
> try:
>     answer = agent.ask(question)
> except Exception as e:
>     error_message = f"获取回答时出错: {str(e)}"
>     answer_sample.update_system_answer(error_message, agent_name)
> ```
>
> Agent 报错时，system_answer 会被设置为错误信息字符串。这个字符串会被正常传入各个指标计算——EM 和 F1 会给出极低分（因为错误信息和标准答案完全不匹配），LLM 指标也会给低分。
>
> **这确实会拉低平均分**，但这是合理的——Agent 报错本身就是质量问题的表现。如果跳过报错的样本只算成功的，会虚高平均分，掩盖系统的稳定性缺陷。
>
> **改进方向**：可以在结果中增加一个 `error_rate` 指标，单独统计报错比例，让使用者区分"回答质量低"和"系统不稳定"两种不同的问题。

---

## Q9：如果面试官问"你的评估数据集有多大？怎么生成标准答案？" ⭐⭐

**追问背景**：面试官考察你对评估方法论的理解——小数据集的评估是否有统计意义。

> 当前测试集是 `test/questions.json` 和 `test/answer.json`，规模较小（约 10-20 个问题），主要用于开发阶段的快速验证。
>
> **标准答案的生成方式**：由人工根据源文档（学生管理规章制度）编写，确保每个答案都有明确的文档依据。这是"银标准"（silver standard）——不是完美的，但足够可靠。
>
> **统计意义的问题**：20 个样本的评估结果确实有较大方差。如果要做严格的对比实验，应该：
> 1. 扩大测试集到 100+ 个问题，覆盖简单/中等/复杂三个难度
> 2. 用 bootstrap 方法计算置信区间——"F1 = 0.75 ± 0.08 (95% CI)"比"F1 = 0.75"更有说服力
> 3. 做统计显著性检验（paired t-test）——"GraphAgent 显著优于 NaiveRAG (p < 0.05)"
>
> 但对于一个学习复现项目来说，当前规模足够展示评估框架的设计能力和指标的区分度。

---

## Q10：综合串联题 — 从用户提问到评估打分，画出完整的数据流 ⭐⭐⭐

**追问背景**：面试官考察你对整个系统端到端的理解深度。

> ```
> 用户提问："旷课 30 学时会被退学吗？"
> │
> ├── 1. [前端层] Streamlit → POST /chat → FastAPI
> │
> ├── 2. [服务层] AgentService 获取会话锁 → AgentManager 懒加载 GraphAgent
> │
> ├── 3. [缓存层]
> │   ├── 检查全局缓存（GlobalCacheKeyStrategy: key = hash("旷课30学时会被退学吗")）
> │   ├── 检查快速缓存（关键词匹配）
> │   ├── 检查会话缓存（ContextAwareCacheKeyStrategy: key = hash(query + thread_id)）
> │   └── 全部 miss → 进入 Agent 执行
> │
> ├── 4. [Agent 层] GraphAgent.ask(question)
> │   ├── LangGraph: START → agent 节点（LLM 决定调用 LocalSearch 工具）
> │   ├── → retrieve 节点（执行 LocalSearch）
> │   │   ├── query → Embedding → Neo4j 向量索引 → Top-K 实体
> │   │   ├── 以实体为种子 → 1-2 跳邻居扩展 → 子图
> │   │   └── 子图序列化为上下文文本
> │   ├── → _grade_documents 节点（检索质量评估）
> │   │   └── 关键词匹配率 > 阈值 → 进入 generate
> │   ├── → generate 节点（LLM 生成最终答案 + 引用数据）
> │   └── 返回: "根据学校规定...旷课累计达到30学时...将被退学处理。
> │            #### 引用数据 {"Entities": [42, 87], "Relationships": [15]}"
> │
> ├── 5. [缓存回写] 将结果写入全局缓存 + 会话缓存
> │
> ├── 6. [返回前端] SSE 推送 / JSON 返回
> │
> └── 7. [评估层] CompositeEvaluator（离线运行）
>     ├── 预处理：
>     │   ├── clean_thinking_process() → 无 <think> 标签，跳过
>     │   ├── clean_references() → 移除 "#### 引用数据" 部分
>     │   └── extract_references() → entities=[42,87], relationships=[15]
>     │
>     ├── AnswerEvaluator：
>     │   ├── EM: "根据学校规定..." vs golden → 规则不完全匹配 → LLM 回退 → 0.85
>     │   ├── F1: jieba 分词 → {"旷课","学时","退学","处理"} ∩ golden → 0.72
>     │   ├── ResponseCoherence: LLM 评估逻辑清晰 → 0.90
>     │   ├── FactualConsistency: LLM 评估无矛盾 → 0.88
>     │   ├── ComprehensiveAnswer: LLM 评估覆盖完整 → 0.85
>     │   └── LLMGraphRagEvaluator: 4 维加权 → 0.87
>     │
>     ├── GraphRAGRetrievalEvaluator：
>     │   ├── RetrievalPrecision: 2/2 实体被引用 → 0.95
>     │   ├── RetrievalLatency: 8.2 秒
>     │   ├── EntityCoverage: 实体匹配问题关键词 → 0.80
>     │   ├── GraphCoverage: 结构 0.7 + 相关性 0.8 + 连通性 0.9 → 0.80
>     │   └── ...
>     │
>     └── 输出: {"em": 0.85, "f1": 0.72, "retrieval_latency": 8.2, ...}
> ```
>
> 这张图把 Day 3（Agent）、Day 4（搜索）、Day 6（缓存/服务）、Day 10（评估）四天的知识串成了一条完整链路。面试中如果能画出这张图，面试官会非常认可你对系统全貌的掌握程度。

---

## Q11：F1Score 评估指标的具体计算流水线（分词、去停用词、Precision/Recall/F1、规则+LLM回退）是怎么运作的？ ⭐⭐⭐

**追问背景**：面试官考察你对 NLP 经典指标底层实现的掌握程度，以及如何通过大模型解决传统指标的局限性。

> 这是一个经典的 **NLP 文本重叠度打分流水线**，主要分为 4 个关键步骤：
>
> 1. **jieba 中文分词**
> 英文天然有空格隔开单词，F1 计算很简单。但中文是一连串的汉字，如果不做分词而是**按单字切分**，会引发严重问题。例如：标准答案是“国家奖学金”，系统回答是“国家励志奖学金”。如果按字算，有 5 个字重合，F1 会非常高，但语义不一致。通过 `jieba` 进行分词，能把“国家奖学金”当成一个整体 token，精准比对词汇级而不是字符级的含义。
>
> 2. **去停用词**
> 中文里有大量“的、了、和、是”等虚词。如果连这些词都参与计算，只要两句话长一点，都会命中一堆“的”，导致两个毫无意义的句子也能算出很高的 F1 分数（即“虚高”）。项目中定义了一个精简的停用词表，将这些无实际意义的词从分词结果中剔除，**只保留有实际信息量的“实体词/动词/核心名词”**。
>
> 3. **计算 Precision / Recall / F1（传统规则计算）**
> 经过清洗后，两个答案都变成了“有效词汇池（Set）”。系统接下来会计算：
> - **精准率（Precision）** = 共同重合的词汇数 / **系统答案**词汇数。衡量的是“答出的内容有多大比例是废话”。
> - **召回率（Recall）** = 共同重合的词汇数 / **标准答案**词汇数。衡量的是“该答出来的信息有没有遗漏”。
> - **F1 分数** = `2 * P * R / (P + R)`。它是两者的调和平均数。
> *注意：项目中这步转成了 `set()` 交集，这意味着一个词即使出现多次也只算 1 次，测量的是“信息覆盖度”，而不是“频次”。*
>
> 4. **规则 + LLM 回退（三层回退的特色设计）**
> 传统 F1 只能做**字面词汇匹配**。如果标准答案是“开除学籍”，系统回答是“退学处理”。在规则中，由于词汇完全不同，F1 等于 0。这显然不合理，因为它们**语义等价**。算完规则 F1 后，系统会把这两句话再扔给 LLM 评估语义相似度。最后系统会取 **`max(规则 F1分数, LLM 评估的 F1分数)`**。如果 LLM 发现语义等价给了 0.9，即使规则结果为 0，最终 F1 也是 0.9。这种 **字面（规则）+ 语义（大模型）** 的双重保障，既严谨又能防止被误判。

---

*学习日期：2026-03-03 | 项目路径：`/home/wkt/project/graph-rag-agent`*
