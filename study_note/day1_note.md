# Day 1 学习笔记 — 项目全貌与技术背景

> 日期：2026-02-22

---

## 一、为什么需要 GraphRAG？

### 传统 RAG 的局限

```
用户问题 → 向量化 → 相似度召回文档片段 → 拼上下文 → LLM 回答
```

**核心问题**：向量检索只能找"语义相近的文本片段"，**无法理解实体之间的关系**。

- 示例：问"张三和李四有什么关联？"——若两人分别出现在不同文档，向量检索各自召回，但无法感知两者之间的关系边。

### GraphRAG 的解法

```
文档 → LLM抽取实体关系 → 知识图谱(Neo4j) → 本地/全局图搜索 → 丰富上下文 → LLM回答
```

**核心优势**：
- **本地搜索**：以实体为中心，展开 1-2 跳邻居，得到"关系网络"上下文
- **全局搜索**：通过社区检测将图分群，每个社区生成摘要，回答全局性问题（统计、总结）

### DeepSearch 融合

本项目创新：DeepSearch（通常基于向量库）嫁接到知识图谱，实现 Chain of Exploration：

```
问题 → 分解子任务 → 多步图上探索 → 积累证据 → 最终综合回答
```

---

## 二、项目三层架构

```
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Agent 推理层 (LangGraph StateGraph)            │
│  NaiveRAG → GraphAgent → HybridAgent                   │
│  DeepResearchAgent → FusionGraphRAGAgent                │
└──────────────────────┬──────────────────────────────────┘
                       │ 调用搜索工具
┌──────────────────────▼──────────────────────────────────┐
│ Layer 2: 搜索检索层                                      │
│  LocalSearch(图邻居扩展) → GlobalSearch(社区摘要聚合)    │
│  HybridSearch → DeepResearchTool(Chain of Exploration)  │
└──────────────────────┬──────────────────────────────────┘
                       │ 读取图数据
┌──────────────────────▼──────────────────────────────────┐
│ Layer 1: 图构建层                                        │
│  文档摄取 → 分块 → LLM实体提取 → Neo4j存储              │
│  → 社区检测(Leiden/SLLPA) → 向量索引                    │
└─────────────────────────────────────────────────────────┘
```

---

## 三、真实文件 ↔ 架构对应表

### Layer 1：图构建层

| 文件/目录 | 作用 |
|-----------|------|
| `pipelines/ingestion/file_reader.py` | 解析 PDF/DOCX/TXT/CSV 等格式 |
| `pipelines/ingestion/text_chunker.py` | 长文本分块 |
| `pipelines/ingestion/document_processor.py` | 主流程控制 |
| `graph/extraction/` | LLM 提取实体和关系三元组 |
| `graph/processing/` | 实体消歧 & 实体对齐 |
| `graph/indexing/` | 向量索引管理 |
| `community/detector/` | Leiden / SLLPA 社区检测算法 |
| `community/summary/` | LLM 生成社区摘要 |
| `integrations/build/main.py` | ⭐ 图构建总入口（三步顺序执行） |

### Layer 2：搜索检索层

| 文件 | 作用 |
|------|------|
| `search/local_search.py` | 实体中心 + 图邻居扩展 |
| `search/global_search.py` | 社区摘要聚合 + Map-Reduce |
| `search/tool/deep_research_tool.py` | Chain of Exploration 多步探索 |
| `search/tool/naive_search_tool.py` | 纯向量检索 |
| `search/tool_registry.py` | Agent 注册/调用搜索工具的接口 |

### Layer 3：Agent 推理层

| 文件 | 作用 |
|------|------|
| `agents/base.py` | ⭐ 所有 Agent 的父类，StateGraph 骨架 |
| `agents/naive_rag_agent.py` | 向量召回 → 回答 |
| `agents/graph_agent.py` | 图搜索（local + global） → 回答 |
| `agents/hybrid_agent.py` | 三路搜索（local + global + naive） |
| `agents/deep_research_agent.py` | 多步推理 + 思考过程可视化 |
| `agents/fusion_agent.py` | ⭐ 调用整个 multi_agent 子系统 |
| `agents/multi_agent/planner/` | Clarifier → TaskDecomposer → PlanReviewer |
| `agents/multi_agent/executor/` | WorkerCoordinator 调度执行器 |
| `agents/multi_agent/reporter/` | OutlineBuilder → SectionWriter → ConsistencyChecker |

---

## 四、5 种 Agent 对比（面试必背）

| Agent | 适用问题类型 | 技术特点 | 速度 |
|-------|------------|---------|------|
| **NaiveRagAgent** | 简单事实查询 | 纯向量相似度检索 | 最快 |
| **GraphAgent** | 关系推理（A和B的关系） | 图邻居扩展 + 社区搜索 | 中 |
| **HybridAgent** | 通用问答 | local + global + naive 三路叠加 | 中 |
| **DeepResearchAgent** | 需要多步分析的复杂问题 | Chain of Exploration + 思考可视化 | 慢 |
| **FusionGraphRAGAgent** | 深度分析 + 长文档输出 | Plan → 并行Execute → Map-Reduce Report | 最慢/最全 |

> 💡 **记忆口诀**：5个Agent是能力递进——从"一个人查字典"到"团队分工写研究报告"。

---

## 五、技术选型面试答案

### 为什么选 Neo4j？
Neo4j 是原生图数据库，使用 Cypher 查询语言，支持图算法库（GDS）——社区检测 Leiden 就是用 GDS 实现的。多跳遍历性能远优于关系型数据库模拟图，且有 LangChain 官方集成。

### 为什么选 LangGraph？
LangGraph 原生支持**有状态的 Agent**：`StateGraph` + `MemorySaver` 实现多轮对话记忆。工作流建模为有向图，支持条件路由（检索质量低时 → 触发 reduce 路由），比纯链式更灵活可控。

### 为什么用 Streamlit 做前端？
项目定位是 AI 研究系统，Streamlit 让 Python 工程师直接写前端，开发效率极高。调试模式下的 LangGraph 轨迹可视化、Neo4j 图交互组件都是 Streamlit 生态现成的。

---

## 六、自测问题 & 答案

**Q1：传统 RAG 为什么处理不好"张三和李四什么关系"这类问题？**

> 传统 RAG 按相似度召回孤立文档片段，张三在文档A、李四在文档B，系统看不到两者之间的关系边。GraphRAG 用图存储，两者的关系被显式建模为一条边，局部搜索时直接能展开邻居关系网络。

**Q2：用户问"整个知识库里哪个主题出现频率最高"，用 LocalSearch 还是 GlobalSearch？**

> **GlobalSearch**。全局总结类问题需要俯瞰整个知识库，社区摘要提供了不同主题群落的"宏观摘要"，再做 Map-Reduce 聚合能回答这类统计性全局问题。LocalSearch 只擅长围绕特定实体展开，不适合全局统计。

---

## 七、Day 1 打卡

- [x] 能口述项目解决的核心问题（传统 RAG 不够 → GraphRAG 图结构补充）
- [x] 能说清 5 种 Agent 的名字和适用场景
- [x] 理解三层架构及其对应文件位置
- [x] 能回答技术选型（Neo4j / LangGraph / Streamlit）的选择理由
