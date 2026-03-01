# Day 2 学习笔记 — 环境配置 & 图构建全流程

> 日期：2026-02-22

---

## 一、三层配置体系

项目采用**三层配置**，优先级：`.env` > `settings.py` > 默认值

```
.env                        ← 运行时参数（API Key、密码、性能调参）
graphrag_agent/config/
  settings.py               ← 图谱 Schema（entity_types、关系类型、主题等）
server/ & frontend/         ← 自动继承以上两层配置
```

### `.env` 关键配置项速查

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `OPENAI_API_KEY` | LLM 服务密钥 | `sk-xxx` |
| `OPENAI_BASE_URL` | API 代理地址 | `http://localhost:13000/v1` |
| `OPENAI_LLM_MODEL` | 推理模型名 | `gpt-4o` |
| `OPENAI_EMBEDDINGS_MODEL` | 向量化模型 | `text-embedding-3-large` |
| `NEO4J_URI` | 图数据库地址 | `neo4j://localhost:7687` |
| `NEO4J_PASSWORD` | 数据库密码 | `12345678` |
| `MAX_WORKERS` | 并行线程数 | `4` |
| `CHUNK_SIZE` | 分块 token 数 | `500` |
| `CHUNK_OVERLAP` | 分块重叠 token 数 | `100` |
| `GRAPH_CONFLICT_STRATEGY` | 实体冲突策略 | `manual_first` / `auto_first` / `merge` |
| `GRAPH_COMMUNITY_ALGORITHM` | 社区检测算法 | `leiden` / `sllpa` |

### `settings.py` 关键内容（图谱 Schema）

```python
# 这个项目是华东理工大学学生管理知识库
theme = "华东理工大学学生管理"

entity_types = [
    "学生类型", "奖学金类型", "处分类型",
    "部门", "学生职责", "管理规定"
]

relationship_types = [
    "申请", "评选", "违纪", "资助",
    "申诉", "管理", "权利义务", "互斥"
]
```

> **面试要点**：schema 直接影响实体提取质量——entity_types 告诉 LLM"该抽什么类型的实体"，relationship_types 告诉 LLM"这些实体之间有什么关系"。改变 schema 就是领域定制的核心手段。

---

## 二、图构建全流程解析

### 入口：`integrations/build/main.py`

```python
class KnowledgeGraphProcessor:
    def process_all(self):
        # 步骤 0：清除旧索引（防止冲突）
        connection_manager.drop_all_indexes()

        # 步骤 1：构建基础图谱（实体+关系 → Neo4j）
        KnowledgeGraphBuilder().process()

        # 步骤 2：构建实体向量索引 + 社区检测+摘要
        IndexCommunityBuilder().process()

        # 步骤 3：构建 Chunk 向量索引
        ChunkIndexBuilder().process()
```

> ⚠️ **关键约束**：步骤必须按 1→2→3 执行，Chunk 索引依赖实体索引已建好。

---

## 三、步骤 1 详解：构建基础图谱（`KnowledgeGraphBuilder`）

> 对应源码：`integrations/build/build_graph.py`
> 这是最核心、最耗时的步骤，后续 §3.1~§3.5 全部是此步骤的子环节。

### 3.0 步骤 1 整体流程

```
初始化：LLM + Embedding + Neo4j连接 + DocumentProcessor + EntityRelationExtractor
  ↓
文件处理：DocumentProcessor.process_directory()
  → FileReader  ← 解析 PDF/DOCX/TXT/CSV 等格式
  → ChineseTextChunker ← HanLP 分词 + 滑动窗口分块
  ↓
图结构构建：GraphStructureBuilder
  → 创建 Document 节点
  → 创建 Chunk 节点（大文件并行处理）
  → 建立 Chunk 之间的 NEXT 关系链
  ↓
实体+关系抽取：EntityRelationExtractor（批量调用 LLM）
  → 每个 Chunk + schema 送给 LLM
  → LLM 输出结构化三元组 (实体, 关系, 实体)
  ↓
写入数据库：GraphWriter
  → 将实体节点、关系边批量写入 Neo4j
```

#### `GraphStructureBuilder` 详细步骤（`graph/structure/struct_builder.py`）

这个阶段**不调用任何 LLM**，只负责建立图的骨架结构。

**① 创建 `__Document__` 节点**
```cypher
MERGE(d:`__Document__` {fileName: $file_name})
SET d.type=$type, d.uri=$uri, d.domain=$domain
```
`MERGE` 保证幂等：同一文件重复运行不会重复创建节点。

**② 创建 `__Chunk__` 节点 + `PART_OF` 归属关系**
```cypher
MERGE (c:`__Chunk__` {id: data.id})         -- id = SHA256(文本内容)，内容寻址
SET c.text = data.pg_content,
    c.position = data.position,             -- 第几块（从1开始）
    c.length = data.length,
    c.fileName = data.f_name,
    c.content_offset = data.content_offset, -- 在原文中的字符偏移量
    c.tokens = data.tokens
WITH c, data
MATCH (d:`__Document__` {fileName: data.f_name})
MERGE (c)-[:PART_OF]->(d)                  -- 每块归属哪个文档
```
`id` 用内容 SHA256 生成：相同文本永远是同一节点，**天然去重，支持增量更新**。

**③ 建立顺序关系链**
```cypher
-- 标记第一块
MERGE (d:`__Document__`)-[:FIRST_CHUNK]->(c)
-- 相邻块形成有序链表
MERGE (pc:`__Chunk__`)-[:NEXT_CHUNK]->(c)
```
`NEXT_CHUNK` 链保留原文阅读顺序。Local Search 时可以沿链表向前后展开相邻 chunk，补充单个 chunk 不够的上下文。

**④ 大文件并行处理策略**
```python
if len(chunks) < 100:
    # 串行：create_relation_between_chunks()
else:
    # 并行：parallel_process_chunks()
    # → ThreadPoolExecutor 分批准备数据
    # → 统一批量写入 Neo4j（每次 500 个节点）
```
chunk 数 ≥ 100 时并行构建，避免逐一写 Neo4j 导致大文件处理过慢。

---

### 3.1 文本分块器深挖（`text_chunker.py`）

`ChineseTextChunker` 是专门针对中文的分块器，基于 **HanLP** 分词。

> 👉 **[QA指路] CHUNK_SIZE 过大为什么漏实体？超长文本处理是为了什么？分词器到底在做什么？请看 `day2_note_QA.md` 的 QA-4、QA-5、QA-6**

### 分块策略：滑动窗口 + 句子边界对齐

```python
# 核心参数（来自 settings.py）
CHUNK_SIZE = 500    # 每块目标 token 数
OVERLAP = 100       # 相邻块重叠 token 数（保证上下文连续性）

# 两步走：
# 1. 优先在句子边界（。！？）结束 chunk，避免截断语义
# 2. 下一块回退 overlap 个 token，保证上下文重叠
```

### 超长文本处理（`_preprocess_large_text`）

- 若文本超过 `MAX_TEXT_LENGTH`（默认 50万字），先按段落/句子切分成"段"
- 每段再走滑动窗口分块
- 兜底：遇到超长单句，按固定长度强制切分

### 为什么需要 Overlap？

```
Chunk A: ...上海市奖学金申请...需要满足成绩要求[END]
Chunk B: [START 回退100 token]...成绩要求：绩点3.5以上，且...
```
Overlap 确保同一语义段落不会因切块边界而断裂，后续向量检索时召回更准确。

---

### 3.2 实体提取机制

### Prompt 结构（`config/prompts/`）

```
系统提示（system_template_build_graph）：
  你是一个知识图谱构建专家，请从文本中提取实体和关系。
  实体类型：['学生类型', '奖学金类型', '处分类型', '部门', '学生职责', '管理规定']
  关系类型：['申请', '评选', '违纪', '资助', '申诉', '管理', '权利义务', '互斥']

用户输入（human_template_build_graph）：
  文本：{chunk_text}
  请提取所有实体和关系，输出 JSON 格式...
```

### 并行批处理（性能优化）

```python
if total_chunks > 100:
    # 大数据集：批处理模式（减少线程切换开销）
    entity_extractor.process_chunks_batch(...)
else:
    # 小数据集：并行模式（MAX_WORKERS 线程同时处理）
    entity_extractor.process_chunks(...)
```

---

### 3.3 Neo4j 图结构（步骤 1 构建结果）

> 👉 **[QA指路] 关于 Neo4j 图结构的详细解析（节点属性、关系方向、Cypher 查询示例），请看 `day2_note_QA.md` 的 QA-2**

构建完成后 Neo4j 中的节点类型：

```
Document（文档节点）
  └── CONTAINS → Chunk（文本块节点，含向量）
                    └── NEXT → Chunk（保持顺序关系）
                    └── MENTIONS → Entity（实体节点）

Entity（实体节点，含向量）
  └── [关系类型] → Entity（实体间关系边）
  └── IN_COMMUNITY → Community（社区节点，步骤2添加）
```

---

### 3.4 增量更新机制（`file_registry.json`）

```json
{
  "files/学生手册.pdf": {
    "hash": "sha256_abc...",
    "processed_at": "2025-10-01T12:00:00"
  }
}
```

- 每次构建后记录文件 hash
- 增量时：仅处理**新增**（hash 不存在）或**修改**（hash 变化）的文件
- 删除的文件：标记后从图中移除对应节点

---

### 3.5 性能调参（面试加分项）

> 👉 **[QA指路] 并行 vs 批处理的实现原理是什么？性能调参的面试话术该怎么说？请看 `day2_note_QA.md` 的 QA-1、QA-3**

| `.env` 参数 | 作用 | 调大影响 |
|-------------|------|---------|
| `MAX_WORKERS` | 并行线程数 | 实体提取更快，但 API 并发压力增大 |
| `BATCH_SIZE` | Neo4j 批写大小 | 写入更快，内存占用增加 |
| `CHUNK_SIZE` | 每块 token 数 | 块变大：上下文更完整，但提取精度可能下降 |
| `CHUNK_OVERLAP` | 重叠 token 数 | 语义连续性更好，但冗余内容增加 |
| `GDS_MEMORY_LIMIT` | Neo4j GDS 内存(GB) | 社区检测可处理更大图 |

---

## 四、步骤 2 详解：索引与社区构建（`IndexCommunityBuilder`）

> 对应源码：`integrations/build/build_index_and_community.py`
> 依赖步骤1 已写入的 Entity 节点，共 5 个子步骤。

```
4.1 创建实体向量索引
  -> 4.2 检测相似实体（GDS KNN + WCC）
  -> 4.3 合并相似实体（LLM 判断 + Neo4j 合并）
  -> 4.4 实体消歧与对齐（质量提升）
  -> 4.5 社区检测（Leiden / SLLPA）
```

### 4.1 创建实体向量索引

`create_entity_index()` 实际做了**两步**：

**第一步：调用 Embedding 模型为每个实体生成向量**

```python
# entity_indexer.py 核心逻辑
entities = graph.query('MATCH (e:__Entity__) WHERE e.embedding IS NULL RETURN ...')
# 取出每个实体的 id + description，拼接成文本
entity_text = entity.id + " " + entity.description   # 如 "国家奖学金 由国家设立的最高荣誉奖学金"
# 调用 Embedding API 生成向量
embedding = embeddings_model.embed_documents([entity_text])
# 写回 Neo4j
graph.query('SET e.embedding = $embedding')
```

**第二步：在 Neo4j 上建 Vector Index**

```cypher
CREATE VECTOR INDEX entity_vector_index
FOR (e:__Entity__) ON e.embedding
OPTIONS {indexConfig: {`vector.dimensions`: 1536, `vector.similarityFunction`: 'cosine'}}
```

> 注意区分：步骤1（KnowledgeGraphBuilder）创建了 Entity 节点但**没有**生成 embedding；步骤2 的 4.1 才真正调用 Embedding 模型计算向量并建索引。建好后 Local Search 的 `db.index.vector.queryNodes()` 才能工作。

### 4.2 检测相似实体（`similar_entity.py`）

> 核心问题：不同 Chunk 中对同一事物可能有不同称呼，需要识别出来。
> 例如："国家奖学金"和"国奖"、"学业绩点"和"GPA"、"学生处"和"学工部"

> 👉 **[QA指路] 为什么定义了 entity_types 还会有重复实体？请看 `day2_note_QA.md` 的 QA-7**

#### 什么是 GDS（Graph Data Science）

GDS 是 Neo4j 的**图算法库**，提供 KNN、WCC、Leiden 等算法。它的工作方式：

```
Neo4j 磁盘上的图数据
  -> gds.graph.project()  把子图加载到内存（称为"投影"）
  -> 在内存中跑算法（速度远快于直接查 Neo4j）
  -> 结果写回 Neo4j 节点属性
```

投影是必须的第一步——GDS 算法不直接操作数据库，只在内存投影上计算。

#### 第一步：创建投影（Projection）

```python
# similar_entity.py L103-108
self.G, result = self.gds.graph.project(
    "entities",              # 投影名称
    "__Entity__",            # 只投影 Entity 节点
    "*",                     # 包含所有关系类型
    nodeProperties=["embedding"]  # 带上 embedding 向量
)
```

把所有 Entity 节点（含 embedding 向量）加载到 GDS 内存中，形成一个**内存子图**。

#### 第二步：KNN 算法（K-Nearest Neighbors）

KNN 的作用：**对每个实体，找到向量空间中最相似的 K 个邻居**。

```python
# similar_entity.py L169-176
self.gds.knn.mutate(
    self.G,
    nodeProperties=['embedding'],        # 用 embedding 向量计算相似度
    mutateRelationshipType='SIMILAR',    # 创建 SIMILAR 关系
    mutateProperty='score',              # 相似度分数
    similarityCutoff=0.95,               # 阈值：相似度 > 0.95 才算
    topK=10                              # 每个实体最多找 10 个邻居
)
```

**具体过程（以本项目为例）**：

```
实体"国家奖学金"的 embedding = [0.12, -0.34, 0.56, ...]
实体"国奖"的 embedding       = [0.11, -0.33, 0.55, ...]
                               ↓ 余弦相似度 = 0.97（> 0.95 阈值）
                               ↓ 创建关系：(国家奖学金)-[:SIMILAR {score:0.97}]->(国奖)

实体"学生处"的 embedding      = [0.45, 0.23, -0.12, ...]
实体"奖学金类型"的 embedding  = [-0.32, 0.67, 0.11, ...]
                               ↓ 余弦相似度 = 0.31（< 0.95 阈值）
                               ↓ 不创建关系（这两个不相似）
```

KNN 结束后，Neo4j 中多出 `SIMILAR` 关系边（本项目中创建了 12 条 SIMILAR 关系）。

#### 第三步：WCC 算法（Weakly Connected Components）

WCC 的作用：**在 SIMILAR 关系网络中找出"连通分量"**——即通过 SIMILAR 关系直接或间接连接的实体组。

```python
# similar_entity.py L247-252
self.gds.wcc.write(
    self.G,
    writeProperty="wcc",                 # 结果写入 wcc 属性
    relationshipTypes=["SIMILAR"],       # 只看 SIMILAR 关系
    consecutiveIds=True                  # 连续编号
)
```

**图解**：

```
KNN 产生的 SIMILAR 关系：

  国家奖学金 --SIMILAR-- 国奖
  国家奖学金 --SIMILAR-- 国家级奖学金
                                          ← 连通分量 1（wcc=0）

  学生处 --SIMILAR-- 学工部
                                          ← 连通分量 2（wcc=1）

  绩点 --SIMILAR-- GPA
  学业绩点 --SIMILAR-- 绩点
                                          ← 连通分量 3（wcc=2）

WCC 结果：
  国家奖学金.wcc = 0, 国奖.wcc = 0, 国家级奖学金.wcc = 0   → 同一组
  学生处.wcc = 1, 学工部.wcc = 1                            → 同一组
  绩点.wcc = 2, GPA.wcc = 2, 学业绩点.wcc = 2              → 同一组
```

> **为什么不直接用 KNN 的结果？** KNN 只找到两两之间的相似关系。如果 A 和 B 相似、B 和 C 相似但 A 和 C 不直接相似，KNN 不会把 A-C 关联起来。WCC 通过传递性（A→B→C 连通）把它们归为同组。

#### 第四步：查找候选重复组 + 文本距离过滤

WCC 分组后，还要加一层**文本编辑距离过滤**（防止向量相似但文字完全不同的误判）：

```cypher
-- similar_entity.py L322-358（简化版）
MATCH (e:__Entity__)
WITH e.wcc AS community, collect(e) AS nodes, count(*) AS count
WHERE count > 1                         -- 社区内有多个实体
UNWIND nodes AS node
-- apoc.text.distance 计算编辑距离，< 3 才认为文字也相似
WITH [n IN nodes WHERE apoc.text.distance(toLower(node.id), toLower(n.id)) < 3 | n.id]
  AS intermediate_results
WHERE size(intermediate_results) > 1
RETURN combinedResult                   -- 输出最终候选重复组
```

#### 完整流程总结

```
105 个 Entity（含 embedding）
  ↓ GDS 投影（加载到内存）
  ↓ KNN（cosine > 0.95，topK=10）→ 创建 12 条 SIMILAR 关系
  ↓ WCC（沿 SIMILAR 找连通分量）→ 每个 Entity 写入 wcc 编号
  ↓ 按 wcc 分组 + 编辑距离 < 3 过滤
  → 输出候选重复组，交给 4.3 的 LLM 做最终判断
```

#### 相关参数

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `similarity_threshold` | 0.95 | KNN 相似度阈值，越高越严格 |
| `SIMILAR_ENTITY_TOP_K` | 10 | 每个实体最多找 K 个相似邻居 |
| `SIMILAR_ENTITY_WORD_EDIT_DISTANCE` | 3 | 文本编辑距离阈值（字符级） |
| `GDS_MEMORY_LIMIT` | 6 GB | GDS 投影可用内存上限 |

### 4.3 合并相似实体（`entity_merger.py`）

#### 第一步：LLM 判断是否真的要合并

4.2 输出的是**候选**重复组，还需要 LLM 做最终判断（避免误合并）：

```python
# entity_merger.py 核心逻辑
for group in duplicate_groups:
    # 调用 LLM：「以下实体是否是同一事物？["国家奖学金", "国奖"]」
    answer = chain.invoke({"entities": group})
    # LLM 返回：[["国家奖学金", "国奖"]]  ← 确认要合并
    # 或返回：[]  ← 不合并（LLM 认为不是同一事物）
    merge_groups = convert_to_list(answer)
```

#### 第二步：执行合并 — `apoc.refactor.mergeNodes()`

合并使用的是 Neo4j APOC 库的 `mergeNodes` 函数：

```cypher
-- entity_merger.py L348-360
UNWIND $data AS candidates
CALL {
  WITH candidates
  MATCH (e:__Entity__) WHERE e.id IN candidates
  RETURN collect(e) AS nodes
}
CALL apoc.refactor.mergeNodes(nodes, {properties: {`.*`: 'discard'}})
YIELD node
RETURN count(*) as merged_count
```

#### `mergeNodes` 具体做了什么？

以合并 `["国家奖学金", "国奖"]` 为例：

**合并前**的图结构：

```
Chunk_37 -[:MENTIONS]-> (国家奖学金) -[:评选]-> (学院)
Chunk_102 -[:MENTIONS]-> (国奖)       -[:申请]-> (本科生)
                         (国奖)       -[:SIMILAR]-> (国家奖学金)
```

**`mergeNodes` 自动处理所有关系**：

```
1. 选择第一个节点作为"主节点"（国家奖学金）
2. 将"国奖"的所有入边（MENTIONS）重定向到主节点
3. 将"国奖"的所有出边（申请）重定向从主节点出发
4. 删除"国奖"节点

                            ┌─ properties: {`.*`: 'discard'}
                            │  意思是：属性冲突时丢弃被合并节点的属性，保留主节点的
```

**合并后**的图结构：

```
Chunk_37  -[:MENTIONS]-> (国家奖学金) -[:评选]-> (学院)
Chunk_102 -[:MENTIONS]-> (国家奖学金) -[:申请]-> (本科生)
```

#### 关系处理的具体规则

| 关系类型 | 合并时的处理 |
|----------|------------|
| `MENTIONS`（Chunk → Entity）| 所有 chunk 的 MENTIONS 都指向主节点，**不会丢失任何 chunk 关联** |
| 实体间关系（`申请`/`管理`等）| 被合并节点的关系边全部迁移到主节点 |
| `SIMILAR`（KNN 产生的）| 暂时保留，后续清理 |

#### 第三步：清理重复关系

合并后可能产生重复关系（如主节点原本和节点 C 有关系，被合并节点也和 C 有同类型关系）：

```python
# entity_merger.py L418-426 — 清除同方向重复关系
MATCH (a)-[r]->(b)
WITH a, b, type(r) as type, collect(r) as rels
WHERE size(rels) > 1               -- 同方向同类型有多条
WITH a, b, type, rels[0] as kept, rels[1..] as rels
UNWIND rels as rel
DELETE rel                         -- 只保留一条

# L432-442 — 清除 SIMILAR 关系的双向冗余
MATCH (a)-[r1:SIMILAR]->(b)
MATCH (b)-[r2:SIMILAR]->(a)
WHERE a.id < b.id
DELETE r2                          -- 只保留一个方向
```

#### 完整合并流程图

```
4.2 输出候选重复组：[["国家奖学金","国奖"], ["学生处","学工部"], ...]
  ↓
LLM 逐组判断（并行，ThreadPoolExecutor）
  → 确认合并：[["国家奖学金","国奖"]]
  → 拒绝合并：[]（如果 LLM 认为不是同一事物）
  ↓
apoc.refactor.mergeNodes()（批量执行）
  → 保留主节点，迁移所有关系，删除被合并节点
  ↓
清理重复关系
  → 删除同方向重复边
  → 删除 SIMILAR 双向冗余
```

> **面试要点**：合并不是简单的删除，而是用 `apoc.refactor.mergeNodes` 自动化处理所有关系迁移。这保证了合并后图的**连通性不受损**——被合并节点关联的所有 chunk 和其他实体都不会丢失连接。

### 4.4 实体消歧与对齐（`entity_quality.py`）

> 👉 **[QA指路] 4.3 合并完了为什么 WCC 组里还有多个实体？请看 `day2_note_QA.md` 的 QA-8**

消歧和对齐是 4.3 合并之后的**进一步质量提升**，由 `EntityQualityProcessor` 顺序执行两个阶段。

---

#### 阶段1：消歧（`entity_disambiguation.py`）

> 目的：在 4.3 已合并的实体组内，选出一个**主代表实体（canonical_id）**，为后续对齐做准备。

`apply_to_graph()` 的核心逻辑：

```python
# entity_disambiguation.py L183-214（简化）
# 1. 查询所有 WCC 分组中尚未设置 canonical_id 的多实体组
MATCH (e:__Entity__)
WHERE e.wcc IS NOT NULL AND e.canonical_id IS NULL
WITH e.wcc AS community, collect(e) AS entities
WHERE size(entities) >= 2

# 2. 对每组：选择"度数最高"的实体作为 canonical
canonical = max(entities, key=lambda x: x['degree'])  # degree = 关系边数

# 3. 其他实体标记指向主实体
SET e.canonical_id = $canonical_id, e.disambiguated = true
```

**为什么用"度数最高"**：关系边越多的实体，说明在图中与更多 chunk 和其他实体有连接，它是**最具代表性**的那个。

**具体例子**：

```
WCC 分组 {wcc=0}：["国家奖学金"(度数=15), "国奖"(度数=3)]
  → canonical_id = "国家奖学金"（度数最高）
  → "国奖".canonical_id = "国家奖学金"

WCC 分组 {wcc=1}：["学生处"(度数=8), "学工部"(度数=12)]
  → canonical_id = "学工部"（度数最高）
  → "学生处".canonical_id = "学工部"
```

##### 消歧器还内置了三阶段管道（用于未来扩展）

`disambiguate()` 方法实现了完整的消歧管道，但目前 `apply_to_graph()` 用的是简化版本（直接按度数选 canonical）。完整管道的设计如下：

```
输入 mention（如"国奖"）
  ↓ 阶段1: 字符串召回（Levenshtein 编辑距离 >= 0.7）
  ↓ 阶段2: 向量重排（综合分数 = 0.4×文本相似 + 0.6×向量相似）
  ↓ 阶段3: NIL 检测（最佳候选分数 < 0.6 → 判定为未知实体）
  → 输出 canonical_id 或 NIL
```

> 👉 **[QA指路] 三阶段管道到底有什么用？它和 `apply_to_graph` 的区别是什么？请看 `day2_note_QA.md` 的 QA-10**

---

#### 阶段2：对齐（`entity_alignment.py`）

> 目的：将指向同一个 `canonical_id` 的实体**真正合并**为一个节点，同时处理属性冲突和关系迁移。

对齐分三步：

##### ① 按 canonical_id 分组

```python
# entity_alignment.py L35-44
MATCH (e:__Entity__)
WHERE e.canonical_id IS NOT NULL
WITH e.canonical_id AS canonical_id, collect(e.id) AS entity_ids
WHERE size(entity_ids) >= 2    -- ALIGNMENT_MIN_GROUP_SIZE
RETURN canonical_id, entity_ids
```

##### ② 冲突检测（Jaccard 相似度）

检查同组内实体的**关系类型**是否一致——如果差异太大，可能不该合并：

```python
# entity_alignment.py L100-122
# 收集每个实体的关系类型集合
# 如 "国家奖学金" 的关系类型 = {"评选", "管理", "申请"}
#    "国奖"       的关系类型 = {"评选", "管理"}

# 计算 Jaccard 相似度 = 交集/并集
jaccard = len({"评选","管理"}) / len({"评选","管理","申请"}) = 2/3 = 0.67

# 如果 jaccard < ALIGNMENT_CONFLICT_THRESHOLD(0.5) → 有冲突
# 0.67 > 0.5 → 无冲突，可以直接合并
```

有冲突时，调用 LLM 决定保留哪个实体。

##### ③ 合并实体（带关系迁移）

```cypher
-- entity_alignment.py L178-254（简化）
-- 1. 找到目标节点（canonical）
MERGE (target:__Entity__ {id: $target_id})

-- 2. 处理被删除实体的出边
UNWIND $to_delete AS del_id
MATCH (old:__Entity__ {id: del_id})
OPTIONAL MATCH (old)-[r_out]->(other)
WHERE other.id <> $target_id
-- 如果目标节点没有同类型关系，才迁移（避免重复）
CALL apoc.create.relationship(target, rel_type, rel_props, other)

-- 3. 处理被删除实体的入边（如 MENTIONS）
OPTIONAL MATCH (other)-[r_in]->(old)
CALL apoc.create.relationship(other, rel_type, rel_props, target)

-- 4. 合并属性
SET target.description = COALESCE(target.description, old.description),
    target.aligned_from = COALESCE(target.aligned_from, []) + [old.id]

-- 5. 删除旧实体
DETACH DELETE old
```

> 注意：这里比 4.3 的 `mergeNodes` 更精细——它**检查目标节点是否已有相同类型的关系**，避免创建重复边。

> 👉 **[QA指路] 更直观的理解：冲突检测和合并 5 步的通俗解释及图解例子，请看 `day2_note_QA.md` 的 QA-9**

---

#### 消歧 vs 对齐 vs 4.3合并 的区别

| 步骤 | 做什么 | 核心函数 | 触发条件 |
|------|--------|---------|----------|
| 4.3 合并 | KNN+WCC 候选组 → LLM 确认 → `mergeNodes` 合并 | `apoc.refactor.mergeNodes` | 候选重复组存在 |
| 4.4 消歧 | 在 WCC 分组内选出 canonical（主代表） | `apply_to_graph` | wcc 分组 ≥ 2 个实体 |
| 4.4 对齐 | 按 canonical_id 分组 → 冲突检测 → 合并节点 | `merge_entities` | canonical_id 分组 ≥ 2 |

三步是递进关系：4.3 做粗粒度合并，4.4 消歧选主代表，4.4 对齐做细粒度合并并处理关系冲突。

---

#### 相关参数详解

| 参数 | 默认值 | 用在哪里 | 含义 |
|------|--------|---------|------|
| `DISAMBIG_STRING_THRESHOLD` | 0.7 | 消歧-字符串召回 | Levenshtein 相似度阈值，≥ 0.7 的实体才进入候选。值越低召回越多但噪声也多 |
| `DISAMBIG_VECTOR_THRESHOLD` | 0.85 | 消歧-向量重排 | 向量相似度的参考阈值（当前代码中未直接用作硬阈值） |
| `DISAMBIG_NIL_THRESHOLD` | 0.6 | 消歧-NIL 检测 | 综合分数（0.4×文本+0.6×向量）低于此值，判定为未知实体。值越高越容易判为 NIL |
| `DISAMBIG_TOP_K` | 5 | 消歧-字符串召回 | 每个 mention 最多召回多少个候选实体 |
| `ALIGNMENT_CONFLICT_THRESHOLD` | 0.5 | 对齐-冲突检测 | 关系类型的 Jaccard 相似度低于此值视为有冲突，需要 LLM 介入。值越高越容易触发 LLM |
| `ALIGNMENT_MIN_GROUP_SIZE` | 2 | 对齐-分组 | canonical_id 分组中至少要有多少个实体才需要对齐 |

### 4.5 社区检测（`community/detector/`）

#### 为什么需要社区检测？

前面的步骤构建了实体和关系，但用户可能会问**宏观问题**：

```
"学校的学生管理制度整体上包含哪些方面？"
"奖学金体系是怎么运作的？"
```

这类问题不涉及某个具体实体，而是问**一整块主题**。如果没有社区，系统只能搜到零散的单个实体，无法给出"整体概览"。

**社区检测**就是把**关系紧密的实体自动分群**，每个群就是一个"主题块"，再用 LLM 为每个群生成摘要。

```
合并后的实体图：

  [国家奖学金]──评选──→[学院]
  [国家奖学金]──管理──→[学生处]
  [国家助学金]──管理──→[学生处]        → 这一团关系密切 → 社区 A："奖助学金管理"
  [国家助学金]──资助──→[本科生]

  [学分]──管理──→[教务处]
  [绩点]──评定──→[教务处]             → 这一团关系密切 → 社区 B："学业评价体系"
  [学业绩点]──管理──→[教务处]

  [处分]──违纪──→[学生]
  [警告]──处分──→[学生]               → 这一团关系密切 → 社区 C："纪律处分"
  [记过]──处分──→[学生]
```

#### Leiden 算法做了什么？

Leiden 是目前最主流的社区检测算法（Louvain 的改进版），核心思想是**最大化模块度（modularity）**——让社区内部的边尽可能多，社区之间的边尽可能少。

```python
# leiden.py L23-29
result = self.gds.leiden.write(
    self.G,
    writeProperty="communities",         # 结果写入 communities 属性
    includeIntermediateCommunities=True,  # 保留多层级社区
    relationshipWeightProperty="weight",  # 用关系权重
    gamma=1.0,       # 分辨率参数：越大社区越小越多
    tolerance=0.0001, # 收敛精度
    maxLevels=10      # 最多迭代层数
)
```

**关键参数 `gamma`**：
- `gamma > 1`：倾向于拆成更多小社区（细粒度）
- `gamma < 1`：倾向于合成更少大社区（粗粒度）
- `gamma = 1`：标准模块度优化

#### 多层级社区（Leiden 的核心特性）

`includeIntermediateCommunities=True` 让 Leiden 产生**层级结构**：

```
Level 0（最细）：
  社区 0-0: [国家奖学金, 国家助学金, 学院]
  社区 0-1: [学分, 绩点, 学业绩点, 教务处]
  社区 0-2: [处分, 警告, 记过, 学生]
  社区 0-3: [学生处, 辅导员]

Level 1（聚合）：
  社区 1-0: [社区0-0, 社区0-3]  → "学生事务管理"
  社区 1-1: [社区0-1, 社区0-2]  → "教学与纪律"
```

这让 Global Search 可以在不同粒度层级上回答问题。

#### SLLPA 算法（备选）

SLLPA（Speaker-Listener Label Propagation Algorithm）适合**重叠社区**——即一个实体可以属于多个社区：

```
[学生处] → 既属于"奖助学金管理"社区，也属于"纪律处分"社区
```

Leiden 每个实体只属于一个社区，SLLPA 允许重叠。项目通过 `CommunityDetectorFactory` 工厂类支持两种算法切换。

#### 社区保存到 Neo4j

```python
# leiden.py L102-113
# 1. 为每个社区创建 __Community__ 节点
MERGE (c:__Community__ {id: '0-' + toString(community)})
ON CREATE SET c.level = 0

# 2. 建立 Entity → Community 的 IN_COMMUNITY 关系
MATCH (e) WHERE id(e) = item.entityId
MERGE (e)-[:IN_COMMUNITY]->(c)

# 3. 高层级社区之间也有 IN_COMMUNITY 关系
# 社区0-0 -[:IN_COMMUNITY]-> 社区1-0
```

#### LLM 生成社区摘要

每个社区收集其内部的实体和关系，发给 LLM 生成一段摘要文字：

```python
# summary/leiden.py 核心逻辑
# 1. 查询社区内的所有实体和关系
nodes = [{"id": "国家奖学金", "type": "奖学金类型", "description": "..."},
         {"id": "国家助学金", "type": "奖学金类型", "description": "..."}]
rels  = [{"start": "国家奖学金", "type": "管理", "end": "学生处"}]

# 2. 发给 LLM 生成摘要
summary = llm.invoke("请根据以下实体和关系，总结这个社区的主题：" + ...)
# → "该社区主要涉及学校的奖助学金管理体系，包括国家奖学金和国家助学金的评选、
#    管理和资助流程，由学生处负责统筹管理。"

# 3. 写入 Community 节点
SET c.summary = $summary
```

#### 本项目的社区检测结果

| 指标 | 值 |
|------|------|
| Community 节点数 | **14 个** |
| IN_COMMUNITY 关系 | **105 条** |
| 算法 | Leiden |

#### 社区在搜索中的用途

| 搜索方式 | 用社区做什么 |
|----------|------------|
| **Global Search** | 按社区层级聚合答案：先找相关社区的 summary → 合并多个社区的回答 |
| **Local Search** | 社区 summary 作为补充上下文，增强单实体检索的回答 |

> **面试要点**：社区检测把"平铺的实体图"变成了"有层级的主题结构"。Leiden 的多层级特性让系统既能回答细粒度问题（Level 0 社区），也能回答宏观问题（高 Level 社区），这是 GraphRAG 相比普通 RAG 的核心优势之一。

#### 相关参数

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `GDS_MEMORY_LIMIT` | 6 GB | Leiden 需把实体图加载进 GDS 内存 |
| `GDS_CONCURRENCY` | 4 | 算法并行度 |
| `GDS_TIMEOUT_SECONDS` | 300 | 超时保护，防止大图卡死 |
| `gamma` | 1.0 | 分辨率参数，越大社区越小越细 |
| `maxLevels` | 3~10 | 层级数，根据内存自动调整 |

---

## 五、步骤 3 详解：Chunk 向量索引（`ChunkIndexBuilder`）

> 对应源码：`integrations/build/build_chunk_index.py`
> 依赖步骤2 的实体索引已建好。

### 做了什么

为步骤1 创建的 `__Chunk__` 节点计算 embedding 向量并建索引：

```python
# ChunkIndexManager.create_chunk_index()
# 1. 遍历所有 __Chunk__ 节点
# 2. 对每个 chunk.text 调用 Embedding 模型生成向量
# 3. 将向量写回节点属性：c.embedding = [0.12, -0.34, ...]
# 4. 创建 Vector Index
```

```cypher
CREATE VECTOR INDEX chunk_vector_index
FOR (c:`__Chunk__`) ON c.embedding
OPTIONS {indexConfig: {`vector.dimensions`: 1536, `vector.similarityFunction`: 'cosine'}}
```

### 与步骤2实体索引的区别

| 对比 | 步骤2 实体索引 | 步骤3 Chunk 索引 |
|------|--------------|-----------------|
| 索引目标 | Entity 节点 | Chunk 节点 |
| 用于什么搜索 | Local Search（实体检索）| Naive RAG（文本块检索）|
| 向量来源 | 实体名称+描述 | 原始文本内容 |

> Chunk 索引建好后，`NaiveRagAgent` 的向量检索才能工作——它直接搜最相似的文本块，不经过实体和图结构。

### 相关参数

| 参数 | 作用 |
|------|------|
| `CHUNK_BATCH_SIZE=100` | 每批计算多少个 chunk 的 embedding |
| `EMBEDDING_BATCH_SIZE=64` | Embedding API 单次请求处理的文本数 |

### 踩坑记录：Embedding 模型的 token 限制

**现象**：步骤3 运行时大量报错

```
Error code: 413 - {'code': 20042, 'message': 'input must have less than 512 tokens'}
```

**原因**：硅基流动上的 `BAAI/bge-large-zh-v1.5` 模型最大只接受 **512 tokens** 输入。项目 `CHUNK_SIZE=500` 是 HanLP 分词后的 token 数，但 BGE 模型有自己的 tokenizer，中文一个 HanLP token 可能被 BGE 切成多个 sub-word，导致实际输入超过 512。

**后果**：超长 chunk 的 embedding 计算失败，代码 fallback 写入了零向量 `[0.0]*1536`，这些 chunk 在 Naive RAG 向量检索时**永远不会被召回**。

**解决方案**：换用 `BAAI/bge-m3`（支持 8192 tokens，同样免费）

```bash
# .env 修改
OPENAI_EMBEDDINGS_MODEL = 'BAAI/bge-m3'   # 原来是 'BAAI/bge-large-zh-v1.5'
```

**注意**：只需重跑步骤3，不需要全量重建。但 `build_chunk_index.py` 内部会跳过已有 embedding 的 chunk，所以需要先手动清除旧的 embedding：

```bash
# 1. 清除旧 embedding
python -c "
from graphrag_agent.config.neo4jdb import get_db_manager
db = get_db_manager()
db.graph.query('MATCH (c:\`__Chunk__\`) REMOVE c.embedding')
print('已清除')
"

# 2. 重新计算（直接调用底层，避免 shutup.please() 吞掉输出）
python -c "
from graphrag_agent.graph.indexing.chunk_indexer import ChunkIndexManager
mgr = ChunkIndexManager()
mgr.create_chunk_index()
print('完成')
"
```

| 对比 | bge-large-zh-v1.5 | bge-m3 |
|------|--------------------|--------|
| 最大输入 | 512 tokens | **8192 tokens** |
| 构建耗时 | 25.93 秒（含大量报错重试） | **5.63 秒** |
| 512 错误 | 大量 | **零** |
| 206 个 chunk 全部成功 | 部分失败 | **全部成功** |

> **面试要点**：这个问题体现了 Embedding 模型选型时容易忽略的细节——模型的 max_input_length 必须大于 CHUNK_SIZE 在该模型 tokenizer 下的实际 token 数，而不仅是 HanLP 的 token 数。两个 tokenizer 的计数方式不同，这是一个容易踩的坑。

---

## 六、环境搭建命令

```bash
# 创建虚拟环境
conda create -n graphrag python==3.10
conda activate graphrag

# 安装依赖
pip install -r requirements.txt
pip install -e .

# 启动 Neo4j（docker）
cd /home/wkt/project/graph-rag-agent
docker compose up -d

# 配置环境变量
cp .env.example .env
# 编辑 .env，填入 API Key、Neo4j 密码等

# 执行全量图构建
python graphrag_agent/integrations/build/main.py

# 查看 Neo4j 图
# 浏览器打开 http://localhost:7474
# 执行 MATCH (n) RETURN n LIMIT 50
```

---

## 6.1 构建结果验证（Neo4j Browser）

打开 `http://localhost:7474`，用户名 `neo4j`，密码 `12345678`。

### 节点统计

| 节点类型 | 数量 | 说明 |
|----------|------|------|
| `__Chunk__` | 206 | 文本块 |
| `__Entity__` | 105 | 实体（含管理规定31、部门25、学生类型19 等） |
| `__Document__` | 15 | 文档节点 |
| `__Community__` | 14 | 社区节点 |

### 关系统计

| 关系 | 数量 | 说明 |
|------|------|------|
| `MENTIONS` | 481 | Chunk 提到了哪些实体 |
| `PART_OF` | 206 | Chunk 归属文档 |
| `管理` | 195 | 实体间最多的领域关系 |
| `NEXT_CHUNK` | 191 | Chunk 顺序链 |
| `IN_COMMUNITY` | 105 | 实体归属社区 |
| `权利义务`/`评选`/`申请` 等 | 多条 | 其他领域关系 |

### 常用查询命令

```cypher
-- 统计各类节点
MATCH (n) RETURN labels(n) AS type, count(n) AS count ORDER BY count DESC

-- 查看文档和第一个 chunk
MATCH (d:__Document__)-[:FIRST_CHUNK]->(c:__Chunk__) RETURN d, c LIMIT 10

-- 查看实体间关系
MATCH (e1:__Entity__)-[r]->(e2:__Entity__) RETURN e1, r, e2 LIMIT 50

-- 查看社区（注意标签是带双下划线的 __Community__）
MATCH (e:__Entity__)-[:IN_COMMUNITY]->(c:__Community__) RETURN e, c LIMIT 30
```

> ⚠️ **易错点**：社区节点的标签是 `__Community__`（带双下划线），不是 `Community`。用错标签会返回空结果。

---

## 七、自测问题

**Q1：图构建三步骤为什么必须按顺序执行？**

> 步骤1 建立实体节点（Entity）及其向量；步骤2 对实体向量建索引（向量搜索依赖此索引），再做社区检测（依赖图中已有实体节点）；步骤3 建 Chunk 向量索引时需要与实体节点建联系（依赖步骤2 的实体索引）。任何一步缺失，后续步骤会报"索引不存在"或"节点不存在"错误。

**Q2：text_chunker 为什么用 HanLP 分词而不是直接按字数切分？**

> 直接按字数切分会在词中间截断（如"奖学金"被截成"奖学"+"金"），造成语义破碎。HanLP 基于深度学习分词，能在词边界处切分，更好地保留语义完整性。再结合句子边界对齐，确保 chunk 以完整句子开头和结尾。

**Q3：`file_registry.json` 保存了什么，在增量更新中如何工作？**

> 保存每个已处理文件的路径和 SHA256 hash。增量时遍历 `files/` 目录，对比 hash：新文件→全量处理并创建新节点；修改文件→删除旧节点+重新提取；删除文件→从图中移除对应节点和关系。不用重新处理未变更文件，大幅节省 LLM API 调用费用。

## 八、图构建涉及的代码文件索引

按图构建流程顺序排列，路径均相对于 `graphrag_agent/`。

### 配置层

| 文件 | 核心内容 |
|------|---------|
| `config/settings.py` | 所有参数定义（阈值、批量大小、API 配置等） |
| `config/prompts/graph_prompts.py` | 实体提取、实体合并、社区摘要的 Prompt 模板 |
| `config/neo4jdb.py` | Neo4j 数据库连接配置 |
| `.env` | API Key、模型名称、Neo4j 密码等敏感配置 |

### 入口 & 编排层

| 文件 | 核心类/函数 | 职责 |
|------|-----------|------|
| `integrations/build/main.py` | `KnowledgeGraphProcessor` | 总入口，按顺序调用步骤1→2→3 |
| `integrations/build/build_graph.py` | `KnowledgeGraphBuilder` | 步骤1 编排：文件读取→分块→提取→写入 |
| `integrations/build/build_index_and_community.py` | `IndexCommunityBuilder` | 步骤2 编排：索引→相似检测→合并→消歧→社区 |
| `integrations/build/build_chunk_index.py` | `ChunkIndexBuilder` | 步骤3 编排：Chunk embedding + 向量索引 |

### 步骤1：构建基础图

| 文件 | 核心类 | 职责 |
|------|-------|------|
| `pipelines/ingestion/file_reader.py` | `FileReader` | 读取 PDF/TXT 等文件，提取文本 |
| `pipelines/ingestion/text_chunker.py` | `ChineseTextChunker` | HanLP 中文分词 + 滑动窗口分块 |
| `graph/structure/struct_builder.py` | `GraphStructureBuilder` | 创建 Document/Chunk 节点和链式关系 |
| `graph/extraction/entity_extractor.py` | `EntityRelationExtractor` | LLM 提取实体和关系（并行/批处理） |
| `graph/extraction/graph_writer.py` | `GraphWriter` | 将实体和关系写入 Neo4j（MENTIONS 两阶段写入） |
| `graph/core/graph_connection.py` | `ConnectionManager` | Neo4j 连接池管理 |
| `graph/core/utils.py` | `timer`, `get_performance_stats` | 性能统计工具函数 |

### 步骤2-4.1：实体向量索引

| 文件 | 核心类 | 职责 |
|------|-------|------|
| `graph/indexing/entity_indexer.py` | `EntityIndexManager` | 为 Entity 生成 embedding + 创建 vector index |
| `graph/indexing/embedding_manager.py` | `EmbeddingManager` | Embedding 模型调用和批量处理 |

### 步骤2-4.2~4.3：相似实体检测与合并

| 文件 | 核心类 | 职责 |
|------|-------|------|
| `graph/processing/similar_entity.py` | `SimilarEntityDetector` | GDS 投影 → KNN → WCC → 候选重复组 |
| `graph/processing/entity_merger.py` | `EntityMerger` | LLM 确认 + `mergeNodes` 合并 + 清理重复关系 |

### 步骤2-4.4：实体消歧与对齐

| 文件 | 核心类 | 职责 |
|------|-------|------|
| `graph/processing/entity_quality.py` | `EntityQualityProcessor` | 编排：先消歧后对齐 |
| `graph/processing/entity_disambiguation.py` | `EntityDisambiguator` | WCC 组内选 canonical（含三阶段管道预留） |
| `graph/processing/entity_alignment.py` | `EntityAligner` | 按 canonical 分组 → Jaccard 冲突检测 → 合并 |

### 步骤2-4.5：社区检测

| 文件 | 核心类 | 职责 |
|------|-------|------|
| `community/detector/__init__.py` | `CommunityDetectorFactory` | 工厂类，支持 Leiden/SLLPA 切换 |
| `community/detector/base.py` | `BaseCommunityDetector` | 社区检测基类（投影→检测→保存流程） |
| `community/detector/leiden.py` | `LeidenDetector` | Leiden 算法实现（多层级社区） |
| `community/detector/sllpa.py` | `SLLPADetector` | SLLPA 算法实现（重叠社区） |
| `community/detector/projections.py` | `GraphProjectionMixin` | GDS 图投影的通用逻辑 |
| `community/summary/base.py` | `BaseSummarizer` | 社区摘要生成基类 |
| `community/summary/leiden.py` | `LeidenSummarizer` | 收集社区内实体+关系 → LLM 生成 summary |

### 步骤3：Chunk 向量索引

| 文件 | 核心类 | 职责 |
|------|-------|------|
| `graph/indexing/chunk_indexer.py` | `ChunkIndexManager` | 为 Chunk 生成 embedding + 创建 chunk_vector_index |

### 增量更新

| 文件 | 核心类 | 职责 |
|------|-------|------|
| `integrations/build/incremental/file_change_manager.py` | `FileChangeManager` | SHA256 哈希比对，检测文件增删改 |
| `integrations/build/incremental_graph_builder.py` | `IncrementalGraphBuilder` | 增量图构建编排（只处理变更文件） |

---

## 九、Day 2 打卡

- [x] 理解三层配置体系（.env / settings.py / 服务层）
- [x] 能解释图构建三步骤及其顺序依赖关系
- [x] 理解 ChineseTextChunker 的分块策略（HanLP + 滑动窗口 + 句子边界）
- [x] 能描述 Neo4j 中的节点类型和关系结构
- [x] 理解 `file_registry.json` 的增量更新机制

---

## 九、自测问题详细参考答案

### Q1：图构建三步骤为什么必须按顺序执行？

步骤之间存在**严格的数据依赖链**：

```
步骤1（KnowledgeGraphBuilder）
  → 读取文档 → 分 Chunk → LLM 提取实体和关系
  → 在 Neo4j 中创建 __Document__、__Chunk__、__Entity__ 节点
  → 建立 FIRST_CHUNK、NEXT_CHUNK、PART_OF、MENTIONS 关系
  → ⚠️ 此时 Entity 节点存在，但没有 embedding 向量

步骤2（IndexCommunityBuilder）
  → 4.1 为 Entity 生成 embedding + 建向量索引（依赖步骤1的 Entity 节点存在）
  → 4.2 KNN+WCC 检测相似实体（依赖 4.1 的 embedding）
  → 4.3 LLM 确认 + mergeNodes 合并（依赖 4.2 的候选组）
  → 4.4 消歧+对齐（依赖 4.2 的 WCC 分组）
  → 4.5 Leiden 社区检测（依赖合并后的干净实体图）
  → 创建 __Community__ 节点 + IN_COMMUNITY 关系 + LLM 摘要

步骤3（ChunkIndexBuilder）
  → 为 __Chunk__ 节点生成 embedding + 建 chunk_vector_index
  → 依赖步骤1的 Chunk 节点存在
  → 逻辑上放在最后，因为步骤2 的实体合并可能影响 Chunk 的 MENTIONS 关系指向
```

**如果跳过步骤1 直接跑步骤2**：图里没有 Entity 节点 → GDS 投影时节点数=0 → 社区检测无法运行。
**如果跳过步骤2 直接跑步骤3**：步骤3 本身能运行（只依赖 Chunk 节点），但搜索时 Entity 没有向量索引、没有社区结构，Local Search 和 Global Search 都无法正常工作。

---

### Q2：text_chunker 为什么用 HanLP 分词而不是直接按字数切分？

**直接按字数切分的问题**：  
中文词语是多字组合的语义单元，按字数切割会拆词。例如 `奖学金` 被截断为 `奖学` + `金`，后续向量检索时 `奖学` 无法代表"奖学金"这个概念，召回率严重下降。

**HanLP 分词的优势**：  
HanLP 基于深度学习（ELECTRA 模型），在词边界处切分，保证 `奖学金`、`处分委员会`、`绩点` 等专有词汇不被拆断。

**滑动窗口 + 句子边界对齐** 进一步保证：
- 每个 chunk 在 `。！？` 处结束，语义上是完整的句子
- 相邻 chunk 重叠 100 token，防止跨 chunk 的语义关系断裂
- 大文本预切分防止 HanLP 因文本过长崩溃（max_text_length = 500000 字）

---

### Q3：`file_registry.json` 保存了什么，在增量更新中如何工作？

**存储内容**（`file_change_manager.py`）：每个已处理文件的路径、SHA256 哈希、大小、修改时间：

```json
{
  "学生手册.pdf": {
    "hash": "a3b8d1b6...",
    "size": 1048576,
    "last_modified": 1698796800.0,
    "last_scanned": 1698796900.0,
    "last_processed": "2025-10-01T12:00:00",
    "processing_history": [{"timestamp": "...", "nodes_created": 105}]
  }
}
```

**增量更新核心代码**（`detect_changes()` 方法）：

```python
# file_change_manager.py L109-139
def detect_changes(self):
    current_files = self._scan_current_files()  # 扫描目录，计算每个文件的 SHA256

    for file_path, file_info in current_files.items():
        if file_path not in self.registry:      # ① registry 中无此路径
            added_files.append(file_path)        #   → 新文件
        elif file_info["hash"] != self.registry[file_path]["hash"]:  # ② hash 不同
            modified_files.append(file_path)     #   → 修改过的文件

    for file_path in self.registry:
        if file_path not in current_files:       # ③ 文件已被删除
            deleted_files.append(file_path)

    return {"added": [...], "modified": [...], "deleted": [...]}
```

**四种文件状态的处理**：

| 文件状态 | 检测方式 | 处理动作 |
|----------|---------|---------|
| 新文件 | registry 中无此路径 | 全量提取 + 写入 Neo4j + 记录 hash |
| 修改文件 | 路径存在但 hash 不同 | 删除旧节点 + 重新提取 + 更新 hash |
| 删除文件 | registry 有记录但文件不存在 | 从 Neo4j 删除对应 Document/Chunk/Entity |
| 未变更文件 | hash 完全相同 | **跳过**，不调用任何 LLM |

**为什么重要**：LLM API 调用是图构建成本最高的部分（按 token 计费）。文件不变就跳过，可节省 90%+ 的 API 费用和时间。第二次运行图构建时，只有新增或修改的文件才会触发 LLM 提取。

---

### Q4：实体提取用了哪两种性能优化策略，各自解决什么问题？

| 策略 | 适用场景 | 解决的瓶颈 | 核心机制 |
|------|---------|-----------|---------|
| **并行模式** | ≤100 chunks | 串行等待时间 | ThreadPoolExecutor，MAX_WORKERS 个线程同时调用 LLM |
| **批处理模式** | >100 chunks | API 调用次数/限流 | 多个 chunk 拼成一次请求，减少 80% 网络往返 |

两者都叠加了**磁盘缓存**（SHA256 → pickle 文件），中途中断重跑时命中缓存直接跳过 LLM 调用，日志里的"缓存命中率"就是这个系统的汇报。

---

### Q5：Neo4j 图中有哪些节点类型和关系边，各在哪个步骤创建？

**节点**：

| 节点标签 | 创建时机 | 关键属性 |
|----------|---------|---------|
| `__Document__` | 步骤1-图结构构建 | fileName, uri, domain |
| `__Chunk__` | 步骤1-图结构构建 | id(SHA256), text, position, tokens |
| `Entity` | 步骤1-实体写入 | id, type, description |
| `Community` | 步骤2-社区检测 | id, level, summary |

**关系边**：

| 关系 | 方向 | 创建时机 |
|------|------|---------|
| `FIRST_CHUNK` | Document → Chunk | 步骤1 |
| `PART_OF` | Chunk → Document | 步骤1 |
| `NEXT_CHUNK` | Chunk → Chunk | 步骤1 |
| `MENTIONS` | Chunk → Entity | 步骤1（两阶段提交） |
| `[申请/评选/…]` | Entity → Entity | 步骤1 |
| `IN_COMMUNITY` | Entity → Community | 步骤2 |

> **MENTIONS 两阶段提交**：LangChain 的 `add_graph_documents` 先把实体挂在临时 `Document` 节点，再通过 `merge_chunk_relationships()` 中的 Cypher 迁移到正式 `__Chunk__` 节点并删除临时节点，保证图结构一致性。

