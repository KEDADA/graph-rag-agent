# Graph-RAG-Agent 项目：10天深度掌握 & 实习面试复现指南

> **目标**：10天内深度理解该项目，能够在面试中用 STAR 法则流利叙述你的贡献与技术难点。
>
> **当前日期**：2026-02-22（第1天从今天开始）

---

## 四阶段总览

| 阶段 | 天数 | 重点 |
|------|------|------|
| 架构通读 | Day 1-2 | 全局理解 + 环境搭建 |
| 核心模块深挖 | Day 3-6 | 逐模块阅读代码 + 理解原理 |
| 复现实践 | Day 7-8 | 亲手跑通 + 能改参数 |
| 面试备战 | Day 9-10 | STAR 话术打磨 + 模拟答题 |

---

## Day 1（2/22）— 项目全貌与技术背景

### 今日目标
建立项目的"鸟瞰图"，弄懂"为什么做这个 & 做了什么"。

### 学习任务

**1. 读懂 README（约 1h）**
- 重点看：项目亮点、功能模块、Agent 类型表
- 关键问题：GraphRAG vs 传统 RAG 有什么本质区别？

**2. 理解技术背景（约 1h）**
- 核心概念图谱：
  ```
  传统 RAG    → 向量检索 → 上下文拼接 → LLM 回答
  GraphRAG   → 知识图谱 → 本地/全局搜索 → 社区摘要 → 更丰富的上下文
  DeepSearch → 多步推理 → 证据追踪 → 逻辑链输出
  ```

**3. 梳理技术栈（约 0.5h）**

| 组件 | 技术 | 作用 |
|------|------|------|
| 知识图谱存储 | Neo4j | 持久化图数据库 |
| LLM 编排 | LangGraph | Agent 状态机 |
| 向量检索 | Embedding + 向量索引 | 语义搜索 |
| 后端 | FastAPI | RESTful API |
| 前端 | Streamlit | 可视化交互 |

**4. 绘制项目架构草图（约 0.5h）**
- 参考 [assets/structure.svg](file:///home/wkt/project/graph-rag-agent/assets/structure.svg)
- 手绘三层结构：图构建层 → 搜索检索层 → Agent 推理层

### 今日打卡
- [ ] 能口述项目解决的核心问题（为什么传统 RAG 不够）
- [ ] 能说清 5 种 Agent 的名字和适用场景

---

## Day 2（2/23）— 环境搭建 + 图构建流程

### 今日目标
搭建运行环境，理解知识图谱的构建全流程。

### 学习任务

**1. 环境配置（约 1h）**
```bash
conda create -n graphrag python==3.10
conda activate graphrag
pip install -r requirements.txt
pip install -e .
docker compose up -d   # 启动 Neo4j
```
- 阅读 [.env.example](file:///home/wkt/project/graph-rag-agent/.env.example)：理解每个关键配置项含义（LLM、Neo4j、缓存）

**2. 研读图构建入口（约 1.5h）**
- 文件路径：[graphrag_agent/integrations/build/main.py](file:///home/wkt/project/graph-rag-agent/graphrag_agent/integrations/build/main.py)
- 流程序列：
  ```
  1. KnowledgeGraphBuilder  → 实体/关系提取 → Neo4j 存储
  2. IndexCommunityBuilder  → 社区检测（Leiden/SLLPA）+ 社区摘要
  3. ChunkIndexBuilder      → 文本块向量索引
  ```
- **注意**：必须按顺序执行，Chunk 索引依赖实体索引

**3. 阅读文档摄取管道（约 1h）**
- `graphrag_agent/pipelines/ingestion/`
  - `file_reader.py`：多格式解析（PDF/DOCX/CSV/JSON 等）
  - `text_chunker.py`：文本分块策略
  - `document_processor.py`：主控流程

**4. 关键技术：实体提取（LLM-based）**
- 查看 `graphrag_agent/graph/extraction/` 目录
- 核心思路：将文本块 + schema 喂给 LLM，提取结构化的 (实体, 关系, 实体) 三元组

### 今日打卡
- [ ] 能解释 `文件 → 分块 → 实体提取 → 图存储 → 社区检测` 全流程
- [ ] 理解 `file_registry.json` 在增量更新中的作用

---

## Day 3（2/24）— Agent 基础架构深挖

### 今日目标
彻底弄懂 BaseAgent 和 LangGraph 状态机机制。

### 学习任务

**1. 精读 BaseAgent（约 2h）**
- 文件：`graphrag_agent/agents/base.py`（约 750 行）
- 重点理解：
  - `StateGraph` 构建过程（节点定义 + 边连接）
  - 双 LLM 实例：`self.llm` 和 `self.stream_llm` 的区别
  - 两级缓存：`cache_manager`（会话级）+ `global_cache_manager`（跨会话）
  - `MemorySaver`：对话状态持久化原理

**2. 阅读 NaiveRagAgent（约 0.5h）**
- 文件：`graphrag_agent/agents/naive_rag_agent.py`
- 理解最基础的向量检索流程：向量召回 → 上下文拼接 → LLM 回答

**3. 阅读 GraphAgent（约 1h）**
- 文件：`graphrag_agent/agents/graph_agent.py`
- 与 NaiveRAG 的核心差异：以实体为中心，通过图邻居扩展上下文

**4. LangGraph 概念梳理**
```python
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)   # 推理节点
graph.add_node("tools", tool_node)    # 工具执行节点
graph.add_edge("agent", "tools")      # 节点连接
graph.add_conditional_edges(...)      # 条件路由
```

### 今日打卡
- [ ] 能在白纸上画出 BaseAgent 的 LangGraph 状态图
- [ ] 理解缓存两层设计的原因（为什么会话级+全局级分开）

---

## Day 4（2/25）— 搜索策略深挖

### 今日目标
深入理解本地搜索、全局搜索、深度研究三种策略的原理与实现。

### 学习任务

**1. 本地搜索（约 1.5h）**
- 文件：`graphrag_agent/search/local_search.py`
- 核心算法：
  ```
  用户问题 → 实体向量召回 → 图邻居扩展（1-2跳）→ 子图提取 → 上下文组装
  ```
- 关注：如何将图结构（节点+边）序列化成 LLM 可读的文本上下文

**2. 全局搜索（约 1h）**
- 文件：`graphrag_agent/search/global_search.py`
- 核心算法：
  ```
  社区摘要向量库 → 语义匹配 → Map-Reduce → 综合回答
  ```
- 关注：如何应对"全局性问题"（统计类、比较类问题）

**3. 深度研究工具（约 1h）**
- 文件：`graphrag_agent/search/tool/deep_research_tool.py`
- Chain of Exploration：在知识图谱上多步探索
  ```
  问题分解 → 子图探索 → 中间结论 → 继续探索 → 证据汇总
  ```

**4. 工具注册机制**
- 文件：`graphrag_agent/search/tool_registry.py`
- 理解 Agent 如何动态注册和使用搜索工具

### 今日打卡
- [ ] 能解释本地搜索 vs 全局搜索的适用场景差异
- [ ] 理解"社区摘要"在全局搜索中的关键作用

---

## Day 5（2/26）— 实体质量与社区检测

### 今日目标
理解实体消歧、实体对齐以及社区检测的原理。

### 学习任务

**1. 实体消歧（约 1.5h）**
- 目录：`graphrag_agent/graph/processing/`
- 三步流程：
  ```
  字符串召回（模糊匹配）→ 向量重排（Embedding 相似度）→ NIL 检测（确认是否为新实体）
  ```
- 解决问题：同一实体用不同表达描述（"习总书记" vs "习近平"）

**2. 实体对齐（冲突解决）（约 1h）**
- 解决问题：同一 canonical 实体下属性冲突（两条记录描述矛盾）
- 策略：`manual_first` / `auto_first` / `merge`

**3. 社区检测（约 1h）**
- 目录：`graphrag_agent/community/`
- Leiden 算法：优化模块度的高性能社区检测
- SLLPA 算法：基于传播的软标签方法（Leiden 不收敛时回退）
- 社区摘要生成：LLM 对社区内节点描述生成结构化摘要

### 今日打卡
- [ ] 能解释实体消歧三步骤（字符串→向量→NIL）
- [ ] 理解为什么需要社区检测（而不是只用向量检索）

---

## Day 6（2/27）— FusionAgent 多智能体架构

### 今日目标
深入理解 Plan-Execute-Report 多智能体架构（项目最核心的亮点）。

### 学习任务

**1. 整体架构梳理（约 1h）**
- 目录：`graphrag_agent/agents/multi_agent/`
- 三阶段流程：`Plan（规划）→ Execute（执行）→ Report（报告）`

**2. Planner 深挖（约 1h）**
- `planner/`：Clarifier → TaskDecomposer → PlanReviewer
- 输出 `PlanSpec`：结构化任务图（含依赖关系 `depends_on`）
- 关键设计：**任务 DAG（有向无环图）** 支持并行与串行混合执行

**3. Executor 深挖（约 1h）**
- `executor/`：RetrievalExecutor、ResearchExecutor、ReflectionExecutor
- WorkerCoordinator：根据任务类型调度对应 Executor
- ReflectionExecutor：对执行结果进行自动复核（answer_validator）

**4. Reporter 深挖（约 0.5h）**
- `reporter/`：OutlineBuilder → SectionWriter → ConsistencyChecker
- Map-Reduce 模式：各节 SectionWriter 并行写作，ConsistencyChecker 做事实核查

**5. 阅读 fusion_agent.py**
- 文件：`graphrag_agent/agents/fusion_agent.py`
- 理解如何作为最高层入口调用多智能体系统

### 今日打卡
- [ ] 能在白板上画出 Plan-Execute-Report 的完整数据流图
- [ ] 理解 PlanSpec 中 `depends_on` 如何实现任务依赖调度

---

## Day 7（2/28）— 亲手复现：图构建实战

### 今日目标
真正跑通知识图谱构建，能在 Neo4j 中看到图数据。

### 实战任务

**1. 配置 .env 文件**
```bash
cp .env.example .env
# 填入：LLM API Key、Neo4j 密码、Embedding 模型名称
```

**2. 准备测试数据**
- 在 `files/` 放入 2-3 个 txt/pdf 文件
- 推荐使用项目默认数据集（`datasets/` 目录）

**3. 执行图构建**
```bash
python graphrag_agent/integrations/build/main.py
```
- 观察输出日志：实体提取数量、关系数量、社区数量
- 打开 Neo4j Browser（http://localhost:7474）查看图

**4. 测试增量更新**
```bash
# 增加一个新文件到 files/
python graphrag_agent/integrations/build/incremental_update.py --once
```

**5. 记录实验结果**
- 截图 Neo4j 中的图结构
- 记录：文件数 / 实体数 / 关系数 / 社区数

### 今日打卡
- [ ] Neo4j Browser 中可看到知识图谱节点和关系
- [ ] 能解释增量更新时 `file_registry.json` 的变化

---

## Day 8（3/1）— 亲手复现：Agent 问答实战

### 今日目标
跑通所有 Agent，对比不同 Agent 的回答质量。

### 实战任务

**1. 启动服务**
```bash
python server/main.py        # 后端 FastAPI
streamlit run frontend/app.py  # 前端可视化
```

**2. 测试各 Agent**
```bash
cd test/
python search_without_stream.py   # 逐一测试各 Agent
```
- 对同一个问题分别测试 `GraphAgent`、`HybridAgent`、`DeepResearchAgent`、`FusionGraphRAGAgent`
- 记录：回答质量 / 用时 / 证据来源 / PlanSpec 结构

**3. 开启调试模式**
- 在前端切换到 Debug 模式
- 观察：LangGraph 轨迹 / 命中的知识图谱节点 / 文档源内容

**4. 调参实验**
- 修改 `.env` 中的 `MAX_WORKERS`、`BATCH_SIZE`，观察性能变化
- 修改 `graphrag_agent/config/settings.py` 中的 `entity_types`，重新构建图

### 今日打卡
- [ ] 能对同一问题展示 4 种以上 Agent 的对比结果
- [ ] 能解释为什么 FusionAgent 回答最长但用时也最久

---

## Day 9（3/2）— 面试 STAR 话术打磨

### 今日目标
将项目经历转化为可在面试中流利讲述的 STAR 结构故事。

---

### Story 1：项目整体介绍

**S（Situation）**：传统 RAG 基于纯向量检索，在处理关系推理类问题时缺乏图结构信息，答案片面。

**T（Task）**：从零复现 GraphRAG，并创新性地融合 DeepSearch 框架，构建一套可解释的多 Agent 知识问答系统。

**A（Action）**：
1. 基于 LLM 驱动的实体关系提取，将文档解析为知识图谱（Neo4j）
2. 实现本地搜索（实体邻居扩展）和全局搜索（社区摘要聚合）
3. 设计 Plan-Execute-Report 多智能体架构，支持复杂问题分解与长文档生成

**R（Result）**：系统支持 5 种 Agent 模式，配备 20+ 评估指标，FusionAgent 在复杂问题上显著优于传统 RAG

---

### Story 2：技术亮点 — 实体消歧

**S**：同一实体在不同文档中用不同表达出现，导致图谱出现大量重复、冲突节点。

**T**：设计并实现三步式实体消歧管道，确保图谱实体的唯一性和准确性。

**A**：
1. **字符串召回**：基于编辑距离，快速筛选候选实体
2. **向量重排**：用 Embedding 模型计算语义相似度，找到最佳匹配
3. **NIL 检测**：若无高置信匹配，判定为新实体而非合并

**R**：实体消歧后图谱节点去重率明显提升，提高了后续检索的准确性

---

### Story 3：技术难点 — 多 Agent 任务调度

**S**：处理复杂用户问题时，单个 Agent 上下文有限，无法生成有深度的长文档。

**T**：设计 Plan-Execute-Report 多智能体协作架构，实现自动任务规划和并行执行。

**A**：
1. **Planner** 将问题分解为 DAG 格式的 `PlanSpec`（含任务依赖关系）
2. **WorkerCoordinator** 根据 `depends_on` 拓扑排序，并行调度独立任务
3. **Reporter** 采用 Map-Reduce：并行生成各章节，再用 ConsistencyChecker 做事实一致性校验

**R**：FusionAgent 可生成 5000+ 字的结构化长文档，并自动完成证据引用和事实核验

---

### 常见面试追问准备

| 面试问题 | 简明回答要点 |
|----------|-------------|
| GraphRAG 和传统 RAG 核心区别？ | 传统 RAG 用向量相似度检索孤立文档片段；GraphRAG 用知识图谱捕获实体关系，支持多跳推理 |
| 为什么用 LangGraph？ | 原生支持状态机（有记忆的 Agent）、工具调用、流式输出，比手写状态机更健壮 |
| 缓存两层架构是什么？ | 会话缓存：同一对话内复用；全局缓存：跨会话语义去重，用向量相似度判断缓存命中 |
| Leiden vs SLLPA 社区检测？ | Leiden 基于模块度优化，结果更稳定；SLLPA 基于标签传播，速度更快，Leiden 不收敛时回退 |
| 流式输出的实现？ | 当前是"伪流式"：LLM 完整生成后切片推送，受 LangChain 版本制约，真流式待框架升级 |
| 增量更新如何防止重复建图？ | `file_registry.json` 记录已处理文件的 hash，增量时仅处理新增/修改文件 |
| 有什么遗留问题？ | Embedding 语义近似导致"优秀学生（荣誉）"被混淆为"奖学金"，需领域 Embedding 微调 |

---

## Day 10（3/3）— 模拟答题 + 深化薄弱点

### 今日目标
完成完整模拟面试，查漏补缺。

### 任务

**1. 模拟技术面试（自问自答，录音或写下来）**
- 用 2 分钟介绍项目（不看笔记）
- 模拟追问：「LangGraph 的状态图是怎么设计的？」、「实体消歧会漏判吗，怎么保证？」

**2. 评估系统了解（加分项）**
- 目录：`graphrag_agent/evaluation/`
- 面试亮点：「我们设计了 20+ 维度评估体系，覆盖答案质量、检索性能和图谱质量」

**3. 准备项目代码讲解**
- 能现场带面试官 walk-through 的 3 个关键文件：
  - `agents/base.py`（LangGraph 架构）
  - `agents/multi_agent/planner/`（任务分解逻辑）
  - `search/local_search.py`（图检索核心）

### 最终检验
- [ ] 能用 2 分钟流利介绍项目（S→T→A→R 结构）
- [ ] 能回答表格中全部 7 个追问
- [ ] 能现场打开代码，指着代码解释 3 个技术难点

---

## 关键文件速查表

| 模块 | 关键文件 | 核心作用 |
|------|----------|----------|
| 图构建 | `integrations/build/main.py` | 全量构建入口 |
| 实体提取 | `graph/extraction/` | LLM-based 三元组提取 |
| 实体消歧 | `graph/processing/` | 三步式去重管道 |
| 社区检测 | `community/detector/` | Leiden/SLLPA 算法 |
| Agent 基类 | `agents/base.py` | LangGraph 状态机 |
| 本地搜索 | `search/local_search.py` | 图邻居扩展检索 |
| 全局搜索 | `search/global_search.py` | 社区摘要聚合检索 |
| 多 Agent | `agents/multi_agent/` | P-E-R 三阶段编排 |
| 缓存系统 | `cache_manager/` | 两层语义缓存 |
| 评估系统 | `evaluation/` | 20+ 指标评估框架 |
| 配置 | `config/settings.py` | 图谱 Schema 定义 |

---

## 面试关键词备忘

在面试中高频触达这些词汇，展现技术深度：

`Knowledge Graph` · `Entity Disambiguation` · `Community Detection (Leiden)` · `Chain of Exploration` · `Plan-Execute-Report` · `Map-Reduce Report Generation` · `Semantic Cache` · `LangGraph StateGraph` · `Two-tier Caching` · `Incremental Update` · `Consistency Checking` · `Evidence Tracking`

---

*制定日期：2026-02-22 | 项目路径：`/home/wkt/project/graph-rag-agent`*
