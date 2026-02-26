# Day 2 延伸答疑

> 本文记录学习 `day2_note.md` 过程中产生的追问与解答。
> 每条 QA 标注对应的原文位置，方便后续查阅和补充。

---

## QA-1：批处理模式 vs 并行模式的详细实现与原理

**出处**：`day2_note.md` § 四、实体提取机制 → 并行批处理（性能优化）（约 L150–L159）

```python
# day2_note.md 中的原始代码片段
if total_chunks > 100:
    # 大数据集：批处理模式
    entity_extractor.process_chunks_batch(...)
else:
    # 小数据集：并行模式
    entity_extractor.process_chunks(...)
```

---

### 问题一：两种模式具体是怎么实现的？

**并行模式（`process_chunks`，小数据集 ≤100 chunks）**

对应源码：`graphrag_agent/graph/extraction/entity_extractor.py` → `process_chunks()`

```python
# 只对「未缓存」的 chunk 创建并发任务
with concurrent.futures.ThreadPoolExecutor(max_workers=self.max_workers) as executor:
    future_to_chunk = {
        executor.submit(self._process_single_chunk, ''.join(chunks[idx])): idx
        for idx in non_cached_indices   # ← 跳过已缓存的 chunk
    }
# 哪个任务先完成先处理（不用等其他的）
for future in concurrent.futures.as_completed(future_to_chunk):
    result = future.result()
```

效果：MAX_WORKERS=4 时，4 个线程同时各调用一次 LLM，总时间压缩至约 1/4。

```
串行：[C1]─[C2]─[C3]─[C4]─[C5]─[C6]─[C7]─[C8]   8 × 单次耗时
并行：[C1][C2][C3][C4]
                   [C5][C6][C7][C8]               约 2 × 单次耗时
```

---

**批处理模式（`process_chunks_batch`，大数据集 >100 chunks）**

对应源码：`graphrag_agent/graph/extraction/entity_extractor.py` → `process_chunks_batch()`

```python
# 动态计算每批放多少个 chunk（防止超出 LLM context window）
dynamic_batch_size = max(1, min(self.batch_size, int(10000 / (avg_chunk_size + 1))))

# 把 N 个 chunk 用分隔线拼成一个大文本，只发一次 API 请求
batch_text = f"\n{'─'*50}\n".join(batch_inputs)
batch_response = self.chain.invoke({..., "input_text": batch_text})

# 用同样分隔线把响应还原成 N 个结果
parts = batch_content.split(f"\n{'─'*50}\n")
```

效果：100 个 chunk、batch_size=5 → 只需 20 次 API 调用，减少 80% 网络往返。

---

### 问题二：大数据集用并行不是也能压缩时间？小数据集用批处理不是也能减少请求？为什么偏偏反着用？

**核心原因：两种场景的瓶颈根本不同。**

**小数据集（≤100 chunks）的瓶颈：等待延迟**

- 串行等待是主要耗时，并行把等待时间折叠，效果直接。
- 批处理在此**收益有限**（10个chunk只有2批），反而引入解析风险（见下）。

**大数据集（>100 chunks）的瓶颈：API 限流（RPM）**

大多数 LLM API 有**RPM（Requests Per Minute）上限**，例如 60 次/分钟。

```
大数据集用并行模式（300 chunks，MAX_WORKERS=4）：
  → 4 线程高速消耗 API 配额，快速触发 429 Too Many Requests
  → 重试逻辑启动，实际吞吐比串行更差

大数据集用批处理模式（300 chunks，batch_size=5）：
  → 只需 60 次 API 调用（比并行的 300 次少 80%）
  → 调用间隔自然存在（处理+解析耗时），绕开限流
```

**本质**：并行让你更快地「烧完」API 配额；批处理用更少的请求完成同等工作。

---

**批处理的致命缺点——LLM 解析失败降级**

LLM 不保证严格按分隔线返回 N 段结果，解析失败时有降级逻辑：

```python
# 对应源码 entity_extractor.py L282–L293
if len(batch_results) != len(batch_chunks):
    # ⚠️ 整批退化为逐个单独调用
    for chunk in batch_chunks:
        individual_result = self._process_single_chunk(''.join(chunk))
```

| | 并行模式失败代价 | 批处理模式失败代价 |
|--|----------------|-----------------|
| 1个chunk失败 | 重试这 1 个 | 整批退化（1次失败 + N次单独调用）|
| 小数据集（10 chunks）| 影响可控 | 可能翻倍调用次数 |
| 大数据集（300 chunks）| 限流导致大量重试 | 单批失败仍整体可控 |

这就是小数据集不用批处理的根本原因：**失败代价相对工作量太高**。

---

**理论最优：并行批处理**

多线程 × 每线程处理多个 chunk，代码中 `stream_process_large_files()` 有类似雏形：

```python
# 超大单文件：并发调用 + 完成一个 chunk 立刻写入 Neo4j（流式）
with concurrent.futures.ThreadPoolExecutor(max_workers=self.max_workers) as executor:
    for chunk_data in chunks_with_hash:
        future = executor.submit(self._process_single_chunk, chunk_text)
```

100 chunk 的阈值是**经验值**，不是严格数学推导。

---

**总结对比**

| 维度 | 并行模式 | 批处理模式 |
|------|---------|-----------|
| 适用规模 | ≤100 chunks | >100 chunks |
| 解决的瓶颈 | 串行等待延迟 | API 调用次数/限流 |
| API 调用次数 | N 次 | N/batch_size 次 |
| 解析复杂度 | 低（每次单独解析）| 高（需切分 LLM 响应）|
| 失败退化代价 | 重试单个 chunk | 整批退化为单个 |
| 限流风险 | 大数据集下高 | 低 |

**面试话术**：

> 「实体提取根据数据规模选用了两套策略。小数据集的瓶颈是串行等待，用 ThreadPoolExecutor 4线程并发，时间压缩到约 1/4；大数据集的瓶颈是 API 限流，并发太快会触发 RPM 上限，所以改用批处理，把 5 个 chunk 合并成一次请求，调用次数降低 80%。批处理的代价是 LLM 响应解析可能失败，代码里有完整降级逻辑。理论上最优是并行批处理，`stream_process_large_files` 对超大文件已有类似实现。」

---

---

## QA-2：Neo4j 图结构详细解析

**出处**：`day2_note.md` § 五、Neo4j 图结构（构建结果）（约 L163–L176）

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

### 节点 1：`__Document__`（文档节点）

**创建时机**：图结构构建阶段，对应源码 `graph/structure/struct_builder.py → create_document()`

```cypher
MERGE(d:`__Document__` {fileName: $file_name})
SET d.type=$type, d.uri=$uri, d.domain=$domain
```

| 属性 | 内容 | 示例 |
|------|------|------|
| `fileName` | 文件名（主键） | `学生手册.pdf` |
| `type` | 文档类型 | `local` |
| `uri` | 原始文件路径 | `/home/.../files/` |
| `domain` | 知识库主题 | `华东理工大学学生管理` |

---

### 节点 2：`__Chunk__`（文本块节点）

**创建时机**：`create_relation_between_chunks()`，每个文本块建一个节点

```cypher
MERGE (c:`__Chunk__` {id: data.id})
SET c.text = data.pg_content,
    c.position = data.position,        -- 第几块（从1开始）
    c.length = data.length,            -- 字符长度
    c.fileName = data.f_name,          -- 归属文件
    c.content_offset = data.content_offset,  -- 在原文中的起始偏移量
    c.tokens = data.tokens             -- token 数量
```

**id 的生成方式**：
```python
current_chunk_id = generate_hash(page_content)  # SHA256(文本内容)
```
> 内容寻址设计：相同文本永远得到相同 id，**天然去重**——增量更新时同一段文字不会重复写入。

---

### 节点 3：`Entity`（实体节点）

**创建时机**：`GraphWriter.process_and_write_graph_documents()` 解析 LLM 输出后写入

LLM 输出的原始格式（项目自定义格式）：
```
("entity" : "国家奖学金" : "奖学金类型" : "由国家设立的最高等级奖学金")
("relationship" : "学生" : "国家奖学金" : "申请" : "学生可以申请国家奖学金" : 0.9)
```

解析代码（`graph/extraction/graph_writer.py`）：
```python
node_pattern = re.compile(r'\("entity" : "(.+?)" : "(.+?)" : "(.+?)"\)')
#                                        实体名      实体类型   描述

relationship_pattern = re.compile(r'\("relationship" : "(.+?)" : "(.+?)" : "(.+?)" : "(.+?)" : (.+?)\)')
#                                                      源实体    目标实体  关系类型   描述       权重
```

解析后构造写入 Neo4j：
```python
Node(id="国家奖学金", type="奖学金类型", properties={"description": "..."})
Relationship(source=学生节点, target=国家奖学金节点, type="申请", properties={"weight": 0.9})
```

---

### 节点 4：`Community`（社区节点）

**创建时机**：步骤 2（`IndexCommunityBuilder`），由 Leiden/SLLPA 算法计算后生成，必须在实体节点建好之后才能运行。

---

### 6 种关系边的来龙去脉

| 关系边 | 方向 | 含义 | 创建时机 |
|--------|------|------|---------|
| `FIRST_CHUNK` | Document → Chunk | 文档的第一个块 | 步骤1图结构构建 |
| `PART_OF` | Chunk → Document | 每块归属哪个文档 | 步骤1图结构构建 |
| `NEXT_CHUNK` | Chunk → Chunk | 相邻块的顺序链 | 步骤1图结构构建 |
| `MENTIONS` | Chunk → Entity | 这个块提到了哪些实体 | 步骤1实体写入后合并 |
| `[申请/评选/违纪...]` | Entity → Entity | 实体间的语义关系 | 步骤1实体写入 |
| `IN_COMMUNITY` | Entity → Community | 实体归属哪个社区 | 步骤2社区检测 |

> ⚠️ 笔记原文的 `CONTAINS` 并非代码里的真实关系名，**实际是 `PART_OF`（反向）和 `FIRST_CHUNK`**。

---

### MENTIONS 关系的两阶段提交

#### 背景：LangChain 的 `add_graph_documents` 做了什么

`GraphWriter` 调用的是 LangChain 封装好的 API：

```python
self.graph.add_graph_documents(
    [graph_document],
    baseEntityLabel=True,
    include_source=True   # ← 关键参数：把"这段文字来自哪里"也存进图里
)
```

`include_source=True` 会让 LangChain **自动**在 Neo4j 里为这段文字创建一个 `Document` 节点，然后用 `MENTIONS` 把实体挂到它下面：

```
LangChain 自动创建的结构：

Document（LangChain 自动创建的临时节点）
  chunk_id: "abc123"
    │ MENTIONS
    │ MENTIONS
    ▼
Entity("国家奖学金")  Entity("绩点")  Entity("学生")
```

#### 问题：它和项目自建的 `__Chunk__` 是两个不同节点！

在写实体之前，项目已经建好了自己的 `__Chunk__` 节点：

```
第1步前的图：

__Document__("学生手册.pdf")
    │ PART_OF（反向）
    │
__Chunk__("abc123")    ← 项目自己建的，有 position/tokens 等完整属性
  text: "国家奖学金申请须绩点..."
```

`add_graph_documents` 执行后，图变成：

```
第1步后的图：（出现了冗余节点！）

__Document__("学生手册.pdf")
    │
__Chunk__("abc123")       ← 项目建的 Chunk

Document("abc123")        ← LangChain 自动新建的临时节点，和上面是两个独立节点！
    │ MENTIONS
    │ MENTIONS
    ▼
Entity("国家奖学金")  Entity("学生")    ← 实体挂在错误的临时节点上
```

此时图里有**两个"代表同一段文字"的节点**，而且实体挂在错误的临时节点下，不在真正的 `__Chunk__` 下。

#### 第2步：`merge_chunk_relationships()` 的清理工作

```cypher
-- 找到临时 Document 节点和真正的 __Chunk__ 节点（chunk_id 相同）
MATCH (c:`__Chunk__` {id: data.chunk_id}),
      (d:Document{chunk_id: data.chunk_id})

-- 把仍在临时节点下的 MENTIONS 复制到真正的 __Chunk__ 上
MATCH (d)-[r:MENTIONS]->(e)
MERGE (c)-[newR:MENTIONS]->(e)          ← 正式 Chunk 上建新 MENTIONS
ON CREATE SET newR += properties(r)     ← 保留原关系的属性（weight 等）

-- 删除临时节点（DETACH 会同时删掉它身上的所有关系）
DETACH DELETE d
```

执行后，图恢复整洁：

```
第2步后的图：（临时节点已清除）

__Document__("学生手册.pdf")
    │ PART_OF（反向）
    │
__Chunk__("abc123")    ← MENTIONS 现在正确挂在这里 ✅
    │ MENTIONS
    │ MENTIONS
    ▼
Entity("国家奖学金")  Entity("学生")

（临时 Document 节点已被 DETACH DELETE 删除 🗑️）
```

#### 为什么不直接自己写 Cypher，绕开 LangChain？

| 方案 | 优点 | 缺点 |
|------|------|------|
| 自己写 Cypher 写实体 | 无临时节点，图结构干净 | 需要自己处理实体去重、MERGE 逻辑、关系属性等复杂细节 |
| 借用 `add_graph_documents` + 事后清理 | 复用 LangChain 成熟的解析+写入能力 | 多一步清理步骤，但整体更稳定少 bug |

> **一句话总结**：这是"借用 LangChain 封装能力 + 事后清理副作用"的工程设计——LangChain 负责解析实体、去重、写关系，代价是留了个临时垃圾节点，项目在最后统一清理。



---

### 完整真实图结构（修正版）

```
__Document__（文件节点）
   │ FIRST_CHUNK ─────────────────────────────────┐
   │                                               ▼
   └─◄─ PART_OF ── __Chunk__[1] ──NEXT_CHUNK──► __Chunk__[2] ──NEXT_CHUNK──► ...
                        │                           │
                    MENTIONS                    MENTIONS
                        ▼                           ▼
                   Entity("国家奖学金")         Entity("学生")
                   type="奖学金类型"                │
                   description="..."               │ 申请（weight=0.9）
                        ▲────────────────────────── ┘
                        │
                   IN_COMMUNITY
                        ▼
                   Community（社区节点，步骤2建立）
```

---

### 面试话术

> 「Neo4j 里分四类节点：`__Document__` 代表文件，`__Chunk__` 是分块后的文本片段（SHA256 内容寻址天然去重），`Entity` 是 LLM 提取的实体，`Community` 是社区检测的分群。关系层面，Chunk 通过 `NEXT_CHUNK` 链保持原文顺序，通过 `MENTIONS` 指向实体，实体之间用领域关系类型相连。`MENTIONS` 写入采用两阶段提交——LangChain 先把实体挂在临时节点，再 MERGE 迁移到正式 Chunk 节点并删除临时节点，保证写入完整性。」

---

## QA-3：性能调参为什么是"面试加分项"

**出处**：`day2_note.md` § 七、性能调参（面试加分项）（约 L197–L205）

---

### 加分的本质：证明你真的跑过系统

大多数候选人只会背原理。能说清楚调参的 trade-off，说明你**真的动手跑过、踩过坑、有自己的判断**。

**普通回答**：「用的默认配置，跑通了，效果还不错。」

**加分回答**：「实体提取阶段最慢，因为每个 Chunk 都要调 LLM。把 `MAX_WORKERS` 从 4 调到 8 后吞吐量提升，但触发了 API 限流，最终折中到 6。`CHUNK_SIZE` 调大到 800 时，单次提取的三元组更多，但 LLM 偶尔漏掉尾部实体，所以保留了默认 500。」

---

### 每个参数背后的 Trade-off（面试核心）

| 参数 | 调大收益 | 调大代价 | 面试关键词 |
|------|---------|---------|-----------|
| `MAX_WORKERS` | 并行吞吐量↑ | API 限流风险↑ | **并发与限流的权衡** |
| `BATCH_SIZE` | Neo4j 写入请求次数↓ | 内存↑ | **批处理与资源的权衡** |
| `CHUNK_SIZE` | 上下文完整性↑ | 提取精度↓，API 费用↑ | **粒度与质量的权衡** |
| `CHUNK_OVERLAP` | 语义连续性↑ | 冗余数据增加，存储↑ | **覆盖率与冗余的权衡** |
| `GDS_MEMORY_LIMIT` | 可处理更大图↑ | 机器内存占用↑ | **图规模与资源的权衡** |

---

### 高频追问 & 标准回答

**Q：图构建一次要多久？影响速度的主要因素是什么？**
> 「最耗时的是实体提取，每个 Chunk 都要调一次 LLM。调大 `MAX_WORKERS` 可以并行处理，但受限于 API 的 RPM 上限；调大 `BATCH_SIZE` 可以减少请求次数，但 LLM 单次处理的文本变长后质量可能下降。这两个参数的平衡是核心调参点。」

**Q：CHUNK_SIZE 设多少合适，你们怎么决定这个值的？**
>「默认 500 token，是在"上下文完整性"和"提取精度"之间的权衡。太小（100以下）：LLM 看不到足够上下文，实体关系容易缺失；太大（1000以上）：LLM 容易漏掉文本尾部的实体，且每次调用 token 消耗更多、更贵。我们保留默认值，因为项目的文档是中文规章制度，段落结构清晰，500 token 通常能覆盖一个完整语义段。」

**Q：GDS_MEMORY_LIMIT 是什么，为什么要关注它？**
> 「GDS 是 Neo4j 的图数据科学库，Leiden 社区检测通过 GDS 运行，它会把整个图加载到内存里做计算。默认 6GB，图很大时（实体节点超过5万）会 OOM 或超时。可以调高配置，或换用内存占用更低的 SLLPA 算法。」

---

## QA-4：为什么 CHUNK_SIZE 过大容易漏掉文本尾部的实体？

**出处**：`day2_note.md` § 七、性能调参 → `CHUNK_SIZE` 调大影响（约 L202）

---

### 原因一：LLM 输出被 MAX_TOKENS 硬截断

实体提取要求 LLM 把所有实体逐条输出，当文本块很大时：

```
输入 2000 token 的文本 → 包含约 50 个实体
每个实体输出约 30 token → 需要输出 ~1500 token

但若 MAX_TOKENS = 1000（常见默认值）：
  → LLM 强制截断，只输出前 33 个实体
  → 后 17 个实体物理丢失，不是 LLM"忘了"，而是被截掉了
```

这是最直接、最主要的原因。项目 `settings.py` 中：
```python
LLM_MAX_TOKENS = _get_env_int("MAX_TOKENS", None)  # None = 模型默认值（通常有上限）
```

---

### 原因二："Lost in the Middle"现象（软遗漏）

2023年斯坦福团队论文 [Lost in the Middle](https://arxiv.org/abs/2307.03172) 实验证明：

> **LLM 对输入文本的注意力分布不均匀——开头和结尾关注最多，中间部分最容易被忽略。**

Transformer 的 attention 权重在超长输入时，整体注意力更集中在前半段：

```
短文本（500 token）：注意力分布均匀
  [开头实体] ←→ [中间实体] ←→ [结尾实体]  ← 都能被"看见"

超长文本（2000 token）：
  [开头实体]（强）→ [中间实体]（弱）→ [尾部实体]（最弱）
  即使没有截断，尾部实体也更容易在输出时被遗漏
```

---

### 两个原因的叠加

```
CHUNK_SIZE 过大
  → 文本里实体数量多
  → 需要输出的实体条目多
  ├─→ 超出 MAX_TOKENS → 尾部直接被截断（硬截断）
  └─→ Attention 对尾部文本权重本身就弱 → 即使没截断也更容易漏（软遗漏）
```

---

### 实际调参建议

| 文档类型 | 推荐策略 |
|---------|---------|
| 中文规章制度（段落清晰） | 默认 500 token，段落语义完整 |
| 密集型表格/条款文档 | 调小到 300，保证每块实体密度不过高 |
| 叙事型文档（上下文依赖强） | 适当调大到 700，同时加大 OVERLAP |
| 对召回率要求极高 | 不要单纯加大 CHUNK_SIZE，依靠**实体消歧管道**兜底召回 |

---

### 面试话术

> 「CHUNK_SIZE 不是越大越好。过大有两个问题：一是 LLM 输出的实体列表会超出 MAX_TOKENS 被硬截断，尾部实体物理丢失；二是 Transformer 本身存在"Lost in the Middle"现象，超长输入时对尾部文本的注意力权重更低，即使没截断也容易漏。所以我们保留了 500 token 的默认值，对于可能被漏掉的实体，依靠后续的实体消歧管道做兜底召回，而不是靠增大块大小去覆盖。」

---

## QA-5：为什么需要超长文本处理（`_preprocess_large_text`）？

**出处**：`day2_note.md` § 三、文本分块器深挖 → 超长文本处理（约 L119–L123）

```
- 若文本超过 MAX_TEXT_LENGTH（默认 50万字），先按段落/句子切分成"段"
- 每段再走滑动窗口分块
- 兜底：遇到超长单句，按固定长度强制切分
```

---

### 根本原因：HanLP 是神经网络，无法直接处理超长文本

`ChineseTextChunker` 使用的是：

```python
self.tokenizer = hanlp.load(hanlp.pretrained.tok.COARSE_ELECTRA_SMALL_ZH)
```

ELECTRA 是 **Transformer 架构**的深度学习模型，其内存占用约为 O(n²)（attention 矩阵是 n×n）：

```
n = 1,000 字  → 正常
n = 10,000 字 → 内存增大 100 倍，变慢
n = 500,000 字 → OOM（内存溢出）或几十分钟跑不完
```

`_safe_tokenize` 里已有降级处理：

```python
def _safe_tokenize(self, text: str) -> List[str]:
    if len(text) > self.max_text_length:
        return list(text)   # ← 退化为逐字符切分（最差情况！）
    tokens = self.tokenizer(text)
    return tokens
```

`list(text)` 会把"奖学金"切成 `["奖","学","金"]` 三个字符，**完全丧失语义**，这是必须避免的。

---

### 预处理的目的：让每段都在 HanLP 能正常处理的范围内

```
500,000 字的超大文档
  ↓ _preprocess_large_text()（target_segment_size ≈ 10,000 字）
  → 段落1（8,000字）→ HanLP 正常分词 → 滑动窗口 → chunk1, 2...
  → 段落2（7,500字）→ HanLP 正常分词 → 滑动窗口 → chunk3, 4...
  → 段落3（9,000字）→ HanLP 正常分词 → 滑动窗口 → chunk5, 6...
```

每段都在安全范围内，分词质量得到保证。

---

### 三层兜底策略（优先保留语义）

```python
# 第1层：按段落（\n\n）切   ← 语义最完整
paragraphs = text.split('\n\n')

# 第2层：段落太少就按行（\n）切
if len(paragraphs) < 5:
    paragraphs = text.split('\n')

# 第3层：单个段落本身超长，按句子（。！？）切
# 最后兜底：按固定长度强制切  ← 语义损失最大
for i in range(0, len(sentence), max_size):
    segments.append(sentence[i:i + max_size])
```

---

### 实际项目中会触发吗？

本项目文件（华东理工学生管理文档）一般只有几千到几万字，远低于 50 万字上限，**正常情况下不会触发**。这是面向更大规模文档（整本教材、法律全书）的**防御性设计**，保证项目具备通用性。

---

## QA-6：分词器到底在做什么，它的作用是什么？

**出处**：`day2_note.md` § 三、文本分块器深挖 → 分块策略（约 L107–L117）

---

### 核心问题：chunk 边界应该切在哪里？

**不用分词器，直接按字符数切**（chunk_size=20字符）：

```
原文：国家奖学金申请须满足以下条件：学业绩点不低于3.5，且无违纪记录。

chunk1: "国家奖学金申请须满足以下条件：学业绩"   ← ⚠️ "绩点"被截断！
chunk2: "点不低于3.5，且无违纪记录。"
```

`绩` 和 `点` 分属两个 chunk，向量化后两个 chunk 都无法正确表达"绩点"这个语义。

---

**用 HanLP 分词后，按 token 数切**：

```python
tokenizer("国家奖学金申请须满足以下条件：学业绩点不低于3.5，且无违纪记录。")

# 输出 token 序列：
["国家", "奖学金", "申请", "须", "满足", "以下", "条件", "：",
 "学业", "绩点", "不", "低于", "3.5", "，", "且", "无", "违纪", "记录", "。"]
```

以 **token（词）为单位**进行滑动窗口（chunk_size=10 tokens）：

```
chunk1: ["国家","奖学金","申请","须","满足","以下","条件","：","学业","绩点"]
        → "国家奖学金申请须满足以下条件：学业绩点"   ✅ 所有词保持完整
```

---

### 分词器的本质作用

```
原始文本（字符流）
    ↓ HanLP 分词（ELECTRA 深度学习模型）
词序列（语义单元流）：["国家", "奖学金", "绩点", "违纪", ...]
    ↓ 滑动窗口（按 token 数计数，在句子边界对齐）
Chunk 列表（每个 chunk 以词边界对齐，语义完整）
```

| | 按字符切 | 按 token（词）切 |
|--|---------|----------------|
| 词完整性 | ❌ 经常被截断 | ✅ 词不会被切断 |
| chunk_size 单位 | 字符数 | 词数（更接近语义单位） |
| 向量化质量 | 低（残词破坏语义） | 高（词语完整，语义清晰） |
| 检索召回率 | 低 | 高 |

---

### 为什么选 HanLP 而不是 jieba？

```
jieba（规则/统计）：  国家 | 奖学 | 金         ← ❌ "奖学金"被错误拆分
HanLP（ELECTRA）：    国家 | 奖学金           ← ✅ 正确识别专有词汇
```

学生管理文档的领域词汇：`违纪处分`、`勤工助学`、`学业绩点`、`申诉委员会`——这类词在 jieba 中容易被拆开，HanLP 基于深度学习对专有词汇识别更准确。

---

### 面试话术

> 「分词器把中文字符流转换成词序列，让 chunk 边界以词为最小单位对齐，而不是在词中间截断。这直接影响后续向量化的质量——残缺的词会让向量无法正确表达语义，导致检索召回率下降。项目选用 HanLP 而非 jieba，是因为学生管理文档有大量领域专有词汇，HanLP 的深度学习模型识别更准确。」

---

## QA-7：settings.py 已经定义了 entity_types，为什么还会出现重复实体？

**出处**：`day2_note.md` § 四、步骤2详解 → 4.2 检测相似实体

---

### 关键区分：entity_types 是"类型"，不是"实example"

```python
# settings.py — 定义的是实体的分类标签
entity_types = ["学生类型", "奖学金类型", "处分类型", "部门", "学生职责", "管理规定"]
```

这告诉 LLM "你应该提取哪些**类别**的实体"，但具体提取出什么**名字**，由 LLM 从每个 chunk 的文本中自行决定。

---

### 重复实体是怎么产生的

LLM 是**逐 chunk 独立调用**的，没有全局视野：

```
Chunk 37：「...获得国家奖学金的学生应品学兼优...」
  → LLM 提取：{id: "国家奖学金", type: "奖学金类型"}

Chunk 102：「...国奖评选须经学院推荐...」
  → LLM 提取：{id: "国奖", type: "奖学金类型"}
```

两次提取的 `type` 完全相同（都是"奖学金类型"），但 `id` 不同。Neo4j 中就产生了两个节点：

```
Node A: {id: "国家奖学金", type: "奖学金类型"}
Node B: {id: "国奖",       type: "奖学金类型"}
```

它们**指向同一个现实事物**，但系统不知道，因为 LLM 处理 Chunk 102 时看不到 Chunk 37 的提取结果。

---

### 三种常见原因

| 原因 | 例子 |
|------|------|
| **简称/别称** | "国家奖学金" vs "国奖"，"学业绩点" vs "GPA" |
| **表述差异** | "学生工作部" vs "学生处" vs "学工部" |
| **LLM 不一致** | 同一概念在不同 chunk 中被 LLM 提取成不同文字 |

---

### 这就是 KNN+WCC 的意义

```
步骤1：LLM 逐 chunk 提取实体 → 产生重复节点（无法避免）
  ↓
步骤2-4.2：KNN+WCC 找出"名字不同但向量相似"的实体组
  ↓
步骤2-4.3：LLM 判断候选组是否真的是同一事物 → 合并节点
```

> 如果不做这一步，"国家奖学金"和"国奖"会是两个独立节点，搜索"国奖"时找不到"国家奖学金"相关的关系和上下文，**图谱的连通性和检索质量都会下降**。

---

### 面试话术

> 「entity_types 约束的是实体的分类维度（如"奖学金类型"），但同一类型下 LLM 从不同 chunk 独立提取的实体实例名称可能不同——因为每个 chunk 是独立送给 LLM 的，LLM 没有全局视野。所以需要后置的 KNN+WCC 管道做实体消重：先用向量相似度（KNN）找候选，再用图连通性（WCC）做传递合并，最后用 LLM 做最终判断。这是一个典型的"先召回后精排"的策略。」

---

## QA-8：4.3 已经合并了实体，为什么 4.4 消歧时 WCC 分组里还有多个实体？

**出处**：`day2_note.md` § 四、步骤2详解 → 4.3 与 4.4 的关系

---

### 核心原因：4.3 的合并有两层过滤，会淘汰一部分候选

```
4.2 KNN+WCC → 所有 Entity 都被分配了 wcc 编号（同组 = 向量相似）

4.3 合并 → 不是直接合并整个 wcc 组，而是：
  ① find_potential_duplicates：文本编辑距离 < 3 才进入候选
  ② LLM 确认：候选组中 LLM 回答"是同一事物"才真正合并

→ 过滤会淘汰一部分，所以同一 wcc 组里可能还有未被合并的实体
```

---

### 举例

假设 WCC 分组 `{wcc=5}` 包含 4 个实体：

```
["奖学金管理办法", "奖学金规定", "奖学金制度", "国家助学金"]
```

它们向量都相似（都和"奖学金"相关），但：

| 4.3 过滤步骤 | 结果 |
|-------------|------|
| ① 文本编辑距离 < 3 | "奖学金管理办法" 和 "奖学金规定"距离=4 → **被过滤** |
| ② LLM 判断 | "奖学金制度" 和 "国家助学金" → LLM 说不是同一事物 → **拒绝合并** |

→ 4.3 结束后，这 4 个实体**一个都没合并**，但 `wcc=5` 属性还在  
→ 4.4 消歧会重新处理这个分组

---

### 4.3 和 4.4 的分工

| | 4.3 合并 | 4.4 消歧+对齐 |
|---|---------|-------------|
| **策略** | 保守（文本距离 + LLM 双重确认） | 宽松（按度数选代表 + Jaccard 冲突检测） |
| **目标** | 只合并"非常确定是同一事物"的实体 | 处理 4.3 剩下的"可能相关"的实体 |
| **风险** | 宁可漏也不错 | 有冲突检测兜底 |

---

### 面试话术

> 「4.3 的合并是精确合并，需要文本编辑距离和 LLM 双重确认，所以会有漏网之鱼。4.4 消歧+对齐是补充清理，用度数选主代表、Jaccard 检测冲突，处理 4.3 遗留的同 WCC 组实体。两步合作是经典的"先精确后召回"策略，在保证准确率的前提下最大化图谱质量。」

---

## QA-9：4.4 对齐中的"冲突检测"和"合并实体 5 步"具体在做什么？

**出处**：`day2_note.md` § 四 → 4.4 实体消歧与对齐 → ② 冲突检测、③ 合并实体

---

### 冲突检测到底在检测什么？

消歧给每组选了 canonical（主代表），但在合并之前要**验证是否真的该合并**——看两个实体在图里的"行为"是否一致，即它们连接的**关系类型**是否相似。

**有冲突的例子**（不该合并）：

```
"学生处" 的关系类型：{管理, 处分, 资助}     ← 管学生纪律和资助
"教务处" 的关系类型：{管理, 教学, 选课}     ← 管教学和课程

交集 = {管理}           → 1 个
并集 = {管理, 处分, 资助, 教学, 选课} → 5 个

Jaccard = 1/5 = 0.2 < 0.5（阈值）
→ ⚠️ 有冲突！名字相似但在图里做的事差太多 → 不能直接合并，让 LLM 判断
```

**无冲突的例子**（可以合并）：

```
"国家奖学金" 的关系类型：{评选, 管理, 申请}
"国奖"       的关系类型：{评选, 管理}

交集 = {评选, 管理}     → 2 个
并集 = {评选, 管理, 申请} → 3 个

Jaccard = 2/3 = 0.67 > 0.5
→ ✅ 无冲突，直接合并
```

> **一句话理解**：Jaccard 就是问「这两个实体在图里扮演的角色像不像？」

---

### 合并实体 5 步的直觉理解

把它想象成**公司合并**——"国奖"要并入"国家奖学金"。

**合并前**：

```
Chunk_37  ──MENTIONS──→ [国家奖学金] ──评选──→ [学院]
                                      ──管理──→ [学生处]

Chunk_102 ──MENTIONS──→ [国奖]        ──申请──→ [本科生]
                                      ──管理──→ [学生处]
```

**5 步逐步执行**：

```
步骤1: 确定目标节点
  → 保留 [国家奖学金]（canonical）

步骤2: 迁移"国奖"的出边
  "国奖"──申请──→[本科生]  →  改为 [国家奖学金]──申请──→[本科生]  ✅ 新增
  "国奖"──管理──→[学生处]  →  [国家奖学金]已有──管理──→[学生处]  ❌ 跳过（去重）

步骤3: 迁移"国奖"的入边
  Chunk_102──MENTIONS──→[国奖]  →  改为 Chunk_102──MENTIONS──→[国家奖学金]  ✅

步骤4: 合并属性
  [国家奖学金].aligned_from = ["国奖"]        ← 记录合并历史
  [国家奖学金].description 保留原有的          ← COALESCE 取非空值

步骤5: 删除"国奖"节点
  DETACH DELETE（同时删除节点和所有残留关系）
```

**合并后**：

```
Chunk_37  ──MENTIONS──→ [国家奖学金] ──评选──→ [学院]
Chunk_102 ──MENTIONS──→ [国家奖学金] ──管理──→ [学生处]
                                      ──申请──→ [本科生]
```

> **关键点**：步骤2 和步骤3 有**去重检查**——如果目标节点已有同类型关系指向同一节点（如两个"管理→学生处"），不重复创建。这是它比 4.3 的 `mergeNodes` 更精细的地方。

---

## QA-10：消歧器的三阶段管道（字符串召回→向量重排→NIL 检测）到底有什么用？

**出处**：`day2_note.md` § 四 → 4.4 消歧 → 三阶段管道

---

### 这个管道不是给"已有实体"用的，是给"新输入"用的

`apply_to_graph()` 处理的是图里**已有的** Entity 节点，直接按度数选 canonical 就够了。

三阶段管道设计的场景是：**一个外部新词进来**（比如用户搜索或新文档），需要判断它对应图里的哪个实体——这叫**实体链接（Entity Linking）**。

---

### 具体场景：用户搜索

```
用户查询："国奖怎么申请？"
  ↓ 提取出 mention = "国奖"
  ↓ 图里有100多个实体，"国奖"到底对应哪个？

图中实体：
  "国家奖学金"、"国家助学金"、"国赛"、"学业绩点"、"学生处" ...
```

三阶段依次处理：

```
阶段1 字符串召回（粗筛）：
  "国奖" vs "国赛"        → Levenshtein=0.67  ← 一个字之差，但意思不同
  "国奖" vs "国家奖学金"   → Levenshtein=0.40  ← 文字差很多，但其实是同一个东西

  → 纯文字匹配会选错！"国赛"比"国家奖学金"拼写更像"国奖"

阶段2 向量重排（精排）：
  "国赛"      → 文本=0.67, 向量=0.3  → 综合分数=0.45  ← 语义完全不同
  "国家奖学金"  → 文本=0.40, 向量=0.97 → 综合分数=0.74  ← 语义一致！

  → 向量权重 0.6 大于文本权重 0.4，所以语义占主导
  → "国家奖学金"排到第一

阶段3 NIL 检测（兜底）：
  最佳候选 0.74 > 0.6 → 不是未知实体
  → 输出 canonical_id = "国家奖学金" ✅
```

**没有这个管道会怎样？**

```
用户搜索"国奖" → 图里没有叫"国奖"的节点 → 搜不到结果 ❌
有了管道      → "国奖" 映射到 "国家奖学金"  → 找到相关信息 ✅
```

---

### 为什么需要三阶段而不是直接向量匹配？

| 方案 | 问题 |
|------|------|
| 只用文本匹配 | "国奖"和"国赛"只差一字但意思不同，会选错 |
| 只用向量匹配 | 需要把 mention 和所有实体算一遍向量相似度，太慢 |
| **先文本召回再向量重排** | 文本粗筛缩小范围（100→5个候选），向量精排保证准确 |

NIL 检测的作用：如果最佳候选的分数都很低，说明图里**真的没有**对应实体，是一个全新概念，不应该强行匹配。

---

### 当前状态

| | `apply_to_graph()`（正在用） | `disambiguate()`（三阶段管道） |
|---|---|---|
| **输入** | 图里已有的 Entity | 外部新词（搜索 query、新文档） |
| **目的** | WCC 组内选代表 | 把新词映射到已有实体 |
| **当前状态** | ✅ 正在使用 | 🔮 预留设计，未启用 |

> **面试话术**：「消歧器内置了完整的 Entity Linking 管道：先用 Levenshtein 做字符串召回控制效率，再用 Embedding 向量做语义重排保证准确性，最后 NIL 检测兜底处理全新实体。虽然当前图构建时用的是简化版（按度数选 canonical），但这个管道为搜索时的查询实体链接和增量文档处理预留了能力。」

<!-- 后续问题追加在此处，格式参考 QA-1 至 QA-10 -->
