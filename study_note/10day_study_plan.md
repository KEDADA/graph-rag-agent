# Graph-RAG-Agent 项目：10天深度掌握 & 实习面试复现指南

> **目标**：10天内深度理解该项目，能够在面试中用 STAR 法则流利叙述你的贡献与技术难点。
>
> **实际学习周期**：2026-02-22 ~ 2026-03-03（10天）

---

## 四阶段总览

| 阶段 | 天数 | 重点 | 状态 |
|------|------|------|------|
| 架构通读 | Day 1-2 | 全局理解 + 环境搭建 + 图构建全流程 | ✅ 已完成 |
| 核心模块深挖 | Day 3-6 | Agent 架构 + 搜索策略 + 实体质量 + 多智能体 | ✅ 已完成 |
| 复现实践 | Day 7-8 | 图构建实战 + Agent 问答实战 | ✅ 已完成 |
| 面试备战 | Day 9-10 | STAR 话术打磨 + 评估系统 + 模拟答题 | ✅ 已完成 |

---

## Day 1（2/22）— 项目全貌与技术背景 ✅

### 学习内容
- 读懂 README，建立项目"鸟瞰图"
- 理解 **GraphRAG vs 传统 RAG** 的本质区别（向量检索 → 知识图谱 → 多跳推理）
- 梳理**三层架构**：图构建层 → 搜索检索层 → Agent 推理层
- 理清**技术栈**选型理由（Neo4j / LangGraph / Streamlit）
- 对比 **5 种 Agent**（NaiveRAG → GraphAgent → HybridAgent → DeepResearchAgent → FusionAgent）

### 学习成果
- [x] 绘制三层架构图，标注每层对应的文件目录
- [x] 能口述 5 种 Agent 的名字、适用场景和技术特点
- [x] 能解释 GraphRAG 相比传统 RAG 的两个核心优势（多跳推理 + 全局概览）
- [x] 能回答技术选型追问（为什么选 Neo4j / LangGraph / Streamlit）

> 📁 笔记文件：[day1_note.md](./day1_note.md)（150行）

---

## Day 2（2/22-2/23）— 环境配置 + 图构建全流程 ✅

### 学习内容
- 理解**三层配置体系**（`.env` > `settings.py` > 默认值）
- 深度解析**图构建入口** `KnowledgeGraphProcessor.process_all()` 的三步流程：
  - Step 1：`KnowledgeGraphBuilder` — 文档摄取 → 分块 → LLM 实体提取 → Neo4j 存储
  - Step 2：`IndexCommunityBuilder` — 实体向量索引 + KNN/WCC 相似实体检测 + LLM 合并 + 消歧对齐 + 社区检测
  - Step 3：`ChunkIndexBuilder` — 文本块向量索引
- 深入学习文本分块器 `ChineseTextChunker`（HanLP 分词 + 滑动窗口）
- 深入学习实体相似检测（GDS KNN + WCC 连通分量）
- 深入学习实体合并（`apoc.refactor.mergeNodes`）
- 深入学习实体消歧与对齐（canonical_id 选举 + Jaccard 冲突检测 + 关系迁移）
- 深入学习社区检测（Leiden 模块度优化 + SLLPA 标签传播 + 多层级社区）

### 学习成果
- [x] 能解释 `文件 → 分块 → 实体提取 → 图存储 → 相似检测 → 合并 → 消歧 → 对齐 → 社区检测` 全流程
- [x] 能画出 Neo4j 中的节点类型和关系结构（Document → Chunk → Entity → Community）
- [x] 理解 `file_registry.json` 在增量更新中的作用
- [x] 能区分 4.3 合并、4.4 消歧、4.4 对齐三步的递进关系
- [x] 能解释 Leiden 多层级社区的设计意义

> 📁 笔记文件：[day2_note.md](./day2_note.md)（1258行）+ [day2_note_QA.md](./day2_note_QA.md)

---

## Day 3（2/25）— Agent 基础架构深挖 ✅

### 学习内容
- **LangGraph 入门**：节点（Node）、边（Edge）、状态（State）三核心概念
- **BaseAgent 抽象基类**解析（模板方法模式）：
  - `_setup_graph()`：三节点状态图（agent → retrieve → generate）
  - 双 LLM 实例：`self.llm`（决策+生成）vs `self.stream_llm`（死代码，未被使用）
  - 对话记忆：`MemorySaver`（纯内存，重启丢失）
  - 安全限制：`recursion_limit = 5`
- **NaiveRagAgent vs GraphAgent** 对比：
  - NaiveRagAgent：1 个工具，直连 generate，167 行
  - GraphAgent：2 个工具，条件路由（`_grade_documents`），含 `reduce` 节点，528 行
- **两层缓存系统**：会话缓存（thread_id 隔离）+ 全局缓存（跨会话复用）+ SentenceTransformer 语义匹配
- **伪流式输出**：LLM 完整生成 → 按句切分 → 攒够 threshold 推送

### 学习成果
- [x] 能画出 BaseAgent 和 GraphAgent 的 LangGraph 状态图
- [x] 能跟踪一个问题在 messages 列表中的完整旅程（4 条消息的含义）
- [x] 能解释两层缓存设计的动机和语义缓存原理
- [x] 能区分 MemorySaver 和 CacheManager 的不同职责
- [x] 发现 `stream_llm` 死代码问题，记入 `project_improvements.md`

> 📁 笔记文件：[day3_note.md](./day3_note.md)（712行）+ [day3_note_QA.md](./day3_note_QA.md)

---

## Day 4（2/26）— 搜索策略深挖 ✅

### 学习内容
- **两层封装架构**（核心设计模式）：核心搜索类（纯逻辑）+ 工具封装类（适配 LangChain 协议）
- **BaseSearchTool** 抽象基类：统一的缓存/监控/Neo4j 连接 + 三个抽象方法
- **本地搜索（LocalSearch）**：
  - 向量检索实体 → 图邻居扩展（1-2 跳）→ 5 类信息抓取（Chunks/Reports/OutsideRels/InsideRels/Entities）
  - 核心 Cypher 查询（retrieval_query）一条语句完成向量检索+图扩展
  - 工具层增强：历史感知检索器（多轮对话支持）+ RAG 链
- **全局搜索（GlobalSearch）**：
  - Map-Reduce 模式：获取社区数据 → 逐社区独立提问 → 汇总合并
  - 工具层增强：关键词预过滤社区 + 批处理优化（5 个/批）
- **混合搜索（HybridSearch）**：双级关键词（low_level + high_level）→ 双路检索 → 合并
- **深度研究（DeepResearch）**：多轮 Think → Search → Reason 迭代 + Chain of Exploration
  - 发现 `DualPathSearcher` 名不副实缺陷（`kg_retriever` 完全未使用）
- **工具注册机制（ToolRegistry）**

### 学习成果
- [x] 能解释两层封装架构的设计动机（独立测试、框架解耦、横切关注点统一）
- [x] 能画出本地搜索的完整流程图（向量召回 → 图扩展 → 上下文组装）
- [x] 能对比 Local vs Global 搜索的 6 个维度差异
- [x] 发现 DualPathSearcher 缺陷，记入 `project_improvements.md`

> 📁 笔记文件：[day4_note.md](./day4_note.md)（1412行）+ [day4_note_QA.md](./day4_note_QA.md)

---

## Day 5（2/28）— 实体质量与社区检测 ✅

### 学习内容
- **实体消歧三步管道**（漏斗式设计）：
  - 阶段 1：字符串召回（Levenshtein 编辑距离，阈值 0.6）
  - 阶段 2：向量重排（0.4×文本 + 0.6×向量组合打分）
  - 阶段 3：NIL 检测（阈值 0.75，宁可漏合不错合）
- **WCC 分组消歧**（批量模式）vs **三步管道**（在线/流式模式）的区别与配合
- **实体对齐三步流程**：
  - 按 canonical_id 分组 → Jaccard 冲突检测（阈值 0.5）→ LLM 裁决 / 度数回退 → CALL 子查询隔离合并
  - 发现**强制缝合缺陷**（缺乏拒绝合并退出机制）
  - 发现**激进回退策略**（LLM 失败时默认按度数合并）
- **Leiden 算法深度解析**：模块度优化 + 自适应参数调优（根据内存）+ 层次社区保存
- **SLLPA 算法**：标签传播，速度快但不稳定，支持重叠社区
- **社区摘要生成**：提取社区实体关系 → LLM 生成结构化摘要 → 存储 + Embedding

### 学习成果
- [x] 能解释消歧三步骤的设计哲学（先粗筛后精排，避免 O(n²) 向量计算）
- [x] 能说清 Jaccard 相似度在冲突检测中的应用
- [x] 能解释 CALL 子查询在关系迁移中的必要性
- [x] 发现对齐阶段 2 个架构缺陷，记入 `project_improvements.md`

> 📁 笔记文件：[day5_note.md](./day5_note.md)（1237行）+ [day5_note_QA.md](./day5_note_QA.md)

---

## Day 6（2/28）— FusionAgent 多智能体架构 ✅

### 学习内容
- **Plan-Execute-Report 三阶段架构**总览
- **Plan 阶段**三步流水线：
  - Clarifier（意图澄清）→ TaskDecomposer（任务分解，输出 TaskGraph）→ PlanReviewer（计划审校）
  - 核心数据结构：`PlanSpec`、`TaskNode`（含 `depends_on` 依赖）、`TaskGraph`
  - 拓扑排序（Kahn 算法）+ 循环依赖检测（DFS）
- **Execute 阶段**：
  - `WorkerCoordinator`：串行/并行两种执行模式
  - 并行执行：`ThreadPoolExecutor` + `wait(FIRST_COMPLETED)` 动态调度
  - 三类 Executor：RetrievalExecutor / ResearchExecutor / ReflectionExecutor
  - 依赖检查：failed/completed/pending/missing 四种状态
  - 证据追踪：`EvidenceTracker` 全局唯一 result_id
- **Report 阶段**：
  - OutlineBuilder → SectionWriter（Map-Reduce 并行写作）→ ConsistencyChecker（事实核验）
  - 发现 ConsistencyChecker **仅检测不修复**缺陷

### 学习成果
- [x] 能画出 Plan-Execute-Report 的完整数据流图
- [x] 能解释 PlanSpec 中 `depends_on` 如何实现任务依赖调度
- [x] 能对比 ThreadPoolExecutor vs ProcessPoolExecutor 的选型理由
- [x] 发现 ConsistencyChecker 缺陷，记入 `project_improvements.md`

> 📁 笔记文件：[day6_note.md](./day6_note.md)（1515行）+ [day6_note_QA.md](./day6_note_QA.md)

---

## Day 7（2/28）— 亲手复现：图构建实战 ✅

### 学习内容
- **环境配置**：Python 3.10 虚拟环境 + Docker Compose 启动 Neo4j + `.env` 配置
- **三步构建实战**：
  - Step 1：`KnowledgeGraphBuilder` — 文档读取 → 分块 → LLM 实体提取 → 消歧对齐 → Neo4j 存储
  - Step 2：`IndexCommunityBuilder` — 实体 Embedding → 向量索引 → 社区检测 → 摘要生成
  - Step 3：`ChunkIndexBuilder` — Chunk Embedding → Chunk 索引
- **Neo4j Browser 验证**：节点统计、关系统计、实体邻居可视化、消歧效果验证
- **增量更新**：`file_registry.json` 机制 + 单次模式 + 守护进程模式
- **常见问题排查**：LLM API 连接、Neo4j 连接、OOM、实体提取质量

### 实验记录

| 实验项 | 数值 |
|--------|------|
| 文档数量 | 3 |
| Chunk 数量 | 250 |
| 提取的实体数 | 128（去重前） |
| 去重后实体数 | 105 |
| 关系数量 | 95 |
| 社区数量 | 12（Leiden） |
| 构建耗时 | ~8 分 32 秒 |

### 学习成果
- [x] 成功启动 Neo4j 并在 Browser 中查看图谱
- [x] 执行完整的三步构建流程
- [x] 验证实体消歧效果（找到合并实体组）
- [x] 执行增量更新，观察 `file_registry.json` 变化

> 📁 笔记文件：[day7_note.md](./day7_note.md)（706行）+ [day7_note_QA.md](./day7_note_QA.md)

---

## Day 8（3/1）— 亲手复现：Agent 问答实战 ✅

### 学习内容
- **系统启动与服务架构**：前后端分离（Streamlit 8501 + FastAPI 8000 + Neo4j 7687）
- **请求生命周期**（6 层完整链路）：前端 API 层 → 路由层 → 服务层（会话锁）→ Agent 管理层 → Agent 执行层（三级缓存 + LangGraph）→ 前端展示层（SSE/JSON）
- **AgentManager 设计哲学**：`instance_key = f"{agent_type}:{session_id}"` 实现会话隔离 + 资源复用 + 懒加载
- **5 种 Agent 深度对比**：
  - NaiveRagAgent: 纯向量检索，~3s
  - GraphAgent: 图邻居扩展 + `_grade_documents` 质量守门员 + reduce 路径，~8s
  - HybridAgent: 多策略融合，~5s
  - DeepResearchAgent: 多轮 Think-Search-Reason 迭代 + Thinking Mode + 矛盾检测，~30s
  - FusionAgent: Plan-Execute-Report 多智能体，不使用 LangGraph，委托 MultiAgentFacade，~60s+
- **测试脚本实战**：`search_without_stream.py` + `search_with_stream.py`
- **Debug 模式**：5 标签页（执行轨迹 / 知识图谱 / 源内容 / 图谱管理 / 性能监控）

### 学习成果
- [x] 能描述请求从浏览器到回答的完整 6 层链路
- [x] 能解释 AgentManager 的 instance_key 设计（会话隔离 vs 资源复用）
- [x] 能对比 5 种 Agent 的搜索策略、工具数量、典型耗时和适用场景
- [x] 能解释 FusionAgent 不使用 LangGraph 的底层原因

> 📁 笔记文件：[day8_note.md](./day8_note.md)（1197行）+ [day8_note_QA.md](./day8_note_QA.md)

---

## Day 9（3/2）— 面试 STAR 话术打磨 ✅

### 学习内容
- **面试评估三层模型**：技术广度 → 技术深度 → 工程素养
- **2 分钟项目介绍**（完整版 + 精简版 1 分钟），含关键要诀（具体技术名词 + 数字量化 + 讲"为什么"）
- **5 个 STAR 故事**：
  1. 知识图谱构建与实体质量保证（KNN+WCC → 三阶段消歧 → 对齐）
  2. 搜索策略设计（本地搜索 vs 全局搜索 + GraphAgent 路由）
  3. Plan-Execute-Report 多智能体架构（任务 DAG + 拓扑排序 + Map-Reduce 报告）
  4. 两层缓存与请求生命周期（三级缓存 + 会话锁 + SSE 流式）
  5. 独立发现的架构问题与优化建议（`messages[-3]` 硬编码 / 对齐强制缝合 / DualPathSearcher 名不副实）
- **15 个高频面试追问**（Q1-Q15），覆盖基础概念、架构设计、实体质量、工程实践、延伸追问
- **面试节奏感**：时间分配 / "我不确定"的正确说法 / 抛钩子引导追问 / 总分总结构

### 学习成果
- [x] 能不看笔记用 2 分钟流利介绍项目
- [x] 能展开讲 5 个 STAR 故事
- [x] 能回答 15 个高频面试追问
- [x] 对 `project_improvements.md` 的改进点能脱口而出

> 📁 笔记文件：[day9_note.md](./day9_note.md)（561行）+ [day9_note_QA.md](./day9_note_QA.md)

---

## Day 10（3/3）— 评估系统深度剖析 + 代码 Walk-Through + 终极查漏 ✅

### 学习内容
- **评估系统架构**：
  - `BaseMetric` → 18 个具体指标（策略模式 + `__subclasses__()` 自动发现）
  - `BaseEvaluator` → AnswerEvaluator + RetrievalEvaluator
  - `CompositeGraphRAGEvaluator`（组合优于继承，编排两个评估器）
- **三层评分回退机制**：规则评分 → LLM 回退 → 默认分数
- **4 大评估维度**（18 个指标）：
  - 答案质量（6）：EM / F1 / 连贯性 / 一致性 / 全面性 / LLM 4 维综合评分（全面性 0.3 + 相关性 0.25 + 增强理解 0.25 + 直接性 0.2）
  - 检索性能（4）：精确率 / 利用率 / 延迟 / Chunk 利用率
  - 图谱质量（5）：实体覆盖 / 图覆盖（结构+相关+连通三维）/ 关系利用 / 社区相关 / 子图质量
  - 深度研究（4）：推理连贯 / 推理深度 / 迭代改进 / 图谱利用
- **Agent 差异化指标配置**（NaiveRAG 无图谱指标，DeepAgent 最完整）
- **数据预处理**：`reference_extractor`（6 层容错策略处理 LLM 格式不可控）
- **3 个关键代码 Walk-Through**：
  1. `agents/base.py` — 双 LLM + 双层缓存 + LangGraph 状态图
  2. `evaluation/evaluators/composite_evaluator.py` — 指标分流 + 逐题评估 + 多 Agent 对标
  3. `evaluation/metrics/answer_metrics.py` — jieba 分词 + F1 计算 + LLM 回退取 max
- **新增 5 个面试追问**（Q16-Q20）：评估框架设计 / jieba 分词 / 三层回退 max 策略 / 组合优于继承 / 评估改进方向
- **30 分钟模拟面试脚本**

### 关键评测数据

| Agent | LLM 综合评分 | F1 | 典型耗时 |
|-------|-------------|-----|---------|
| NaiveRAG | — | 0.43 | ~3s |
| DeepResearch | — | 0.75 | ~30s |
| FusionAgent | 0.93 | 0.82 | ~60s+ |

### 学习成果
- [x] 能画出评估系统类继承体系图
- [x] 能解释三层回退机制和 max 策略
- [x] 能完成 3 个代码 Walk-Through（每个 2-3 分钟）
- [x] 能回答 Q16-Q20 评估系统专项追问
- [x] 完成模拟面试

> 📁 笔记文件：[day10_note.md](./day10_note.md)（827行）+ [day10_note_QA.md](./day10_note_QA.md)

---

## 关键文件速查表

| 模块 | 关键文件 | 核心作用 |
|------|----------|----------|
| 图构建 | `integrations/build/main.py` | 全量构建入口（三步串行） |
| 文本分块 | `pipelines/ingestion/text_chunker.py` | HanLP 中文分词 + 滑动窗口 |
| 实体提取 | `graph/extraction/` | LLM-based 三元组提取 |
| 相似检测 | `graph/processing/similar_entity.py` | KNN + WCC 连通分量 |
| 实体合并 | `graph/processing/entity_merger.py` | `apoc.refactor.mergeNodes` |
| 实体消歧 | `graph/processing/entity_disambiguation.py` | 三阶段漏斗管道 |
| 实体对齐 | `graph/processing/entity_alignment.py` | Jaccard 冲突 + CALL 子查询 |
| 社区检测 | `community/detector/leiden.py` | Leiden 模块度优化 + 多层级 |
| Agent 基类 | `agents/base.py` | LangGraph 状态机 + 双缓存 |
| 本地搜索 | `search/local_search.py` | 图邻居扩展 + retrieval_query |
| 全局搜索 | `search/global_search.py` | 社区摘要 Map-Reduce |
| 深度研究 | `search/tool/deep_research_tool.py` | 多轮 Think-Search-Reason |
| 多 Agent | `agents/multi_agent/` | P-E-R 三阶段编排 |
| 缓存系统 | `cache_manager/` | 两层语义缓存 |
| 评估系统 | `evaluation/` | 4 维度 18 指标评估框架 |
| 配置 | `config/settings.py` | 图谱 Schema 定义 |
| 服务端 | `server/services/agent_service.py` | AgentManager 会话隔离 |

---

## 辅助学习材料

| 文件 | 说明 |
|------|------|
| `project_improvements.md` | 独立发现的 10 个架构问题与优化建议 |
| `day*_note_QA.md`（共 10 份） | 每日学习的深度 Q&A 补充 |

---

## 面试关键词备忘

在面试中高频触达这些词汇，展现技术深度：

`Knowledge Graph` · `Entity Disambiguation (Levenshtein + Embedding + NIL)` · `Community Detection (Leiden / SLLPA)` · `Chain of Exploration` · `Plan-Execute-Report` · `Map-Reduce Report Generation` · `Semantic Cache (SentenceTransformer)` · `LangGraph StateGraph` · `Two-tier Caching` · `Incremental Update (file_registry)` · `Consistency Checking` · `Evidence Tracking` · `Topological Sort (Kahn)` · `Jaccard Similarity` · `WCC (Weakly Connected Components)` · `Three-tier Scoring Fallback` · `CompositeEvaluator (Strategy Pattern)`

---

## 面试核心数据速记

| 数据点 | 数值 | 出处 |
|--------|------|------|
| FusionAgent LLM 综合评分 | 0.93 | Day 10 评估 |
| DeepResearch F1 | 0.75 | Day 10 评估 |
| NaiveRAG F1 | 0.43 | Day 10 评估 |
| 实体消歧去重率 | ~30% | Day 7 实验 |
| 评估指标维度 | 4 维度 18 指标 | Day 10 |
| Agent 种类 | 5 种 | Day 1 |
| 消歧 NIL 阈值 | 0.75 | Day 5 |
| 向量/文本组合权重 | 0.6/0.4 | Day 5 |
| Jaccard 冲突阈值 | 0.5 | Day 5 |
| LLM 评估 4 维权重 | 全面性 0.3 / 相关性 0.25 / 增强理解 0.25 / 直接性 0.2 | Day 10 |

---

*制定日期：2026-02-22 | 根据实际学习记录重新整理：2026-03-01 | 项目路径：`/home/wkt/project/graph-rag-agent`*
