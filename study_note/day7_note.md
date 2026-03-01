# Day 7 学习笔记 — 亲手复现：知识图谱构建实战

> 日期：2026-02-28
>
> **阅读前提**：已完成 Day 1-6，理解了图构建流程、Agent架构、搜索策略、质量保证机制和多智能体架构。

---

## 写在前面：Day 7 要解决的核心问题

前6天我们从理论层面深入学习了 GraphRAG 的各个模块，但**纸上得来终觉浅，绝知此事要躬行**。

Day 7 的核心目标是：**真正动手跑通整个知识图谱构建流程，从零到一亲眼见证知识图谱的诞生。**

想象一下这些场景：
- 你看了6天的代码和架构图，但从未真正执行过 `python main.py`
- 你理解了实体提取、消歧、对齐的原理，但不知道实际运行时会遇到什么问题
- 你知道 Neo4j 存储图谱，但从未在 Neo4j Browser 中亲眼看到节点和关系

Day 7 就是要解决这个问题：

```
理论学习（Day 1-6）→ 实战复现（Day 7）→ 深度调优（Day 8）→ 面试准备（Day 9-10）
```

---

## 一、环境配置：从零搭建运行环境

### 1.1 为什么环境配置是第一步？

**核心原则**：工欲善其事，必先利其器。

在开始构建知识图谱之前，我们需要确保以下组件正常运行：
1. **Python 环境**：项目依赖 Python 3.10
2. **Neo4j 数据库**：存储知识图谱的节点和关系
3. **LLM API**：用于实体提取、消歧、摘要生成
4. **Embedding 模型**：用于向量检索和语义缓存

---

### 1.2 Step 1：创建 Python 虚拟环境

```bash
# 1. 创建虚拟环境（使用 conda）
conda create -n graphrag python==3.10
conda activate graphrag

# 2. 安装项目依赖
cd /home/wkt/project/graph-rag-agent
pip install -r requirements.txt

# 3. 以可编辑模式安装项目（重要！）
pip install -e .
```

**为什么要用 `pip install -e .`？**
- `-e` 表示 editable mode（可编辑模式）
- 这样修改代码后不需要重新安装，直接生效
- 项目中的 `from graphrag_agent.xxx import yyy` 才能正常工作

👉 **[高阶追问] 面试官可能会问：“为什么用可编辑模式而不是普通安装，底层有什么区别？” 详见 `day7_note_QA.md` Q1**

---

### 1.3 Step 2：启动 Neo4j 数据库

```bash
# 使用 Docker Compose 启动 Neo4j
cd /home/wkt/project/graph-rag-agent
docker compose up -d

# 查看 Neo4j 是否启动成功
docker ps | grep neo4j
```

**Neo4j 启动后的关键信息**：
- **Web 界面**：http://localhost:7474
- **Bolt 协议地址**：neo4j://localhost:7687
- **默认用户名**：neo4j
- **默认密码**：需要在 `.env` 中配置

**验证 Neo4j 是否正常运行**：
1. 打开浏览器访问 http://localhost:7474
2. 使用用户名 `neo4j` 和你在 `.env` 中设置的密码登录
3. 在查询框中输入 `MATCH (n) RETURN n LIMIT 10` 测试连接

👉 **[高阶追问] 面试官可能会考容错与架构：“Docker Compose 启动 Neo4j 时内部具体发生了什么，端口是怎么映射的？” 详见 `day7_note_QA.md` Q2**

---

### 1.4 Step 3：配置 .env 文件

```bash
# 复制示例配置文件
cp .env.example .env

# 编辑 .env 文件
vim .env  # 或使用你喜欢的编辑器
```

**关键配置项解析**：

```bash
# === LLM 配置（最重要！）===
OPENAI_API_KEY='sk-xxx'  # 你的 API Key
OPENAI_BASE_URL='http://localhost:13000/v1'  # One-API 代理地址
OPENAI_LLM_MODEL='gpt-4o'  # 生成模型
OPENAI_EMBEDDINGS_MODEL='text-embedding-3-large'  # 向量模型

# === Neo4j 配置 ===
NEO4J_URI='neo4j://localhost:7687'
NEO4J_USERNAME='neo4j'
NEO4J_PASSWORD='12345678'  # 修改为你的密码

# === 性能调优参数 ===
MAX_WORKERS=4  # 线程池大小
BATCH_SIZE=100  # 批处理大小
ENTITY_BATCH_SIZE=50  # 实体批处理大小
EMBEDDING_BATCH_SIZE=64  # 向量批处理大小
```

**配置文件的三个关键点**：
1. **OPENAI_BASE_URL**：如果你使用 One-API 或其他代理，需要修改这个地址
2. **NEO4J_PASSWORD**：必须与 Docker Compose 中的密码一致
3. **批处理参数**：根据你的机器性能调整，避免 OOM（内存溢出）

👉 **[高阶追问] 面试官可能会考性能调优：“各个 Batch Size 参数是怎么影响构建性能的？瓶颈究竟在哪一层？” 详见 `day7_note_QA.md` Q3**

---

## 二、准备测试数据：选择合适的文档

### 2.1 项目自带的数据集

项目在 `files/` 目录下已经提供了测试数据：

```bash
ls -lh files/
```

输出示例：
```
2023学生手册.pdf
华东理工大学《大学英语》课程教学实施方案.pdf
华东理工大学关于印发《学生勤工助学管理办法》的通知.pdf
华东理工大学本科生请假申请表.doc
华东理工大学研究生论文答辩程序.doc
```

**这些文档的特点**：
- 领域：高校学生管理规定
- 格式：PDF、DOC、DOCX
- 内容：包含大量实体（奖学金、处分、申请条件等）和关系（申请、评选、违纪等）

---

### 2.2 如何选择测试数据？

**原则 1：从小规模开始**
- 第一次运行建议只用 1-2 个文档
- 验证流程跑通后再增加文档数量

**原则 2：选择结构化程度高的文档**
- PDF 表格、规章制度类文档效果最好
- 避免使用纯图片 PDF（需要 OCR）

**原则 3：控制文档大小**
- 单个文档建议不超过 50 页
- 总字符数不超过 `.env` 中的 `MAX_TEXT_LENGTH`（默认 500000）

---

### 2.3 文档摄取流程预览

```
文档文件（PDF/DOCX/TXT）
    ↓
FileReader（多格式解析）
    ↓
TextChunker（文本分块）
    ↓
EntityExtractor（LLM 提取实体和关系）
    ↓
EntityDisambiguator（实体消歧）
    ↓
EntityAligner（实体对齐）
    ↓
Neo4j 存储
```

---

## 三、执行图构建：三步走策略

### 3.1 图构建的三个阶段

根据 `graphrag_agent/integrations/build/main.py`，完整的图构建流程分为三步：

```python
# 步骤 1：构建基础图谱
KnowledgeGraphBuilder().process()

# 步骤 2：构建实体索引和社区
IndexCommunityBuilder().process()

# 步骤 3：构建 Chunk 索引
ChunkIndexBuilder().process()
```

**为什么这三步是严格串行的“依赖链”（Interview 重点）？**

1. **步骤 1 -> 步骤 2 的依赖（图结构 -> 社区划分）**：
   - 步骤 2 的核心算法（Leiden）是基于**图的拓扑结构（节点连通度）**来划分社区的。如果步骤 1 还没有把实体（Entity）提取出来并在 Neo4j 里连上线（Relationship），步骤 2 面对的就是一个空图，无法进行任何聚类计算。
2. **步骤 2 -> 步骤 3 的依赖（实体向量 -> 文本块挂载）**：
   - 步骤 3 的目的是让原始的文档片段（Chunk）能够关联到图谱里的实体，实现混合检索（Hybrid Search）。这个过程往往不是简单的字符串匹配，而是强依赖步骤 2 中生成的**实体向量索引（Entity Vector Index）**来进行语义级的挂载与关联。如果跳过步骤 2，Chunk 就找不到挂靠的锚点。

👉 **[高阶追问] 面试官可能会问：“如果我非要颠倒 Chunk 索引和实体索引的构建顺序会报错吗？底层的关联逻辑是什么？” 详见 `day7_note_QA.md` Q4**

---

### 3.2 Step 1：构建基础图谱

**执行命令**：
```bash
python graphrag_agent/integrations/build/main.py
```

**这一步做了什么？**

1. **文档读取**：
   - 扫描 `files/` 目录下的所有文档
   - 支持格式：TXT, PDF, MD, DOCX, DOC, CSV, JSON, YAML

2. **文本分块**：
   - 按照 `CHUNK_SIZE=500` 字符切分文档
   - 保留 `CHUNK_OVERLAP=100` 字符的重叠（避免实体被切断）

3. **实体提取**（LLM-based）：
   - 将每个文本块 + Schema 喂给 LLM
   - LLM 返回结构化的三元组：`(实体1, 关系, 实体2)`
   - Schema 定义在 `graphrag_agent/config/settings.py`

4. **实体消歧**（三步管道）：
   - 字符串召回：编辑距离匹配
   - 向量重排：Embedding 相似度
   - NIL 检测：判断是否为新实体

5. **实体对齐**（冲突解决）：
   - 检测同一 `canonical_id` 下的冲突实体
   - 调用 LLM 决定保留哪个实体
   - 合并关系到代表实体

👉 **[高阶追问] 面试官可能会质疑模型设计：“实体消歧的三步管道为什么要这样设计？为什么不直接使用向量相似度？” 详见 `day7_note_QA.md` Q6**

6. **存储到 Neo4j**：
   - 创建节点（Entity）
   - 创建关系（Relationship）
   - 记录文档来源（Document）

**观察日志输出**：
```
[步骤 1] 读取文档: 2023学生手册.pdf
[步骤 2] 文本分块: 共 245 个 chunks
[步骤 3] 实体提取: 提取到 128 个实体, 95 个关系
[步骤 4] 实体消歧: 合并了 23 个重复实体
[步骤 5] 实体对齐: 解决了 5 个冲突
[步骤 6] 存储到 Neo4j: 成功
```

---

### 3.3 Step 2：构建实体索引和社区

**这一步做了什么？**

1. **创建向量索引**：
   - 为所有实体生成 Embedding
   - 在 Neo4j 中创建向量索引（用于语义搜索）

2. **社区检测**：
   - 使用 Leiden 或 SLLPA 算法
   - 将图谱划分为多个社区（Community）
   - 每个社区代表一个主题簇

👉 **[高阶追问] 面试官可能会考图算法：“社区检测的 Leiden 和 SLLPA 算法有什么区别？各自的适用场景是什么？” 详见 `day7_note_QA.md` Q7**

3. **社区摘要生成**：
   - 对每个社区内的实体调用 LLM 生成摘要
   - 摘要用于全局搜索（Global Search）

👉 **[高阶追问] 本质拷问：“为什么全局搜索一定要依赖社区摘要？” 详见 `day7_note_QA.md` Q8**

**观察日志输出**：
```
[步骤 1] 生成实体 Embedding: 128 个实体
[步骤 2] 创建向量索引: 成功
[步骤 3] 社区检测: 检测到 12 个社区
[步骤 4] 社区摘要: 生成了 12 个摘要
```

---

### 3.4 Step 3：构建 Chunk 索引

**这一步做了什么？**

1. **为文本块生成 Embedding**：
   - 对每个 Chunk 调用 Embedding 模型
   - 存储到 Neo4j 的向量索引

2. **关联 Chunk 和 Entity**：
   - 记录每个 Chunk 中提到了哪些实体
   - 用于混合检索（Hybrid Search）

**观察日志输出**：
```
[步骤 1] 生成 Chunk Embedding: 245 个 chunks
[步骤 2] 创建 Chunk 索引: 成功
[步骤 3] 关联 Chunk-Entity: 成功
```

---

## 四、验证图谱：在 Neo4j Browser 中查看

### 4.1 打开 Neo4j Browser

1. 浏览器访问：http://localhost:7474
2. 登录（用户名：neo4j，密码：你在 `.env` 中设置的密码）

---

### 4.2 查看图谱统计信息

**查询 1：统计节点数量**
```cypher
MATCH (n)
RETURN labels(n) AS 节点类型, count(n) AS 数量
```

输出示例：
```
节点类型          数量
["__Entity__"]    105
["__Document__"]  1
["__Community__"] 12
["__Chunk__"]     245
```

**查询 2：统计关系数量**
```cypher
MATCH ()-[r]->()
RETURN type(r) AS 关系类型, count(r) AS 数量
```

输出示例：
```
关系类型          数量
申请             23
评选             15
违纪             8
资助             12
```

---

### 4.3 可视化图谱结构

**查询 3：查看某个实体的邻居**
```cypher
MATCH (e:`__Entity__` {id: "国家奖学金"})-[r]-(neighbor)
RETURN e, r, neighbor
LIMIT 20
```

**查询 4：查看社区结构**
```cypher
MATCH (e:`__Entity__`)-[:IN_COMMUNITY]->(c:`__Community__`)
WHERE c.id = "0"
RETURN e, c
LIMIT 50
```

---

### 4.4 验证实体消歧效果

> 👉 **面试追问：为什么直接查重复节点会查不到数据？** 参考 `day7_note_QA.md` 的 **[Q11：为什么在 Neo4j 中查不到被合并的重复实体？(物理消灭机制)]**。

**查询 5：检查有哪些实体被合并了（查看合并历史）**
```cypher
MATCH (e:`__Entity__`)
WHERE e.aligned_from IS NOT NULL AND size(e.aligned_from) > 0
RETURN e.id AS canonical, e.aligned_from AS merged_entities
```

输出示例：
```
canonical        merged_entities
习近平           ["习近平主席", "习总书记", "国家主席习近平"]
国家奖学金       ["国家奖学金", "国奖"]
```

---

## 五、增量更新：添加新文档

### 5.1 为什么需要增量更新？

**场景**：
- 你已经构建了一个包含 10 个文档的知识图谱
- 现在你想添加第 11 个文档
- 如果重新运行 `main.py`，会重新处理所有文档（浪费时间和 API 调用）

**增量更新的优势**：
- 只处理新增/修改/删除的文档
- 保留已有的图谱结构
- 自动检测文件变更（基于 `file_registry.json`）

---

### 5.2 file_registry.json 的作用

**文件位置**：项目根目录下的 `file_registry.json`

**文件内容示例**：
```json
{
  "files/2023学生手册.pdf": {
    "hash": "a1b2c3d4e5f6...",
    "last_modified": "2026-02-28T10:30:00",
    "status": "processed"
  }
}
```

**工作原理**：
1. 首次构建时，记录每个文件的 MD5 哈希值
2. 增量更新时，对比当前文件的哈希值
3. 如果哈希值不同，说明文件被修改，需要重新处理

👉 **[高阶追问] 面试官可能会问：“增量更新的具体检测逻辑是怎么实现的？遇到删除文件时，图谱怎么保证不留垃圾数据？” 详见 `day7_note_QA.md` Q5（增量检测逻辑） 和 Q9（图谱一致性保护）**

---

### 5.3 执行增量更新（单次模式）

```bash
# 1. 在 files/ 目录下添加新文件
cp ~/new_document.pdf files/

# 2. 执行增量更新
python graphrag_agent/integrations/build/incremental_update.py --once
```

**观察日志输出**：
```
[检测变更] 新增: 1, 修改: 0, 删除: 0
[处理新增] new_document.pdf
[实体提取] 提取到 45 个实体, 32 个关系
[实体消歧] 合并了 8 个重复实体
[更新图谱] 成功
[更新索引] 成功
[社区检测] 检测到 13 个社区（新增 1 个）
```

---

### 5.4 执行增量更新（守护进程模式）

```bash
# 启动后台守护进程，每 5 分钟检测一次文件变更
python graphrag_agent/integrations/build/incremental_update.py --daemon --interval 300
```

**适用场景**：
- 生产环境中，文档会持续更新
- 希望图谱自动保持最新状态

**停止守护进程**：
```bash
# 按 Ctrl+C 或发送 SIGTERM 信号
kill -TERM <进程ID>
```

---

## 六、常见问题与调试技巧

👉 **如果你在复现过程中遇到了奇怪的报错，不知道怎么排查，详见 `day7_note_QA.md` Q10: 如何调试图构建过程中的错误？**

### 6.1 问题 1：LLM API 调用失败

**错误信息**：
```
OpenAI API error: Connection timeout
```

**排查步骤**：
1. 检查 `.env` 中的 `OPENAI_BASE_URL` 是否正确
2. 测试 API 连接：
   ```bash
   curl -X POST http://localhost:13000/v1/chat/completions \
     -H "Authorization: Bearer sk-xxx" \
     -H "Content-Type: application/json" \
     -d '{"model":"gpt-4o","messages":[{"role":"user","content":"test"}]}'
   ```
3. 检查 One-API 是否正常运行：
   ```bash
   docker ps | grep one-api
   ```

---

### 6.2 问题 2：Neo4j 连接失败

**错误信息**：
```
Neo4j connection error: Unable to connect to neo4j://localhost:7687
```

**排查步骤**：
1. 检查 Neo4j 是否启动：
   ```bash
   docker ps | grep neo4j
   ```
2. 检查端口是否被占用：
   ```bash
   netstat -tuln | grep 7687
   ```
3. 检查 `.env` 中的密码是否正确

---

### 6.3 问题 3：内存溢出（OOM）

**错误信息**：
```
MemoryError: Unable to allocate array
```

**解决方案**：
1. 减小批处理大小：
   ```bash
   # 在 .env 中修改
   BATCH_SIZE=50
   ENTITY_BATCH_SIZE=25
   EMBEDDING_BATCH_SIZE=32
   ```
2. 减少并发线程数：
   ```bash
   MAX_WORKERS=2
   ```
3. 分批处理文档（每次只放 1-2 个文件到 `files/` 目录）

---

### 6.4 问题 4：实体提取质量差

**现象**：
- 提取的实体类型错误（把"优秀学生"识别为"奖学金"）
- 关系提取不准确

**解决方案**：
1. 优化 Schema 定义（`graphrag_agent/config/settings.py`）：
   ```python
   entity_types = [
       "学生类型",  # 明确定义
       "奖学金类型",  # 与"学生类型"区分开
       "处分类型",
       ...
   ]
   ```
2. 增加示例（Few-shot Learning）：
   ```python
   examples = [
       {
           "text": "优秀学生可以申请国家奖学金",
           "entities": [
               {"name": "优秀学生", "type": "学生类型"},
               {"name": "国家奖学金", "type": "奖学金类型"}
           ],
           "relationships": [
               {"source": "优秀学生", "target": "国家奖学金", "type": "申请"}
           ]
       }
   ]
   ```

---

## 七、实验记录模板

### 7.1 为什么要记录实验？

**面试时的价值**：
- 展示你的实践能力（不是纸上谈兵）
- 提供具体的数据支撑（"我构建了一个包含 500 个实体的图谱"）
- 体现你的问题解决能力（遇到了什么问题，如何解决）

---

### 7.2 实验记录表格

| 实验项 | 数值 | 备注 |
|--------|------|------|
| 文档数量 | 3 | 2023学生手册.pdf + 2个规章制度 |
| 总字符数 | 125,000 | |
| Chunk 数量 | 250 | CHUNK_SIZE=500 |
| 提取的实体数 | 128 | 去重前 |
| 去重后实体数 | 105 | 消歧合并了 23 个 |
| 关系数量 | 95 | |
| 社区数量 | 12 | Leiden 算法 |
| 构建耗时 | 8 分 32 秒 | 包含 LLM 调用 |
| LLM 调用次数 | 约 300 次 | 实体提取 + 消歧 + 摘要 |
| API 费用 | 约 $2.5 | GPT-4o 价格 |

---

### 7.3 截图清单

**必须截图的内容**：
1. Neo4j Browser 中的图谱可视化
2. 实体统计查询结果
3. 社区检测结果
4. 增量更新日志

**截图存放位置**：
```
study_note/
  day7_screenshots/
    01_neo4j_graph.png
    02_entity_stats.png
    03_community_detection.png
    04_incremental_update.png
```

---

## 八、面试 STAR 话术准备

### 8.1 Story：亲手构建知识图谱

**S（Situation）**：
在学习 GraphRAG 项目时，我需要从理论走向实践，真正构建一个可用的知识图谱。

**T（Task）**：
我的任务是：
1. 搭建完整的运行环境（Python + Neo4j + LLM API）
2. 使用项目提供的学生手册数据构建知识图谱
3. 验证图谱质量并进行增量更新测试

**A（Action）**：
1. **环境配置**：
   - 使用 Docker Compose 启动 Neo4j
   - 配置 One-API 作为 LLM 代理
   - 调整 `.env` 中的批处理参数以适配我的机器性能

2. **图谱构建**：
   - 执行三步构建流程：基础图谱 → 索引和社区 → Chunk 索引
   - 观察日志输出，理解每一步的作用

3. **质量验证**：
   - 在 Neo4j Browser 中查询实体和关系
   - 验证实体消歧效果（检查是否有重复实体）
   - 可视化社区结构

4. **增量更新**：
   - 添加新文档并执行增量更新
   - 对比 `file_registry.json` 的变化
   - 验证新实体是否正确合并到图谱中

**R（Result）**：
- 成功构建了包含 105 个实体、95 个关系、12 个社区的知识图谱
- 实体消歧率达到 18%（23/128），有效减少了重复节点
- 增量更新耗时仅为全量构建的 1/5，验证了增量机制的高效性
- 通过实践深入理解了图构建的每个环节，为后续优化打下基础

---

## 九、今日打卡清单

- [ ] 成功启动 Neo4j 并在 Browser 中查看图谱
- [ ] 执行完整的三步构建流程，无报错
- [ ] 在 Neo4j 中查询到至少 50 个实体节点
- [ ] 验证实体消歧效果（找到至少 1 组合并的实体）
- [ ] 执行一次增量更新，观察 `file_registry.json` 的变化
- [ ] 截图保存关键结果（图谱可视化、统计信息）
- [ ] 填写实验记录表格

---

## 十、下一步：Day 8 预告

Day 8 我们将进入 **Agent 问答实战**：
- 启动 FastAPI 后端和 Streamlit 前端
- 测试 5 种 Agent 的回答质量
- 对比不同 Agent 的性能和准确性
- 开启 Debug 模式，观察 LangGraph 执行轨迹
- 调参实验：修改 `.env` 参数，观察性能变化

---

*学习日期：2026-02-28 | 项目路径：`/home/wkt/project/graph-rag-agent`*
