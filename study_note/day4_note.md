# Day 4 学习笔记 — 搜索策略深挖：从向量检索到深度研究

> 日期：2026-02-26
>
> **阅读前提**：已完成 Day 1-3，理解图构建全流程和 Agent 架构（LangGraph 状态机 + BaseAgent 模板方法模式）。

---

## 写在前面：Day 4 要解决的核心问题

Day 3 我们学会了 Agent 的决策流程：`agent 节点决策 → retrieve 节点搜索 → generate 节点生成回答`。

但 retrieve 节点内部到底做了什么？当 LLM 决定"我要调 `local_retriever`"或"我要调 `global_retriever`"的时候，这两个工具的搜索过程完全不同。

Day 4 的目标就是把这个"黑盒"打开，彻底弄懂：

```
① 本地搜索（Local Search）—— 以实体为中心的图检索
② 全局搜索（Global Search）—— 以社区为中心的 Map-Reduce 检索
③ 混合搜索（Hybrid Search）—— 双级检索：具体细节 + 宏观主题
④ 深度研究（Deep Research）—— 多轮 思考→搜索→推理 的迭代过程
⑤ 工具注册机制（Tool Registry）—— Agent 怎么"认识"这些搜索工具
⑥ 两层封装架构 —— 为什么搜索逻辑要拆成"核心类"和"工具封装类"
```

---

## 一、先看全貌：搜索模块的整体架构

### 1.1 目录结构

```
graphrag_agent/search/
├── local_search.py           # 本地搜索核心逻辑（231行）
├── global_search.py          # 全局搜索核心逻辑（151行）
├── retrieval_adapter.py      # 检索结果标准化适配器（200行）
├── utils.py                  # 向量工具函数
├── tool_registry.py          # 工具注册表（66行）
│
└── tool/                     # 工具封装层
    ├── base.py               # 搜索工具基类（289行）
    ├── local_search_tool.py  # 本地搜索工具（266行）
    ├── global_search_tool.py # 全局搜索工具（373行）
    ├── hybrid_tool.py        # 混合搜索工具（662行）
    ├── naive_search_tool.py  # 基础向量搜索工具
    ├── deep_research_tool.py # 深度研究工具（1221行）⭐ 最复杂
    ├── chain_exploration_tool.py  # 图谱路径探索工具
    ├── hypothesis_tool.py    # 假设生成工具
    ├── validation_tool.py    # 答案验证工具
    └── reasoning/            # 推理引擎子模块
        ├── thinking.py       # 思考引擎（762行）
        ├── search.py         # 双路径搜索器 + 查询生成器
        ├── chain_of_exploration.py
        └── ...
```

### 1.2 两层封装架构（核心设计模式）

> ❗️ 这是本项目搜索模块最重要的架构设计，面试必答。

项目把搜索功能拆成了**两层**：

```
┌─────────────────────────────────────────────────┐
│          第一层：核心搜索类                        │
│          local_search.py / global_search.py       │
│                                                   │
│   职责：纯粹的搜索逻辑                             │
│   - 执行 Cypher 查询                              │
│   - 调用 Neo4j 向量索引                           │
│   - 执行 Map-Reduce                              │
│   - 返回原始搜索结果                               │
│                                                   │
│   特点：不依赖 LangChain 工具协议                   │
│         可以独立调用和测试                          │
└─────────────────────────────────────────────────┘
                        ↓ 被包装
┌─────────────────────────────────────────────────┐
│          第二层：工具封装类                        │
│          tool/local_search_tool.py 等             │
│                                                   │
│   职责：适配 LangChain/LangGraph 工具协议          │
│   - 继承 BaseSearchTool（统一缓存、性能监控）      │
│   - 实现 get_tool() → 返回 BaseTool 实例          │
│   - 提供 extract_keywords() → 关键词提取          │
│   - 提供 structured_search() → 结构化输出          │
│                                                   │
│   特点：Agent 通过 get_tool() 获取可调用的工具      │
│         是 Agent 和底层搜索之间的"桥梁"            │
└─────────────────────────────────────────────────┘
```

**为什么要拆两层？**

| 问题 | 如果只有一层 | 两层解决方案 |
|------|------------|-------------|
| 想单独测试搜索逻辑 | 必须构造完整的 LangChain 环境 | 直接调用 `LocalSearch.search()` |
| 想换 Agent 框架 | 搜索和工具协议耦合 | 只改第二层，核心搜索不动 |
| 想直接调 API 不走 Agent | 搜索逻辑藏在工具类里 | 直接用核心类 |
| 想加缓存、监控等通用能力 | 每个搜索类都要写一遍 | 在 BaseSearchTool 统一提供 |

> **面试话术**：*"我们采用了两层封装：底层核心类负责纯搜索逻辑，上层工具类适配 LangChain 协议并增加缓存、性能监控等横切关注点。这样搜索逻辑可以独立测试，也便于未来更换 Agent 框架。"*

---

## 二、BaseSearchTool：搜索工具的统一基类

> 对应源码：`search/tool/base.py`（289行）
>
> 类似 Day 3 的 `BaseAgent` 对 Agent 的作用，`BaseSearchTool` 对搜索工具做了同样的事——**抽取公共能力**。

### 2.1 BaseSearchTool 提供了什么

```python
class BaseSearchTool(ABC):
    def __init__(self, cache_dir="./cache/search"):
        # ① 统一的模型实例
        self.llm = get_llm_model()
        self.embeddings = get_embeddings_model()

        # ② 统一的缓存
        self.cache_manager = CacheManager(
            key_strategy=ContextAndKeywordAwareCacheKeyStrategy(),
            storage_backend=MemoryCacheBackend(max_size=200),
            cache_dir=cache_dir
        )

        # ③ 性能监控
        self.performance_metrics = {
            "query_time": 0,
            "llm_time": 0,
            "total_time": 0
        }

        # ④ Neo4j 连接
        self.graph = db_manager.get_graph()
        self.driver = db_manager.get_driver()
```

### 2.2 三个抽象方法（子类必须实现）

```python
@abstractmethod
def _setup_chains(self):
    """设置 LLM 处理链"""

@abstractmethod
def extract_keywords(self, query):
    """提取查询关键词"""

@abstractmethod
def search(self, query):
    """执行搜索"""
```

### 2.3 公共搜索方法（子类可直接使用）

```python
# 向量搜索：在 Neo4j 向量索引中找相似实体
def vector_search(self, query, limit=5) -> List[str]:
    query_embedding = self.embeddings.embed_query(query)
    cypher = "CALL db.index.vector.queryNodes('vector', $limit, $embedding) ..."
    return entity_ids

# 文本搜索：全文匹配（向量搜索失败时的备选方案）
def text_search(self, query, limit=5) -> List[str]:
    cypher = "MATCH (e:__Entity__) WHERE e.id CONTAINS $query ..."
    return entity_ids

# 语义排序：对一组实体按语义相似度重排
def semantic_search(self, query, entities, top_k=5):
    return ranked_entities

# 相关性过滤：按相关性过滤文档
def filter_by_relevance(self, query, docs, top_k=5):
    return filtered_docs
```

### 2.4 继承关系全景图

```
BaseSearchTool (ABC)
  ├── LocalSearchTool      ← 包装 LocalSearch
  ├── GlobalSearchTool     ← 包装 GlobalSearch (Map-Reduce)
  ├── HybridSearchTool     ← 双级检索（low_level + high_level）
  ├── NaiveSearchTool      ← 最简单的纯 Chunk 向量搜索
  ├── DeepResearchTool     ← 最复杂，多轮思考-搜索-推理
  └── DeeperResearchTool   ← Deep的增强版
```

---

## 三、本地搜索（Local Search）— 以实体为中心的图检索

> 对应源码：`search/local_search.py`（231行）+ `search/tool/local_search_tool.py`（266行）
>
> **一句话总结**：把用户问题转成向量 → 在实体向量索引中找到最相似的实体 → 以这些实体为中心，沿图谱关系"扩展"出周围的 Chunk、社区、关系 → 拼成上下文给 LLM 回答。

### 3.1 核心算法流程

```
用户问题："旷课多少学时会被退学？"
     │
     ▼
┌──────────────────────────────────────────────────────┐
│ Step 1：向量检索实体                                   │
│                                                        │
│   query_embedding = embeddings.embed_query("旷课...")  │
│   → 在 Neo4j 的 "vector" 向量索引中找 top_entities     │
│     个最相似的 __Entity__ 节点                          │
│   → 假设找到了：[退学处理, 旷课, 学时标准]               │
│                                                        │
│   ⚙️ 参数：top_entities = 10 （最多返回10个实体）       │
└──────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────┐
│ Step 2：图邻居扩展（一条 Cypher 查询完成）              │
│                                                        │
│   以找到的实体为中心，沿图谱的边向外扩展，抓取 5 类信息：│
│                                                        │
│   ① Chunks（文本块）                                   │
│      MATCH (entity)<-[:MENTIONS]-(chunk:__Chunk__)      │
│      → 提取引用了这些实体的原文段落                      │
│      ⚙️ 参数：top_chunks = 3                           │
│                                                        │
│   ② Reports（社区摘要）                                │
│      MATCH (entity)-[:IN_COMMUNITY]->(community)        │
│      → 提取实体所属社区的摘要                           │
│      ⚙️ 参数：top_communities = 3                      │
│                                                        │
│   ③ Outside Relationships（外部关系）                  │
│      MATCH (entity)-[r]-(other) WHERE other NOT IN 召回  │
│      → 当前实体 与 召回集合外的实体 之间的关系描述        │
│      ⚙️ 参数：top_outside_relationships = 10           │
│                                                        │
│   ④ Inside Relationships（内部关系）                   │
│      MATCH (entity)-[r]-(other) WHERE other IN 召回      │
│      → 召回的实体们相互之间的关系描述                    │
│      ⚙️ 参数：top_inside_relationships = 10            │
│                                                        │
│   ⑤ Entities（实体描述）                               │
│      → 直接返回每个实体的 description 字段               │
└──────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────┐
│ Step 3：组装上下文 → LLM 生成回答                       │
│                                                        │
│   将 5 类信息拼成一个大 JSON：                          │
│   {                                                    │
│     "Chunks": [...原文段落...],                         │
│     "Reports": [...社区摘要...],                        │
│     "Relationships": [...关系描述...],                  │
│     "Entities": [...实体描述...]                        │
│   }                                                    │
│                                                        │
│   → 塞入 Prompt 的 {context} 占位符                    │
│   → LLM 基于这些上下文生成回答                          │
└──────────────────────────────────────────────────────┘
```

### 3.2 那条关键的 Cypher 查询（retrieval_query）

本地搜索的精华都在 `local_search.py` 的 `retrieval_query` 属性中——一条 Cypher 查询同时抓取 5 类信息：

```cypher
-- 输入：nodes = 向量检索召回的实体列表
WITH collect(node) as nodes

-- ① 找到提到这些实体的 Chunk（按频率排序）
WITH collect {
    UNWIND nodes as n
    MATCH (n)<-[:MENTIONS]-(c:__Chunk__)
    WITH distinct c, count(distinct n) as freq  -- freq = 这个 chunk 提到了几个召回实体
    RETURN {id:c.id, text: c.text} AS chunkText
    ORDER BY freq DESC           -- 提到越多实体的 chunk 越靠前
    LIMIT $topChunks             -- 默认3个
} AS text_mapping,

-- ② 找到实体所属的社区摘要
-- 💡 关于 community_rank、社区 weight、关系 weight 的详细解释，请看 [Day 4 QA](day4_note_QA.md#q1社区等级community_rank社区权重weight以及关系权重weight分别是什么怎么定义的)
collect {
    UNWIND nodes as n
    MATCH (n)-[:IN_COMMUNITY]->(c:__Community__)
    WITH distinct c, c.community_rank as rank, c.weight AS weight
    RETURN c.summary
    ORDER BY rank, weight DESC   -- 按社区等级和权重排序
    LIMIT $topCommunities
} AS report_mapping,

-- ③ 外部关系：实体与召回集外的节点之间的关系
collect {
    UNWIND nodes as n
    MATCH (n)-[r]-(m:__Entity__)
    WHERE NOT m IN nodes         -- 只看集合外的
    RETURN r.description AS descriptionText
    ORDER BY r.weight DESC       -- 权重高的关系优先
    LIMIT $topOutsideRels
} as outsideRels,

-- ④ 内部关系：召回的实体们之间的关系
collect {
    UNWIND nodes as n
    MATCH (n)-[r]-(m:__Entity__)
    WHERE m IN nodes             -- 只看集合内的
    RETURN r.description AS descriptionText
    ORDER BY r.weight DESC
    LIMIT $topInsideRels
} as insideRels,

-- ⑤ 实体自身的描述
collect {
    UNWIND nodes as n
    RETURN n.description AS descriptionText
} as entities

-- 最终返回一个大 JSON
RETURN {
    Chunks: text_mapping,
    Reports: report_mapping,
    Relationships: outsideRels + insideRels,
    Entities: entities
} AS text, 1.0 AS score, {} AS metadata
```

> **为什么要区分"内部关系"和"外部关系"？**
>
> 内部关系（insideRels）= 召回的实体**彼此之间**的关系。比如"退学处理"和"旷课"之间有关系"累计X学时以上"。这些关系直接回答了用户的问题。
>
> 外部关系（outsideRels）= 召回的实体和**图谱中其他实体**的关系。比如"退学处理"还和"学业警告"有关系"退学前需经学业警告"。这些关系提供了额外的上下文。
>
> 两者结合，让 LLM 既有"核心答案"又有"补充背景"。

### 3.3 社区权重初始化

`LocalSearch.__init__()` 时会执行一条 Cypher：

```python
def _init_community_weights(self):
    self.db_query("""
    MATCH (n:`__Community__`)<-[:IN_COMMUNITY]-()<-[:MENTIONS]-(c)
    WITH n, count(distinct c) AS chunkCount
    SET n.weight = chunkCount
    """)
```

**含义**：一个社区的 weight = 有多少个不同的 Chunk 引用了这个社区的成员实体。weight 越大，说明这个社区在文档中"出镜率"越高，信息密度越大。排序时优先返回这些高权重社区。

### 3.4 Neo4jVector —— 向量检索是怎么做到的

项目使用 LangChain 的 `Neo4jVector` 来做向量检索。这不是自己实现的——它是 Neo4j 原生支持的向量索引功能：

```python
# 连接到已有的向量索引
vector_store = Neo4jVector.from_existing_index(
    embeddings,                       # Embedding 模型
    url=neo4j_uri,
    index_name="vector",              # 索引名称（图构建时创建的）
    retrieval_query=final_query       # ← 自定义的检索查询（就是上面那条 Cypher）
)

# 执行相似度搜索
docs = vector_store.similarity_search(query, k=10)
```

**关键点**：`retrieval_query` 参数让 Neo4jVector 不只是返回"最相似的节点"，而是在找到节点后执行自定义的 Cypher 查询——这就是把向量检索和图邻居扩展**一步完成**的技巧。

### 3.5 工具封装层做了什么额外的事

`LocalSearchTool`（tool 层）在核心搜索之上增加了：
> 💡关于这里的历史感知检索是如何改写问题、完整 RAG 链是怎么把图谱数据和 LLM 生成串联起来的详细代码级解析，请看 [Day 4 QA](day4_note_QA.md#q2localsearchtool工具层在-localsearch核心层之上具体增加了什么代码里是怎么实现的)

```python
class LocalSearchTool(BaseSearchTool):
    def __init__(self):
        self.local_searcher = LocalSearch(...)    # 创建核心搜索实例
        self.retriever = self.local_searcher.as_retriever()  # 获取检索器

        # ① 历史感知检索器——支持多轮对话
        self.history_aware_retriever = create_history_aware_retriever(
            self.llm, self.retriever, contextualize_q_prompt
        )
        # 如果用户说"那上海市奖学金呢？"，这个检索器会自动读 chat_history，
        # 把问题改写为"上海市奖学金的申请条件是什么"再去检索

        # ② 完整 RAG 链 = 检索 + 生成
        self.rag_chain = create_retrieval_chain(
            self.history_aware_retriever,
            self.question_answer_chain
        )

        # ③ 关键词提取链
        self.keyword_chain = keyword_prompt | self.llm | StrOutputParser()
```

> **核心搜索类** vs **工具封装类**的分工：
>
> | 功能 | `LocalSearch`（核心） | `LocalSearchTool`（工具） |
> |------|---------------------|-------------------------|
> | 向量检索 + 图邻居扩展 | ✅ | 通过核心类实现 |
> | Neo4j 连接管理 | ✅ | 通过核心类实现 |
> | 多轮对话支持 | ❌ | ✅ history_aware_retriever |
> | 关键词提取 | ❌ | ✅ keyword_chain |
> | 缓存 | ❌ | ✅ 继承自 BaseSearchTool |
> | 性能监控 | ❌ | ✅ performance_metrics |
> | get_tool() 适配 LangChain | ❌ | ✅ 返回 BaseTool 实例 |

### 3.6 子类对三个抽象方法的实现 (源码级解析)

`BaseSearchTool` 中规定了子类必须实现三个抽象方法，`LocalSearchTool` 是如何结合图谱和上下文来交答卷的？

#### 1. `_setup_chains(self)`：组装问答大脑
**实现机制**：在这里 `LocalSearchTool` 初始化了三个关键的 LangChain 组件，搭建了从"提取查询"到"生成答案"的流水线。

```python
def _setup_chains(self):
    # 1. 创建历史感知检索器
    # 作用：如果用户问"那上海市奖学金呢？"，它会结合 chat_history 
    # 把问题重写为"上海市奖学金的申请条件是什么"，然后再去执行图谱检索。
    contextualize_q_prompt = ChatPromptTemplate.from_messages([...])
    self.history_aware_retriever = create_history_aware_retriever(
        self.llm, self.retriever, contextualize_q_prompt
    )

    # 2. 创建核心问答链 
    # 作用：将检索到的图谱数据（包含节点、关系、社区上下文等）和重写后的问题一起交给 LLM 生成大段回答。
    lc_prompt_with_history = ChatPromptTemplate.from_messages([...])
    self.question_answer_chain = create_stuff_documents_chain(
        self.llm, lc_prompt_with_history
    )

    # 3. 组合成完整的 RAG 链
    self.rag_chain = create_retrieval_chain(
        self.history_aware_retriever, self.question_answer_chain
    )
    
    # 4. 关键词提取链 (用于 extract_keywords 方法)
    self.keyword_chain = self.keyword_prompt | self.llm | StrOutputParser()
```
**一句话总结**：把底层的纯图谱查询，升级为了一个**懂上下文对话**的问答引擎。

#### 2. `extract_keywords(self, query)`：抽取图谱查询词
**实现机制**：通过 LLM 识别用户输入中的核心实体，并**自带缓存机制**。

```python
def extract_keywords(self, query: str) -> Dict[str, List[str]]:
    # 查缓存，避免重复调用 LLM
    cached_keywords = self.cache_manager.get(f"keywords:{query}")
    if cached_keywords: return cached_keywords
        
    # 调用 _setup_chains 里准备好的 keyword_chain
    result = self.keyword_chain.invoke({"query": query})
    keywords = json.loads(result) # 格式 {"low_level": [], "high_level": []}
    
    # 写入缓存
    self.cache_manager.set(f"keywords:{query}", keywords)
    return keywords
```
**为什么做这一步？** 知识图谱的向量检索和精确匹配非常依赖准确的实体名词。直接拿原问题"帮我查一下旷课怎么退学"去搜，可能不如提取出的 `["旷课", "退学"]` 效果好。

#### 3. `search(self, query)` 及其实际工作者 `structured_search`
**实现机制**：`search` 为了兼容旧接口只返回字符串，实际的脏活累活都在 `structured_search` 里，实现了一个完整的闭环：

```python
def structured_search(self, query_input: Any) -> Dict[str, Any]:
    # 0. 解析输入并生成双缓存的 Key 
    parsed = self._normalize_input(query_input)
    query, keywords = parsed["query"], parsed["keywords"]
    
    # 普通缓存键："问题||词1,词2"；结构化缓存键：追加 "::structured"
    cache_key = query if not keywords else f"{query}||{','.join(sorted(keywords))}"
    structured_cache_key = f"{cache_key}::structured"

    # 1. 查缓存（双重设计）
    # 分别查 "structured"（带元数据的完整结果）和 纯文本 answer
    cached_structured = self.cache_manager.get(structured_cache_key)
    if cached_structured: return cached_structured

    # 2. 核心大戏：执行 RAG 链！
    # 这里会依次触发：改写问题 -> 查图谱(Cypher) -> 丢给 LLM 生成最终答案
    chain_output = self.rag_chain.invoke({
        "input": query, 
        "chat_history": self.chat_history
    })
    
    # 3. 结果标准化转换 (适配器模式)
    # 把杂乱的 documents 统一转为 RetrievalResult 格式，方便 Agent 统一消费处理
    answer = chain_output.get("answer")
    documents = chain_output.get("context") or []
    retrieval_results = results_to_payload(
        results_from_documents(documents, source="local_search")
    )
    
    # 构造要返回并缓存的完整字典结构
    structured_result = {
        "query": query,
        "keywords": keywords,
        "answer": answer,
        "retrieval_results": retrieval_results,
        "raw_context": [{"page_content": d.page_content} for d in documents]
    }

    # 4. 存入双缓存并返回
    self.cache_manager.set(structured_cache_key, structured_result)
    self.cache_manager.set(cache_key, answer)
    return structured_result
```
**这个流程的含金量**：它不仅查到了答案，还借用 `results_from_documents` 完成了**数据协议的标准化**（不同搜索工具查出来的数据结构千奇百怪，但经过 Tool 层后，Agent 拿到的都是统一的 `RetrievalResult` 列表，这是架构设计里非常加分的一环）。

---

## 四、全局搜索（Global Search）— 以社区为中心的 Map-Reduce 检索

> 对应源码：`search/global_search.py`（151行）+ `search/tool/global_search_tool.py`（373行）
>
> **一句话总结**：从图谱的社区摘要中找到与问题相关的社区 → 对每个社区独立提问（Map）→ 汇总所有社区的回答（Reduce）→ 生成综合性回答。

### 4.1 什么时候用全局搜索

| 问题类型 | 适合的搜索 | 原因 |
|---------|-----------|------|
| "旷课退学的具体规定" | 本地搜索 ✅ | 答案在一两个 Chunk 里 |
| "学校的学生管理体系整体框架" | 全局搜索 ✅ | 需要多个社区的信息汇总 |
| "奖学金和助学金有什么区别" | 全局搜索 ✅ | 需要跨社区比较 |
| "所有学生处分类型有哪些" | 全局搜索 ✅ | 需要列举分散在不同地方的信息 |

**核心区别**：本地搜索找"一棵树"，全局搜索看"整片森林"。

### 4.2 核心算法：Map-Reduce

```
用户问题："学校的学生管理制度总体框架是什么？"
     │
     ▼
┌─────────────────────────────────────────────────┐
│ Step 1：获取社区数据                               │
│                                                    │
│   MATCH (c:__Community__) WHERE c.level = 0        │
│   RETURN c.full_content                            │
│                                                    │
│   → 返回所有 level=0 的社区的完整内容                │
│   可能有 10-20 个社区                               │
│                                                    │
│   ⚙️ 参数：level = 0（默认查第0层社区）             │
│    Day 5 会讲社区层级的含义                         │
└─────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────┐
│ Step 2：Map 阶段 —— 对每个社区独立提问              │
│                                                    │
│   for 每个社区:                                    │
│       response = LLM(                             │
│           system: "你是数据分析助手...",             │
│           human: "根据以下社区信息回答问题：          │
│                  {社区的full_content}               │
│                  问题：{用户问题}"                  │
│       )                                           │
│                                                    │
│   → 产出 N 个独立的"中间结果"                      │
│   每个中间结果 = 这个社区对用户问题能提供的信息       │
│                                                    │
│   💡 有的社区跟问题无关（比如奖学金社区之于纪律问题）  │
│   → Map 阶段的 LLM 会回答"此社区无相关信息"         │
└─────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────┐
│ Step 3：Reduce 阶段 —— 汇总所有中间结果            │
│                                                    │
│   response = LLM(                                 │
│       system: "你是一位专业的分析报告撰写者...",      │
│       human: "基于以下各社区的分析报告：              │
│              {所有中间结果}                         │
│              请综合整理回答用户问题：{问题}           │
│              回答类型：{多个段落}"                   │
│   )                                               │
│                                                    │
│   → 产出一个去重、连贯的综合回答                    │
└─────────────────────────────────────────────────┘
```

### 4.3 Map-Reduce 图解

```
社区A（奖学金管理）──→ Map ──→ "国家奖学金评选由学工处负责..."
社区B（纪律处分）  ──→ Map ──→ "学生违纪可受到以下处分..."
社区C（学籍管理）  ──→ Map ──→ "学生注册、转专业流程..."     ──→ Reduce ──→ 综合回答
社区D（体育管理）  ──→ Map ──→ "此社区无相关信息"
社区E（资助体系）  ──→ Map ──→ "学校设有助学金、贷款..."
```

### 4.4 工具封装层的增强

`GlobalSearchTool` 相比核心 `GlobalSearch` 多了几个重要能力：

**① 关键词过滤社区**

核心类遍历所有社区，很慢。工具类先用关键词过滤：

```python
def _get_community_data(self, keywords=None):
    cypher_query = """
    MATCH (c:__Community__) WHERE c.level = $level
    """
    # 如果有关键词，只返回包含关键词的社区
    if keywords:
        cypher_query += " AND (c.full_content CONTAINS $keyword0 OR ...)"

    # 按社区等级和权重排序，最多返回20个
    cypher_query += """
    ORDER BY c.community_rank DESC, c.weight DESC
    LIMIT 20
    """
```

这意味着：不再"遍历全部社区"，而是"先筛选再 Map-Reduce"，大幅减少 LLM 调用次数。

**② 批处理优化**

核心类逐个社区调用 LLM（N 次 API 调用）。工具类把社区合并成批次：

```python
batch_size = 5  # 每批5个社区
for i in range(0, len(communities), batch_size):
    batch = communities[i:i+batch_size]
    # 多个社区的内容拼在一起，一次 LLM 调用处理
    batch_result = self._process_community_batch(query, batch)
```

这把 20 次 LLM 调用减少到 4 次，显著降低延迟和成本。

**③ 结构化输出**

```python
structured_result = {
    "query": query,
    "keywords": keywords,
    "intermediate_results": intermediate_results,  # Map 阶段的中间结果
    "final_answer": final_answer,                  # Reduce 后的最终回答
    "retrieval_results": retrieval_payload,         # 标准化的 RetrievalResult
}
```

### 4.5 本地搜索 vs 全局搜索：完整对比

| 维度 | 本地搜索 | 全局搜索 |
|------|---------|---------|
| **检索对象** | 实体 + 关系 + Chunk + 社区摘要 | 社区完整内容 |
| **检索方式** | 向量相似度 → 图邻居扩展 | 关键词过滤 → Map-Reduce |
| **LLM 调用次数** | 1次（生成回答） | N+1次（N次Map + 1次Reduce） |
| **适用问题** | 具体细节问题 | 宏观概括/比较/统计问题 |
| **延迟** | 较快（1-3秒） | 较慢（5-15秒，取决于社区数量） |
| **上下文质量** | 精确但可能遗漏 | 全面但可能包含无关信息 |
| **上下文质量** | 精确但可能遗漏 | 全面但可能包含无关信息 |
| **设计思路** | "精确打击" | "地毯式搜索" |

### 4.6 子类对三个抽象方法的实现 (源码级解析)

和 `LocalSearchTool` 一样，`GlobalSearchTool` 也要上交这三份答卷，但它的解题思路完全是为了 Map-Reduce 服务的。

#### 1. `_setup_chains(self)`：配置 Map-Reduce 双引擎
**实现机制**：不需要像本地搜索那样建立连续对话的历史感知检索，而是非常干脆地设立了 Map 和 Reduce 两个处理阶段的独立大模型链路。

```python
def _setup_chains(self):
    # 1. 设置 Map 阶段的处理链
    # 作用：拿着用户的问题，去各个独立社区里逐个“抓取”相关信息。
    map_prompt = ChatPromptTemplate.from_messages([
        ("system", MAP_SYSTEM_PROMPT),
        ("human", GLOBAL_SEARCH_MAP_PROMPT),
    ])
    self.map_chain = map_prompt | self.llm | StrOutputParser()
    
    # 2. 设置 Reduce 阶段的处理链
    # 作用：把上面所有社区抓回来的零碎片段大杂烩，提炼整合成一篇连贯的、有条理的最终报告。
    reduce_prompt = ChatPromptTemplate.from_messages([
        ("system", REDUCE_SYSTEM_PROMPT),
        ("human", GLOBAL_SEARCH_REDUCE_PROMPT),
    ])
    self.reduce_chain = reduce_prompt | self.llm | StrOutputParser()
    
    # 3. 关键词提取链 (同理，服务于 extract_keywords)
    self.keyword_prompt = ChatPromptTemplate.from_messages([...])
    self.keyword_chain = self.keyword_prompt | self.llm | StrOutputParser()
```
**一句话总结**：把一个全局大探方，变成了“分头打探（Map）”和“汇编呈报（Reduce）”的两台职能引擎。

#### 2. `extract_keywords(self, query)`：仅提取高级概念
**实现机制**：虽然调用流程（查缓存 -> 调模型 -> 写缓存）和本地搜索长得一样，但**对结果的处理逻辑大相径庭**。

```python
def extract_keywords(self, query: str) -> Dict[str, List[str]]:
    # ... 中间查缓存、调 LLM 步骤同 LocalSearchTool ...
    
    # 最大的区别：由于是全局搜索，不需要纠结细枝末节的 low_level 实体
    # 直接将 LLM 返回的所有词全部塞进 high_level (抽象概念/主题) 中使用！
    if isinstance(keywords, list):
        formatted_keywords = {
            "keywords": keywords,
            "low_level": [],
            "high_level": keywords  # 全局搜索主要关注高级概念！
        }
    
    self.cache_manager.set(f"keywords:{query}", formatted_keywords)
    return formatted_keywords
```
**为什么做这一步？** 这对应了 `4.4 ①` 中的过滤优化。有了这个高级关键词提取，后续用 Cypher 取社区时，只需要取其 `full_content` 包含这些相关高级词汇的社区，那些完全无关的社区就被直接过滤掉了（比如搜“怎么转校”，就不会去查关于“学生会招新”的社区），省了巨量的 API 重复成本！

#### 3. `search(self, query)` 及其实际工作者 `structured_search`
**实现机制**：如果说 `LocalSearchTool` 是简单粗暴调 `rag_chain` 一遍过，那 `GlobalSearchTool` 就是一个精密的三部曲任务流调度核心：

```python
def structured_search(self, query_input: Any) -> Dict[str, Any]:
    # 0. 规范化及双缓存生成... (与本地搜索逻辑一致)
    
    # 1. 过滤获取候选层数据（取社区资料）
    # 返回的是那些包含刚提取出的 keywords 的社区数据
    community_data = self._get_community_data(keywords)
    if not community_data:
        return {"intermediate_results": [], "final_answer": ""}

    # 2. 分头打探（触发 Map 引擎，且内部有**批处理**优化）
    # 对着过滤出的数十个社区，按 5个/批 调用 map_chain，得到各个批次的中期提炼报告
    intermediate_results = self._process_communities(query, community_data)
    
    # 3. 总编收网（触发 Reduce 引擎）
    # 将上面的多份报告塞入 reduce_chain 吐出最终结果
    final_answer = self._reduce_results(query, intermediate_results)
    
    # 4. 适配器转换 (构建统一标准的 RetrievalResult)
    # 作用：将零散的社区语料打包成标准的 JSON 数组，专门供前端 UI 渲染“参考资料/溯源卡片”使用
    retrieval_payload = self._community_results_to_retrieval(community_data)

    # 5. 组装缓存装车返回
    structured_result = {
        "query": query, "keywords": keywords,
        "intermediate_results": intermediate_results,
        "final_answer": final_answer,
        "retrieval_results": retrieval_payload,
    }
    self.cache_manager.set(structured_cache_key, structured_result)
    self.cache_manager.set(cache_key, intermediate_results)
    return structured_result
```

> 💡 **关于这里的 `retrieval_payload` 证据数据结构和 `structured_result` 字典长什么样、分别有什么用，请参见**：[Day 4 QA](day4_note_QA.md#q4-globalsearch-中-retrieval_payload-和-structured_result-的区别是什么具体长什么样)

**这个流程的含金量**：这是经典的分布式系统 Map-Reduce 思想在大模型重构知识领域的完美映射！展现了非常优秀的性能调优意识，完美串起了本大节 4.1-4.4 中的所有理论概念！

---

## 五、混合搜索（Hybrid Search）— 双级检索策略

> 对应源码：`search/tool/hybrid_tool.py`（662行）
>
> **一句话总结**：同时执行"本地细节检索"和"全局主题检索"，然后合并结果。

### 5.1 核心思路：双级关键词

```python
def extract_keywords(self, query):
    # LLM 将用户问题拆成两级关键词
    keywords = {
        "low_level": ["旷课", "学时", "退学"],     # 具体实体/细节
        "high_level": ["学籍管理制度", "学业处理"]   # 抽象概念/主题
    }
```

### 5.2 双路检索

```
low_level 关键词 ──→ _retrieve_low_level_content()
                     │  ① 向量搜索匹配实体
                     │  ② 精确文本匹配（备选）
                     │  ③ 沿图谱扩展邻居（1-2跳）
                     │  ④ 返回：实体描述 + 关系描述 + Chunk
                     │
                     ├──→ 合并 ──→ LLM 生成最终回答
                     │
high_level 关键词 ──→ _retrieve_high_level_content()
                     │  ① 在社区的 full_content 中匹配关键词
                     │  ② 按 community_rank 和 weight 排序
                     │  ③ 返回：社区摘要
```

### 5.3 与 GraphAgent 的关系

Day 3 的 `GraphAgent` 通过 LLM 的 **`bind_tools`** 让 LLM 自己选择用"本地"还是"全局"搜索。而 `HybridSearchTool` 是**自动两种都做**，不需要 LLM 决策。这是不同的策略：

| | GraphAgent | HybridSearchTool |
|---|---|---|
| 搜索策略选择 | LLM 决定用哪个 | 自动两种都做 |
| 优点 | 问题简单时节省一种搜索的成本 | 不会错过任何一种信息 |
| 缺点 | LLM 可能选错 | 所有查询都要执行两种搜索 |

---

## 六、深度研究（Deep Research）— 多轮 思考→搜索→推理

> 对应源码：`search/tool/deep_research_tool.py`（1221行）
>
> **一句话总结**：模仿人类研究员的思考过程——提出问题 → 拆解子问题 → 逐步搜索 → 验证假设 → 生成最终答案。
>
> 这是项目中**最复杂的搜索工具**，也是**面试最大的亮点之一**。

### 6.1 整体架构

```
┌───────────────────────────────────────────────────────┐
│                DeepResearchTool                        │
│                                                        │
│  组装了多个子模块：                                     │
│  ┌──────────────────────────────────────────────────┐  │
│  │ ThinkingEngine（思考引擎）                        │  │
│  │   - 维护"对话历史"（system/human/ai messages）    │  │
│  │   - 生成下一步搜索查询                            │  │
│  │   - 假设生成 & 验证                               │  │
│  │   - 反事实分析                                    │  │
│  │   - 推理分支管理                                  │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │ QueryGenerator（查询生成器）                      │  │
│  │   - 将原始问题分解为多个子查询                     │  │
│  │   - 基于已检索信息生成跟进查询                     │  │
│  │   - 生成多假设查询                               │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │ DualPathSearcher（双路径搜索器）                   │  │
│  │   - KB检索路径：本地搜索（精确查询 + 带KB名查询）  │  │
│  │   - KG检索路径：全局搜索（社区检索）              │  │
│  │   - LLM 评估两个路径的结果质量，智能合并           │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │ AnswerValidator（答案验证器）                     │  │
│  │   - 验证最终答案的质量                            │  │
│  │   - 检查是否覆盖了关键词                          │  │
│  └──────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

### 6.2 图解执行过程（与整体架构对应）

> 先结合整体架构图，直观看一遍各子模块在一次真实查询中是怎么协作的，再深入看代码细节。

```
用户问题："学校对旷课、考试作弊分别有什么处理措施？它们之间有什么关联？"
     │
     ▼
┌──────────────────────────────────────────┐
│ 子模块1: QueryGenerator                  │
│   generate_sub_queries()                 │
│ → 子查询1: "旷课处理措施"                │
│ → 子查询2: "考试作弊处理措施"            │
│ → 子查询3: "旷课和作弊处理的关联"        │
└──────────────────────────────────────────┘
     │
     ▼ 第1轮迭代
┌──────────────────────────────────────────┐
│ 子模块3: DualPathSearcher               │
│   search("旷课处理措施")                 │
│   KB路径: LocalSearch → 找到"旷课"实体   │
│   KG路径: GlobalSearch → "纪律处分"社区  │
│   → LLM评估两路结果，取最优或合并        │
│   → LLM提取: "旷课累计超过三分之一学时..."│
└──────────────────────────────────────────┘
     │
     ▼ 第2轮迭代
┌──────────────────────────────────────────┐
│ 子模块2: ThinkingEngine                  │
│   generate_next_query()                  │
│ "我已知各自处理措施，但不清楚关联，继续"  │
│ → 新查询: "旷课与作弊处理的累积关系"       │
└──────────────────────────────────────────┘
     │
     ▼ 信息收集足够
┌──────────────────────────────────────────┐
│ 子模块1: QueryGenerator                  │
│   generate_followup_queries()            │
│ → 返回空列表 → 停止迭代                   │
└──────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│ 子模块4: AnswerValidator                 │
│   _generate_final_answer() + validate()  │
│ → 整合检索信息 + 思考过程 → 输出最终答案  │
└──────────────────────────────────────────┘
```

### 6.3 thinking()：主控调度循环（源码级解析）

这是整个深度研究的主方法（约 220 行），像一个精密的调度中心，指挥着所有子模块按顺序工作：

```python
def thinking(self, query):
    # ── 初始化 ──
    self.thinking_engine.initialize_with_query(query) # 子模块2 初始化对话上下文

    # Step 1：子模块1 生成初始子查询
    initial_sub_queries = self.query_generator.generate_sub_queries(query)
    # 例："旷课退学制度" → ["旷课的定义和计算方式", "退学的条件有哪些", "旷课和退学的关系"]

    # ── 迭代循环（最多 max_iterations 次）──
    for iteration in range(self.max_iterations):

        # Step 2：确定本轮查询
        if iteration == 0:
            queries_to_process = initial_sub_queries[:2]  # 首轮用子模块1生成的子查询
        else:
            # 后续轮由 子模块2 根据当前已有推理，自主决定下一步搜索什么
            result = self.thinking_engine.generate_next_query()
            if result["status"] == "answer_ready":
                break  # 子模块2判断信息已足够，退出
            queries_to_process = result["queries"]

        # Step 3：子模块3 对每个查询执行双路径搜索（带去重）
        for search_query in queries_to_process:
            if self.thinking_engine.has_executed_query(search_query): # 去重检查
                continue
            kbinfos = self.dual_searcher.search(search_query) # KB+KG 双路搜索

            # Step 4：LLM 从搜索结果中提炼有用信息 → 积累到 all_retrieved_info
            summary = self.llm.invoke(RELEVANT_EXTRACTION_PROMPT.format(...))
            if "**Final Information**" in summary:
                self.all_retrieved_info.append(useful_info)

        # Step 5：子模块1 判断信息有无缺口，决定是否继续
        gap_needed = len(self.query_generator.generate_followup_queries(...)) > 0
        if not gap_needed:
            break

    # Step 6：子模块4 生成 + 验证最终答案
    final_answer = self._generate_final_answer(query, retrieved_content, thinking_process)
    return {"thinking_process": think, "answer": final_answer, ...}
```

### 6.4 子模块1： QueryGenerator（查询生成器）

**职责**：将一个复杂问题拆加成可检索的山子查询，以及在搜索结束时判断还有没有信息缺口。

**核心方法三个**：

```python
class QueryGenerator:
    # 方法一：将大问题拆成多个子问题
    def generate_sub_queries(self, query: str) -> List[str]:
        result = self.llm.invoke(SUB_QUERY_PROMPT.format(question=query))
        # Prompt 就是："将问题拆分为3-4个可独立搜索的子问题"
        # 返回样例： ["旷课累计计算方法", "退学门槛具体有哪些", "旷课退学的关参程序"]
        return result

    # 方法二：基于已检索信息判断还有没有信息缺口
    def generate_followup_queries(self, current_info: str, original_query: str) -> List[str]:
        result = self.llm.invoke(FOLLOWUP_QUERY_PROMPT.format(
            question=original_query, information=current_info
        ))
        # Prompt："基于已知信息，列出还需搜索的问题"
        # 如果返回 [] 空列表 → 信息已充分 → 可以停止迭代了！
        return result
```

> **面试亮点**：这就是 Deep Research 和普通搜索的最大不同，它展现了你会“追问”而不是只会“一次搜索完了事”。

### 6.5 子模块2： ThinkingEngine（思考引擎）

**职责**：维护整个推理的对话历史，基于已发现的信息自主决定下一步搜索方向，并支持类 Git 分支式的多路推理路径探索。

**源码中的关键方法**：

| 能力 | 方法 | 说明 |
|------|------|------|
| 初始化思考 | `initialize_with_query(query)` | 用系统提示词设置思考框架 |
| 生成下一步查询 | `generate_next_query()` | 让 LLM 阅读已有对话历史，决定下一步搜索什么 |
| 假设生成 | `generate_hypotheses(thinking)` | 对问题生成多个可能的假设 |
| 假设验证 | `verify_hypothesis(hypothesis)` | 检验假设是否成立 |
| 深度思考 | `think_deeper(query, context)` | 在现有信息基础上深入分析 |
| 反事实分析 | `counter_factual_analysis(h)` | "如果不是这样，会怎样？" |
| 推理分支 | `branch_reasoning(name)` | 类似 Git 分支，探索不同推理路径 |
| 分支合并 | `merge_branches(src, target)` | 将多条推理路径合并 |

```python
# 核心方法示例：generate_next_query 的实际工作方式
# (thinking.py L576)
def generate_next_query(self) -> Dict[str, Any]:
    # 核心：把全部历史对话传给 LLM
    # 包含：系统设定、之前的思考、摞入的属于每次搜索的信息
    response = self.llm.invoke(self.messages)
    
    # 解析 LLM 输出中的下一步查询 (LLM 会在回复中用 <query>...</query> 标签包裹 )
    queries = self.extract_queries(response.content) # 用正则提取
    
    if not queries or "ANSWER_READY" in response.content:
        return {"status": "answer_ready", "queries": []}
    
    return {"status": "continue", "queries": queries}
```

> **面试亮点**：ThinkingEngine 实现了类似 "Chain of Thought" + "Tree of Thought" 的混合推理——既有线性的思考链，也支持分支探索和合并。

### 6.6 子模块3： DualPathSearcher（双路径搜索器）

**职责**：将同一个问题用两种方式搜索知识库，再由 LLM 评估选择或合并两路结果。

```python
# search.py L28
def search(self, query: str) -> Dict:
    # 路径1：精确查询 ("65f7课处理措施")
    precise_query = query.replace(self.kb_name, "").strip()
    precise_results = self.kb_retriever(precise_query) # 调 LocalSearchTool

    # 路径2：带知识库名的查询 ("华东理工大学旷课处理措施")
    kb_query = f"{self.kb_name} {query}" if self.kb_name.lower() not in query.lower() else query
    kb_results = self.kb_retriever(kb_query)         # 调 LocalSearchTool

    # 判断：如果只有一路有内容，直接返回那个 ← 短路返回，节省 LLM 调用
    if precise_has_content and not kb_has_content: return precise_results
    if kb_has_content and not precise_has_content: return kb_results
    if not precise_has_content and not kb_has_content: 
        return self._merge_results(...) # 合并可能的部分结果

    # 两路都有内容，起用 LLM 评估哪路更好
    evaluation = self._evaluate_results_with_llm(query, precise_text, kb_text)
    # LLM 返回："precise" / "kb" / "both"
    if evaluation == "precise": return precise_results
    elif evaluation == "kb":     return kb_results
    else:                        return self._merge_results(precise_results, kb_results)
```

**为什么需要两种路径？** 向量检索对查询的措词非常敏感——"旷课处理"和"华东理工大学旷课处理"在向量空间中可能距离不同。加上知识库名称可能让检索更精准，也可能引入噪音。让 LLM 来判断哪个更好，是一种自适应的应对策略。

### 6.7 子模块4： AnswerValidator（答案验证器）

**职责**：对最终答案做质量把关——只有通过验证的答案才会写入缓存。

```python
# 验证逻辑 (在 deep_research_tool.py 内使用)
validation_results = self.validator.validate(query, answer)
# 验证器内部会：
# 1. 调用 self.extract_keywords(query) 提取关键词
# 2. 检查 answer 是否覆盖了这些关键词（keywords 覆盖率）
# 3. 采用 LLM 对回答品质打分

if validation_results["passed"]:
    self.cache_manager.set(cache_key, answer)  # 只缓存通过验证的高质量答案
else:
    # 不缓存低质量答案，下次同样的问题重新搜索
    pass
```

**为什么设计这步？** 深度研究要消耗 10-30 次 LLM 调用，若生成了质量差的答案（如关键词覆盖率过低）仍缓存，就是在把错误答案定实下来。AnswerValidator 是对这个大操作的**“最后一层保险”**。

---

## 七、Chain of Exploration（图谱路径探索）

> 对应源码：`search/tool/chain_exploration_tool.py`（115行）
>
> **一句话总结**：给定起始实体，沿着图谱的关系链一步步探索，收集路径上的实体和证据。

### 7.1 与其他搜索的区别

```
本地搜索：向量匹配实体 → 扩展邻居（广度优先，1-2跳）
路径探索：指定起点 → 沿关系链深入（深度优先，最多5步）
```

本地搜索像"看周围"，路径探索像"沿路走"。适合回答"A和B之间有什么关系链"这样的推理问题。

### 7.2 使用方式

```python
tool = ChainOfExplorationTool(max_steps=5, exploration_width=3)
result = tool.explore(
    query="退学和旷课的关系",
    start_entities=["旷课"],       # 从"旷课"实体开始
    max_steps=5,                   # 最多走5步
    exploration_width=3            # 每步最多扩展3个方向
)

# 返回结果
{
    "exploration_path": [...],      # 探索路径
    "entities": [...],              # 发现的实体
    "relationships": [...],         # 发现的关系
    "communities": [...],           # 涉及的社区
    "retrieval_results": [...]      # 标准化的检索结果
}
```

---

## 八、工具注册机制（Tool Registry）

> 对应源码：`search/tool_registry.py`（66行）

### 8.1 设计目的

Agent 需要知道"我有哪些搜索工具可用"。如果在每个 Agent 里硬编码工具列表，添加新工具时要改很多地方。

Tool Registry 的作用就是**集中管理所有可用的搜索工具**：

```python
# 主要搜索工具注册表（继承 BaseSearchTool）
TOOL_REGISTRY = {
    "local_search": LocalSearchTool,
    "global_search": GlobalSearchTool,
    "hybrid_search": HybridSearchTool,
    "naive_search": NaiveSearchTool,
    "deep_research": DeepResearchTool,
    "deeper_research": DeeperResearchTool,
}

# 额外工具（不继承 BaseSearchTool）
EXTRA_TOOL_FACTORIES = {
    "chain_exploration": ChainOfExplorationTool,
    "hypothesis_generator": HypothesisGeneratorTool,
    "answer_validator": AnswerValidationTool,
}
```

### 8.2 使用方式

```python
# Agent 中获取工具
from graphrag_agent.search.tool_registry import get_tool_class

ToolClass = get_tool_class("local_search")   # → LocalSearchTool
tool_instance = ToolClass()                   # → 实例化
langchain_tool = tool_instance.get_tool()     # → 获取 LangChain BaseTool

# 或者列出所有可用工具
from graphrag_agent.search.tool_registry import available_tools
all_tools = available_tools()  # → {"local_search": ..., "global_search": ..., ...}
```

### 8.3 两个注册表的区别

| | TOOL_REGISTRY | EXTRA_TOOL_FACTORIES |
|---|---|---|
| 基类 | `BaseSearchTool` | 无统一基类 |
| 用途 | 主要的搜索工具 | 辅助工具（假设生成、答案验证等） |
| 使用方 | Agent 的 `_setup_tools()` | DeepResearchTool 内部使用 |
| 获取方式 | `get_tool_class()` | `create_extra_tool()` |

---

## 九、检索结果标准化（Retrieval Adapter）

> 对应源码：`search/retrieval_adapter.py`（200行）
>
> 这是一个经常被忽略但设计很好的模块。

### 9.1 解决的问题

不同搜索工具返回的结果格式不同：

- 本地搜索返回 LangChain `Document` 对象
- 全局搜索返回社区摘要字符串列表
- 路径探索返回实体+关系字典

但多 Agent 系统（Day 6 的 FusionAgent）需要统一消费这些结果。`retrieval_adapter.py` 就是把各种格式**标准化为 `RetrievalResult`**：

```python
@dataclass
class RetrievalResult:
    result_id: str           # 唯一ID
    granularity: str         # 粒度："Chunk" / "AtomicKnowledge" / "DO"
    evidence: Any            # 证据内容
    metadata: RetrievalMetadata  # 来源、置信度、社区ID等
    source: str              # 来源：local_search / global_search / ...
    score: float             # 相关性得分
```

### 9.2 转换函数

```python
# Document → RetrievalResult
results_from_documents(docs, source="local_search")

# 实体字典 → RetrievalResult
results_from_entities(entities, source="chain_exploration")

# 关系字典 → RetrievalResult
results_from_relationships(rels, source="chain_exploration")

# 合并 + 去重（按 source_id + granularity 去重，保留高分）
merge_retrieval_results(group1, group2, group3)

# 转为可序列化的字典
results_to_payload(results)  # → [{"result_id": ..., "evidence": ..., ...}]
```

---

## 十、所有搜索策略的横向对比

| 维度 | Naive | Local | Global | Hybrid | Deep Research |
|------|-------|-------|--------|--------|--------------|
| **核心思路** | 纯向量匹配 Chunk | 实体向量 → 图邻居扩展 | 社区摘要 → Map-Reduce | 双级关键词 → 双路检索 | 多轮 思考→搜索→推理 |
| **使用知识图谱** | ❌ | ✅ 实体+关系 | ✅ 社区摘要 | ✅ 两者都用 | ✅ 两者都用 + 迭代 |
| **LLM 调用次数** | 1 | 1 | N+1 | 2-3 | 10-30+ |
| **适用场景** | 简单问答 | 具体细节查询 | 宏观概括题 | 需要全面性的查询 | 复杂推理题 |
| **延迟** | ~1s | ~2s | ~10s | ~5s | ~30-60s |
| **代码量** | ~170行 | ~500行 | ~520行 | ~660行 | ~1220行 |
| **面试重要性** | ⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |

---

## 十一、面试要点总结

### STAR 片段：搜索策略设计

**S**：不同类型的用户问题需要不同的检索策略。简单问题只需查找文档片段，复杂问题需要理解实体间关系，全局性问题需要跨社区综合分析。

**T**：设计并实现多种搜索策略，使系统能自动选择最适合的检索方式。

**A**：
1. **本地搜索**：通过向量索引匹配实体，再用 Cypher 查询沿图谱关系扩展 1-2 跳邻居，在一条查询中同时抓取 Chunk、社区摘要、内外部关系 5 类上下文
2. **全局搜索**：基于关键词筛选相关社区，用 Map-Reduce 模式逐社区提取信息再汇总，并通过批处理优化将 API 调用减少 80%
3. **深度研究**：实现多轮迭代的"思考→搜索→推理"循环，包含双路径搜索器（精确查询+带上下文查询）、LLM 结果评估、假设生成验证和答案质量校验
4. 采用**两层封装**架构（核心搜索 + 工具适配），并通过 Tool Registry 集中管理，支持新搜索策略的快速接入

**R**：系统支持 6 种搜索策略，Deep Research 在复杂推理问题上的答案完整性显著优于单次搜索

### 高频面试问答

| 问题 | 要点 |
|------|------|
| 本地搜索的核心流程？ | 向量匹配实体 → 一条 Cypher 抓取 5 类邻居信息（Chunk、社区、内外关系、实体描述）→ 拼成上下文给 LLM |
| 全局搜索为什么用 Map-Reduce？ | 社区摘要之间独立、可能有重复，不能直接拼接。Map 阶段逐社区提问，Reduce 阶段去重合并 |
| 什么是 retrieval_query 的作用？ | 传给 Neo4jVector.from_existing_index 的自定义 Cypher，让向量检索在找到节点后自动执行图邻居扩展 |
| Deep Research 和普通搜索的区别？ | 多轮迭代（不是一次搜索完）+ 自动分解子问题 + 搜索结果评估 + 假设验证 + 答案质量检查 |
| 为什么要双路径搜索（DualPathSearcher）？ | 向量检索对查询措辞敏感，同一问题加不加知识库名结果可能不同，用 LLM 评估哪个更好 |
| 两层封装架构的好处？ | 核心类可独立测试、不依赖框架；工具类统一缓存/监控/适配协议；新增工具只需在 Registry 注册 |
| 社区权重 weight 怎么算的？ | Count(引用了该社区成员的 Chunk 的数量)，"出镜率"高的社区权重大 |
| 全局搜索工具层做了什么优化？ | ① 关键词预筛选社区（不再遍历全部）② 批处理合并多社区一次调 LLM ③ 结构化输出 |

---

## 十二、自测清单

- [ ] 能用自己的话解释本地搜索的三步流程（向量匹配 → 图邻居扩展 → 上下文组装）
- [ ] 能说清 retrieval_query 中 5 类信息的含义和排序逻辑
- [ ] 能解释全局搜索为什么需要 Map-Reduce（而不是拼接社区摘要）
- [ ] 能画出 Deep Research 的迭代循环（初始化 → 子查询 → 搜索 → 提取 → 评估 → 下一轮）
- [ ] 能解释两层封装架构的设计动机
- [ ] 能在面试中用 STAR 结构讲述搜索策略设计

---

## 十三、Day 4 涉及的代码文件索引

| 文件 | 核心类 | 行数 | 本文对应章节 |
|------|-------|------|-------------|
| `search/local_search.py` | `LocalSearch` | 231 | §三 |
| `search/global_search.py` | `GlobalSearch` | 151 | §四 |
| `search/tool/base.py` | `BaseSearchTool` | 289 | §二 |
| `search/tool/local_search_tool.py` | `LocalSearchTool` | 266 | §三 |
| `search/tool/global_search_tool.py` | `GlobalSearchTool` | 373 | §四 |
| `search/tool/hybrid_tool.py` | `HybridSearchTool` | 662 | §五 |
| `search/tool/deep_research_tool.py` | `DeepResearchTool` | 1221 | §六 |
| `search/tool/chain_exploration_tool.py` | `ChainOfExplorationTool` | 115 | §七 |
| `search/tool_registry.py` | `TOOL_REGISTRY` | 66 | §八 |
| `search/retrieval_adapter.py` | `RetrievalResult` | 200 | §九 |
| `search/tool/reasoning/thinking.py` | `ThinkingEngine` | 762 | §六 |
| `search/tool/reasoning/search.py` | `DualPathSearcher` / `QueryGenerator` | 333 | §六 |
| `config/settings.py` | 搜索配置参数 | 351 | 参数引用 |

---

*完成日期：2026-02-26 | 对应计划：Day 4 — 搜索策略深挖*

---

## 十四、项目优化与功能待办
> 💡 在本项目源码阅读过程中，发现了一些值得优化和改进的设计点（如架构缺陷、性能损耗等），已统一整理记录在：[项目优化与待办清单 (Improvements Tracker)](project_improvements.md)，这些也是面试中展示独立思考能力的重要素材。
