# Day 8 学习笔记 — 亲手复现：Agent 问答实战

> 日期：2026-03-01
>
> **阅读前提**：已完成 Day 1-7，理解了图构建流程、Agent 架构、搜索策略、质量保证机制、多智能体架构，并已亲手跑通知识图谱构建。

---

## 写在前面：Day 8 要解决的核心问题

Day 7 我们已经成功构建了知识图谱，但知识图谱只是"原材料"——**真正的价值在于让 Agent 基于图谱进行智能问答**。

Day 8 的核心目标是：**跑通所有 Agent 的问答流程，理解请求从前端到后端的完整生命周期，并通过对比实验深刻理解不同 Agent 的差异。**

```
Day 7（图构建实战）→ Day 8（Agent 问答实战）→ Day 9-10（面试准备）
       ↓                     ↓
  知识图谱就绪            Agent 能回答问题了
```

今天我们要回答这些核心问题：
1. 一个用户的问题，从浏览器输入到最终回答，**经历了哪些环节**？
2. 5 种 Agent **对同一个问题的回答有什么差异**？差异的根源是什么？
3. **Debug 模式**能看到什么？如何用它理解 Agent 的推理过程？
4. 哪些**配置参数**会影响 Agent 的回答质量和速度？

---

## 一、系统启动：理解服务架构

### 1.1 整体架构：前后端分离

本项目采用经典的**前后端分离架构**：

```
┌────────────────────────┐     HTTP/SSE      ┌────────────────────────┐
│   Frontend (Streamlit) │ ◄──────────────── │   Backend (FastAPI)    │
│   端口: 8501           │ ────────────────► │   端口: 8000           │
│                        │                    │                        │
│  - 用户界面            │                    │  - Agent 管理          │
│  - 消息展示            │                    │  - 知识图谱查询        │
│  - Debug 面板          │                    │  - 缓存管理            │
│  - 性能监控            │                    │  - 反馈处理            │
└────────────────────────┘                    └────────────────────────┘
                                                       │
                                              ┌────────┴────────┐
                                              │   Neo4j (7687)  │
                                              │   知识图谱存储    │
                                              └─────────────────┘
```

**为什么要前后端分离？**
- **关注点分离**：前端只管展示和交互，后端只管业务逻辑
- **独立扩展**：后端可以水平扩展（多个 Worker），前端不受影响
- **多客户端支持**：除了 Streamlit 前端，还可以用 Curl、Postman 或其他客户端调用 API

---

### 1.2 启动后端：FastAPI 服务

**启动命令**：
```bash
cd /home/wkt/project/graph-rag-agent
python server/main.py
```

**后端启动时做了什么？**

```python
# server/main.py 核心逻辑（约 34 行）
app = FastAPI(title="知识图谱问答系统", description="基于知识图谱的智能问答系统后端API")

# 1. 注册路由
app.include_router(api_router)  # 包含 chat、feedback、knowledge_graph、source 四组路由

# 2. 连接 Neo4j 数据库
driver = get_db_manager()

# 3. 初始化 Agent 管理器
agent_manager = AgentManager()  # 管理所有 Agent 实例的生命周期

# 4. 注册关闭钩子（优雅关闭）
@app.on_event("shutdown")
def shutdown_event():
    agent_manager.close_all()  # 释放所有 Agent 资源
    driver.close()             # 关闭 Neo4j 连接
```

**四组路由一览**：

| 路由组 | 前缀 | 核心端点 | 作用 |
|--------|------|----------|------|
| chat | `/chat`, `/chat/stream` | 对话问答 | 接收问题，调用 Agent 返回回答 |
| feedback | `/feedback` | 用户反馈 | 正向反馈标记缓存高质量，负向反馈清除缓存 |
| knowledge_graph | `/knowledge_graph`, `/kg_reasoning` | 图谱操作 | 查询、可视化、推理（最短路径、社区等） |
| source | `/source`, `/source_info` | 源内容 | 获取原始文本块和文件元数据 |

---

### 1.3 启动前端：Streamlit 应用

**启动命令**：
```bash
streamlit run frontend/app.py
```

**前端启动时做了什么？**

```python
# frontend/app.py 核心逻辑（约 48 行）
def main():
    # 1. 初始化会话状态
    init_session_state()
    #    - session_id: UUID（标识当前对话）
    #    - messages: 消息历史列表
    #    - debug_mode: 调试模式开关
    #    - agent_type: 当前选择的 Agent 类型
    #    - use_stream: 是否启用流式输出
    #    - execution_log: 执行轨迹日志
    #    - kg_data: 知识图谱可视化数据

    # 2. 初始化性能监控
    init_performance_monitoring()

    # 3. 渲染界面
    if st.session_state.debug_mode:
        # Debug 模式：5:4 双栏布局
        col1, col2 = st.columns([5, 4])
        with col1:
            display_chat_interface()   # 左侧：聊天界面
        with col2:
            display_debug_panel()      # 右侧：调试面板
    else:
        # 普通模式：全屏聊天
        display_chat_interface()
```

**关键设计决策**：Debug 模式会**自动关闭流式输出**（`use_stream = False`），因为执行轨迹日志只能在 Agent 完整运行后才能捕获。

---

## 二、请求生命周期：一个问题从输入到回答的完整旅程

这是 Day 8 最核心的内容。理解了这个完整链路，你就理解了整个系统的工作原理。

### 2.1 完整请求流程图

```
用户在浏览器输入问题："优秀学生的申请条件是什么？"
    │
    ▼
┌─── ① 前端 API 层 ───────────────────────────────────────────┐
│ frontend/utils/api.py                                        │
│                                                              │
│ 判断输出模式：                                                │
│ ├─ 流式模式 → send_message_stream() → POST /chat/stream     │
│ │   返回 SSE（Server-Sent Events）流                         │
│ └─ 非流式模式 → send_message() → POST /chat                 │
│     返回完整 JSON                                            │
│                                                              │
│ 请求参数：                                                    │
│ {                                                            │
│   "message": "优秀学生的申请条件是什么？",                      │
│   "session_id": "uuid-xxxx",                                 │
│   "debug": false,                                            │
│   "agent_type": "hybrid_agent",                              │
│   "use_deeper_tool": true,                                   │
│   "show_thinking": false                                     │
│ }                                                            │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTP POST
                       ▼
┌─── ② 路由层 ────────────────────────────────────────────────┐
│ server/routers/chat.py                                       │
│                                                              │
│ POST /chat:                                                  │
│   1. 解析 ChatRequest                                        │
│   2. 调用 process_chat() 服务函数                              │
│   3. 返回 ChatResponse                                        │
│                                                              │
│ POST /chat/stream:                                           │
│   1. 解析 JSON 请求                                           │
│   2. 调用 process_chat_stream() 服务函数                       │
│   3. 返回 SSE 事件流                                          │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌─── ③ 服务层（核心业务逻辑）──────────────────────────────────┐
│ server/services/chat_service.py                               │
│                                                              │
│ process_chat() 核心流程：                                      │
│                                                              │
│ Step 1: 获取会话锁（并发控制）                                  │
│   lock_key = f"{session_id}_chat"                            │
│   if not try_acquire_lock(lock_key):                         │
│       return 429 "另一个请求正在处理"                           │
│                                                              │
│ Step 2: 快速缓存检查                                          │
│   fast_result = agent.check_fast_cache(message, session_id)  │
│   if fast_result: return fast_result  # 毫秒级返回             │
│                                                              │
│ Step 3: 获取 Agent 实例                                       │
│   agent = agent_manager.get_agent(agent_type, session_id)    │
│                                                              │
│ Step 4: 调用 Agent 处理                                       │
│   ├─ Debug 模式：                                             │
│   │   ├─ DeepResearch → ask_with_thinking()                  │
│   │   └─ 其他 Agent  → ask_with_trace()                      │
│   └─ 普通模式：                                               │
│       └─ agent.ask(message, thread_id=session_id)            │
│                                                              │
│ Step 5: 后处理                                                │
│   ├─ 提取知识图谱数据（非 DeepResearch）                       │
│   ├─ 格式化执行日志（Debug 模式）                               │
│   └─ 释放会话锁                                               │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌─── ④ Agent 管理层 ──────────────────────────────────────────┐
│ server/services/agent_service.py                              │
│                                                              │
│ AgentManager.get_agent(agent_type, session_id):              │
│   instance_key = f"{agent_type}:{session_id}"                │
│   ├─ 已存在？→ 直接返回（复用实例，保留对话历史）               │
│   └─ 不存在？→ 创建新实例（懒加载）                           │
│                                                              │
│ 关键设计：                                                    │
│ - 每个 (agent_type, session_id) 组合独立实例                  │
│ - RLock 保证线程安全                                          │
│ - 懒加载避免启动时创建所有 Agent                               │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌─── ⑤ Agent 执行层（Day 3-6 深入学习过的内容）────────────────┐
│ graphrag_agent/agents/*.py                                    │
│                                                              │
│ agent.ask(query, thread_id) 内部流程：                         │
│                                                              │
│ A. 三级缓存检查（命中则立即返回）                               │
│   1. 全局缓存 → global_cache_manager.get(query)              │
│   2. 快速缓存 → check_fast_cache(query, session_id)          │
│   3. 会话缓存 → cache_manager.get(query, thread_id=...)      │
│                                                              │
│ B. 缓存未命中 → 执行 LangGraph 状态图                         │
│   START → agent_node → tools → retrieve → generate → END    │
│                                                              │
│ C. 结果写入缓存                                               │
│   cache_manager.set(query, answer)                           │
│   global_cache_manager.set(query, answer)                    │
│                                                              │
│ D. 返回回答字符串                                              │
└──────────────────────┬───────────────────────────────────────┘
                       │ 回答原路返回
                       ▼
┌─── ⑥ 前端展示层 ────────────────────────────────────────────┐
│ frontend/utils/api.py → frontend/app.py                       │
│                                                              │
│ 流式模式：                                                    │
│   SSE 事件类型：                                              │
│   ├─ "start"         → 开始标记                               │
│   ├─ "token"         → 回答文本块（逐块展示）                   │
│   ├─ "thinking"      → 思考过程（DeepResearch 专属）            │
│   ├─ "execution_log" → 执行轨迹（Debug 模式）                  │
│   ├─ "done"          → 完成标记                               │
│   └─ "error"         → 错误信息                               │
│                                                              │
│ 非流式模式：                                                  │
│   直接渲染完整回答 + Debug 面板数据                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 为什么要了解这个完整链路？（面试价值）

**面试官可能的追问**：
- "你能描述一下系统的请求处理流程吗？" → 完整链路
- "并发请求怎么处理？" → 会话锁机制（👉 **详见 `day8_note_QA.md` Q2: 会话锁和 Agent 锁有什么区别？**）
- "缓存命中和未命中的区别？" → 三级缓存策略
- "流式输出是怎么实现的？" → SSE 事件流（👉 **详见 `day8_note_QA.md` Q4: SSE 和 WebSocket 有什么区别？**）

---

## 三、Agent 管理：AgentManager 的设计哲学

### 3.1 为什么需要 AgentManager？（硬核原理解析）

为了理解 `AgentManager` 的重要性，我们直接来看源码中的核心机制：**实例键（`instance_key`）**。

在 `server/services/agent_service.py` 中，你可以看到这段关键代码：
```python
# AgentManager 的核心分发逻辑
instance_key = f"{agent_type}:{session_id}"

if instance_key not in self.agent_instances:
    # 懒加载：只有第一次遇到这个组合时，才在内存里 new 一个对象
    self.agent_instances[instance_key] = self.agent_classes[agent_type]()

return self.agent_instances[instance_key]
```

**场景还原：如果没有 AgentManager 会怎样？**
- 做法 A（**全局单例**）：所有人共享一个 `GraphAgent` 实例。张三问“奖学金有多少钱”，李四接着问“我能申请吗”，机器人的“记忆（MemorySaver）”就串线了，回答李四时会带上张三的上下文。
- 做法 B（**每次新建**）：每次发请求都 `new GraphAgent()`。张三问“奖学金有多少钱”，机器人答了。张三接着追问“那国家级的呢？”，因为是一个全新的机器人实例，它根本不知道上一轮聊了什么，也丢失了上下文。

**AgentManager 的绝妙设计（会话隔离 + 资源复用）：**
我们用一个具体的用户行为轴来推演：
1. **用户 A 登录界面（session_id="ua_001"），选择了 GraphAgent**
   - AgentManager 拼接 `instance_key = "graph_agent:ua_001"`
   - 字典里没找到，于是新建 `对象甲` 存放在字典里并返回。
   - `对象甲` 记住了他们关于“奖学金”的初次对话。
2. **用户 B 登录界面（session_id="ub_002"），也选择了 GraphAgent**
   - 拼接 `instance_key = "graph_agent:ub_002"`
   - 没找到，新建 `对象乙`，完全和 `对象甲` 物理隔离（解决了对话串线问题）。
3. **用户 A 接着追问“那申请条件是什么？”**
   - 再次拼接 `instance_key = "graph_agent:ua_001"`
   - 在字典里找到了刚才的 `对象甲`！直接复用并返回。
   - 因为是同一个 Python 对象，LangGraph 的 `MemorySaver` 里完整保留了上一轮对“奖学金”的探讨（解决了上下文丢失问题）。
4. **用户 A 突发奇想，在界面下拉框里切换成了 NaiveRAGAgent，问了同样的问题**
   - 拼接 `instance_key = "naive_rag_agent:ua_001"`
   - 新建 `对象丙`。这意味着，**哪怕是同一个用户，只要切换了 Agent 模型，也是一条全新的平行世界记忆线**，互相不干扰。（👉 **为什么不共享记忆？详见 `day8_note_QA.md` Q13 隔离记忆的深度解析**）

### 3.2 AgentManager 的实现

```python
# server/services/agent_service.py（约 211 行）
class AgentManager:
    def __init__(self):
        # 注册所有可用的 Agent 类型
        self.agent_classes = {
            "graph_agent":           GraphAgent,
            "hybrid_agent":          HybridAgent,
            "naive_rag_agent":       NaiveRagAgent,
            "deep_research_agent":   DeepResearchAgent,
            "fusion_agent":          FusionGraphRAGAgent,
        }
        self.agent_instances = {}      # 实例池
        self.agent_lock = threading.RLock()  # 线程安全锁

    def get_agent(self, agent_type, session_id="default"):
        instance_key = f"{agent_type}:{session_id}"
        with self.agent_lock:
            if instance_key not in self.agent_instances:
                # 懒加载：首次请求时才创建
                self.agent_instances[instance_key] = self.agent_classes[agent_type]()
            return self.agent_instances[instance_key]
```

**关键设计点**：

| 设计 | 原因 |
|------|------|
| `instance_key = f"{agent_type}:{session_id}"` | 保证不同用户、不同 Agent 之间完全隔离 |
| `threading.RLock()` | 可重入锁，防止多线程并发创建同一实例（👉 **详见 `day8_note_QA.md` Q1: 为什么用 RLock？**） |
| 懒加载 | 避免服务启动时创建所有 Agent（节省资源） |

### 3.3 并发控制：会话锁

```python
# server/services/chat_service.py
lock_key = f"{session_id}_chat"
lock_acquired = chat_manager.try_acquire_lock(lock_key)
if not lock_acquired:
    raise HTTPException(status_code=429, detail="另一个请求正在处理")
```

**为什么需要会话锁？**
- LangGraph 的 `MemorySaver` 不是线程安全的
- 同一会话的并发请求可能导致消息历史错乱
- 返回 HTTP 429（Too Many Requests）让前端知道需要等待

---

## 四、五种 Agent 对比：核心差异深度解析

这是 Day 8 最重要的实战内容。对同一个问题，不同 Agent 的处理方式截然不同。

### 4.1 Agent 能力矩阵

```
                    速度快 ◄────────────────────────► 速度慢
                    深度浅 ◄────────────────────────► 深度深

    NaiveRAG ──── HybridAgent ──── GraphAgent ──── DeepResearch ──── FusionAgent
       │              │                │               │                │
    纯向量检索     混合检索策略     图结构推理      多步迭代推理      多Agent协作
       │              │                │               │                │
    1个工具         2个工具          2个工具         多个工具        无（委托）
       │              │                │               │                │
    ~3秒            ~5秒            ~8秒            ~30秒           ~60秒+
```

### 4.2 逐一解析每种 Agent

#### Agent 1：NaiveRagAgent（朴素检索 Agent）

**定位**：最简单的 baseline，纯粹的向量检索 + LLM 回答。

**注册工具**：`NaiveSearchTool`（1 个工具）

**处理流程**：
```
用户问题 → 向量相似度检索 Top-K 个文本块 → 拼接上下文 → LLM 生成回答
```

**LangGraph 状态图**：
```
START → agent → tools(NaiveSearch) → retrieve → generate → END
```

**System Prompt 核心要求**：
- 严格基于检索到的文档块回答
- 信息不在文档中时回答"不知道"
- 引用源文本块 ID

**优势**：速度最快，实现最简单
**劣势**：
- 无法处理跨文档的关系推理
- 无法回答全局性问题（"有哪些奖学金？"）
- 上下文窗口有限，容易遗漏信息

**面试话术**：NaiveRAG 作为系统的 baseline，验证了"纯向量检索在简单事实性问题上的表现"。它的局限性正是我们引入 GraphRAG 的原因。

---

#### Agent 2：GraphAgent（图结构推理 Agent）

**定位**：利用知识图谱的拓扑结构进行多层次检索。

**注册工具**：`LocalSearchTool` + `GlobalSearchTool`（2 个工具）

**处理流程**：
```
用户问题
    │
    ├─ 关键词提取（LLM 提取低级 + 高级关键词）
    │
    ├─ LocalSearch（实体邻居扩展）
    │   用户问题 → 实体向量召回 → 图邻居扩展（1-2 跳）→ 子图提取
    │
    └─ GlobalSearch（社区摘要聚合）
        用户问题 → 社区摘要向量检索 → Map-Reduce 聚合
```

**LangGraph 状态图（比其他 Agent 更复杂）**：
```
START → agent → tools → retrieve → grade_documents
                                       │
                               ┌───────┴───────┐
                               ▼               ▼
                           generate          reduce
                           (本地搜索)       (全局搜索)
                               │               │
                               ▼               ▼
                              END             END
```

**关键设计：文档质量评估（`_grade_documents`）**：

这是 GraphAgent 独有的关键节点。在检索到文档后，它会评估检索质量：

```python
def _grade_documents(state):
    # 1. 如果调用了 GlobalSearch → 走 reduce 路径
    if global_retriever_was_called:
        return "reduce"

    # 2. 计算关键词匹配率
    match_rate = matched_keywords / total_keywords

    # 3. 如果匹配率足够高 → 走 generate 路径
    if match_rate > threshold:
        return "generate"

    # 4. 如果匹配率不够 → 回退到 local search 补充
    return "local_search_fallback"
```

**两条生成路径**：
1. **Generate 路径**：处理本地搜索结果，生成以实体为中心的回答
2. **Reduce 路径**：处理全局搜索结果，使用 Map-Reduce 模式聚合社区信息

**优势**：
- 利用图结构发现隐性关联（"旷课" → "违纪处分" → "不能申请奖学金"）
- 本地搜索 + 全局搜索互补（👉 **详见 `day8_note_QA.md` Q6: 检索范围对比**）
- 文档质量评估保证回答质量（👉 **详见 `day8_note_QA.md` Q7: _grade_documents 为什么重要？**，以及 **Q12: 为什么有两条生成路径？**）

**劣势**：
- 关键词提取需要额外一次 LLM 调用
- 比 NaiveRAG 慢 2-3 倍

---

#### Agent 3：HybridAgent（混合策略 Agent）

**定位**：融合多种搜索策略的"万金油"型 Agent。

**注册工具**：`HybridSearchTool`（混合搜索）+ `HybridGlobalTool`（全局变体）

**处理流程**：
```
用户问题 → HybridSearchTool 内部融合多种检索策略 → 统一结果 → LLM 生成
```

**LangGraph 状态图**：
```
START → agent → tools(HybridSearch) → retrieve → generate → END
```

**与 GraphAgent 的核心区别**：
- GraphAgent 让 LLM 自己决定调用 LocalSearch 还是 GlobalSearch
- HybridAgent 的搜索工具内部已经融合了多种策略，对 LLM 来说只有一个工具
- 结构更简单（没有 `_grade_documents` 和 `reduce` 节点）

**关键词提取**：委托给 `HybridSearchTool.extract_keywords()`，比 GraphAgent 更高效（工具内部实现，不需要额外 LLM 调用）

**优势**：结构简洁，搜索策略融合在工具内部
**劣势**：灵活性不如 GraphAgent（无法动态选择搜索策略）

---

#### Agent 4：DeepResearchAgent（深度研究 Agent）

**定位**：多轮迭代推理，适合复杂问题的深度研究。

**注册工具**：
- `DeepResearchTool` 或 `DeeperResearchTool`（增强版）
- `exploration_tool`（知识图谱遍历）
- `reasoning_analysis_tool`（推理链分析）
- `stream_tool`（流式输出）

**处理流程**：
```
用户问题
    │
    ▼
┌─── 迭代循环（多轮 Think-Search-Reason）─────────────┐
│                                                      │
│  第 1 轮：                                            │
│    Think: 分析问题，确定搜索方向                        │
│    Search: 执行知识库检索                               │
│    Reason: 评估检索结果，提炼有用信息                    │
│    → 信息不够？继续下一轮                               │
│                                                      │
│  第 2 轮：                                            │
│    Think: 基于第 1 轮结果，调整搜索策略                  │
│    Search: 用新角度检索                                │
│    Reason: 汇总所有证据                                │
│    → 信息充分？退出循环                                │
│                                                      │
│  ...（最多 N 轮）                                     │
└──────────────────────────────────────────────────────┘
    │
    ▼
最终汇总 → 生成结构化长文回答
```

**独有能力**：

1. **Thinking Mode（思考模式）**：
```python
result = agent.ask_with_thinking(query)
# 返回：
# {
#     "thinking_process": "<think>分析步骤...</think>",
#     "answer": "最终回答",
#     "retrieved_info": ["第1轮信息", "第2轮信息"],
#     "execution_logs": ["日志1", "日志2"]
# }
```

2. **矛盾检测**：
```python
result = agent.detect_contradictions(query)
# 检测数值矛盾和语义矛盾
# 返回：
# {
#     "has_contradictions": True,
#     "count": 2,
#     "contradictions": [...],
#     "impact_analysis": "矛盾影响分析..."
# }
```

3. **知识图谱探索**：
```python
result = agent.explore_knowledge(query)
# 在知识图谱上进行多步探索
# 返回探索路径和实体发现
```

**优势**：
- 多轮迭代确保信息充分性
- 思考过程可追溯（可解释性强）
- 矛盾检测提升准确性

**劣势**：
- 耗时长（多轮 LLM + 检索）
- API 消耗大
- 可能在简单问题上"过度思考"

---

#### Agent 5：FusionGraphRAGAgent（融合多智能体 Agent）

**定位**：项目最核心的亮点，Plan-Execute-Report 多智能体协作。

**架构特点**：FusionAgent **不是**一个传统的 LangGraph Agent，而是一个轻量级包装器，将所有工作委托给 `MultiAgentFacade`。

```python
# graphrag_agent/agents/fusion_agent.py
class FusionGraphRAGAgent:
    def __init__(self):
        self.multi_agent = MultiAgentFacade()  # 核心编排器
        self._global_cache = {}                 # 简单内存缓存
        self._session_cache = {}                # 会话缓存
```

**处理流程（Day 6 深入学习过）**：
```
用户问题
    │
    ▼
┌── Plan 阶段 ──────────────────────────────┐
│ Clarifier → TaskDecomposer → PlanReviewer │
│ 输出：PlanSpec（任务 DAG，含依赖关系）      │
└────────────────┬──────────────────────────┘
                 │
                 ▼
┌── Execute 阶段 ───────────────────────────┐
│ WorkerCoordinator 按拓扑排序调度            │
│ ├─ RetrievalExecutor（检索型任务）          │
│ ├─ ResearchExecutor（研究型任务）           │
│ └─ ReflectionExecutor（反思验证）           │
│ 输出：ExecutionRecord 列表                  │
└────────────────┬──────────────────────────┘
                 │
                 ▼
┌── Report 阶段 ────────────────────────────┐
│ OutlineBuilder → SectionWriter(Map-Reduce) │
│ → ConsistencyChecker                       │
│ 输出：结构化长文报告（含引用和事实核验）      │
└───────────────────────────────────────────┘
```

**与其他 Agent 的本质区别**：

| 维度 | 其他 Agent | FusionAgent |
|------|-----------|-------------|
| 工作流引擎 | LangGraph StateGraph | MultiAgentFacade |
| 工具调用 | LLM 自主决定 | Planner 规划分配 |
| 任务粒度 | 整个问题一次处理 | 分解为子任务并行/串行执行 |
| 输出格式 | 简短回答 | 5000+ 字结构化长文 |
| 质量保证 | 无 | ConsistencyChecker 事实核验 |

👉 **关于 FusionAgent 不使用 LangGraph 的底层原因，详见 `day8_note_QA.md` Q8: 架构差异深度解析。**

**优势**：
- 生成最完整、最结构化的回答
- 多智能体协作保证质量
- 支持并行任务执行

**劣势**：
- 耗时最长（60 秒以上）
- API 消耗最大
- 简单问题杀鸡用牛刀

---

### 4.3 五种 Agent 对比总结表

| 维度 | NaiveRAG | GraphAgent | HybridAgent | DeepResearch | FusionAgent |
|------|----------|------------|-------------|--------------|-------------|
| **搜索策略** | 纯向量 | 本地+全局 | 混合融合 | 迭代深搜 | Plan分配 |
| **工具数量** | 1 | 2 | 2 | 4+ | 0（委托） |
| **关键词提取** | 无 | LLM调用 | 工具内置 | 工具内置 | Planner |
| **典型耗时** | ~3s | ~8s | ~5s | ~30s | ~60s+ |
| **回答长度** | 短 | 中 | 中 | 长 | 超长 |
| **适用问题** | 简单事实 | 关系推理 | 通用问答 | 复杂研究 | 报告生成 |
| **图结构利用** | 无 | 深度 | 中度 | 中度 | 取决于子任务 |
| **可解释性** | 低 | 中 | 中 | 高 | 高 |
| **LLM调用次数** | 1-2 | 3-5 | 2-3 | 10+ | 20+ |

---

## 五、实战测试：跑通测试脚本

### 5.1 测试脚本结构

**1. 什么是流式与非流式？为什么分两个脚本当测试？**

最直观的体验差异：
- **非流式（Non-Stream）**：像发邮件。你把问题发给 Agent，然后死等（可能是 5 秒，也可能是 60 秒），直到它把几千字的完整答案全部写完，再一次性“啪”的一下全都糊在你脸上。用户在这个死等的时间里，看到的就是一个转圈圈，体验极差，非常容易烦躁。
- **流式（Stream）**：像打字机/发微信语音。Agent 思考出第一句话，就立马通过网络（SSE 事件流）发到前端展示出来；思考出第二句话，接着发出来。用户能**看着答案一点点像打字一样蹦出来**，虽然总耗时是一样的，但用户会觉得“它在干活”、“很快就理我了”。

**2. 核心关注的性能指标不同**

| 指标 | 非流式测试 (`search_without_stream.py`) | 流式测试 (`search_with_stream.py`) | 业务意义 |
|------|---------------------------------------|-----------------------------------|----------|
| **接口调用** | `answer = agent.ask(query)` | `async for chunk in agent.ask_stream()` | 底层调用方式不同（一次性返回字符串 vs 异步生成器） |
| **主要耗时指标** | **总耗时（Total Execution Time）** | **首块延迟（TTFT: Time-To-First-Token）** | TTFT 决定了用户的“耐心边界”。只要首字延迟控制在 2-3 秒内，总耗时就算 1 分钟，用户也愿意看它慢慢打字。 |
| **测试场景** | 验证**回答是否正确完整**，或者开启 Debug 模式获取**完整的执行轨迹**（因为只有等回答全部生成，逻辑树才完整）。 | 验证**生产环境真实体验**。主要看前端收到响应有多平滑，会不会有长时间的卡顿（比如憋了 30 秒才吐出第一个块）。 |
| **指标维度** | 只有最终的时间和结果 | 额外统计：发了多少个数据块（Chunk）、等待首块的时间、网络断联保护（300s 超时） |

由于 Debug 面板的“执行日志轨迹”必须要等 Agent 全部的生命周期跑完才能完整生成，所以**前面提到的 Debug 模式，在代码底层会被强制降级为非流式处理**（否则日志渲染会错乱）。这就是为啥项目要单独区分两个脚本：平时用流式测真实体验，修 Bug 时用非流式抓全量完整日志。

👉 **伪流式和真流式的技术压制原因，详见 `day8_note_QA.md` Q3 深度解析。**

### 5.2 执行测试

**Step 1：非流式测试（推荐先从这个开始）**

```bash
cd /home/wkt/project/graph-rag-agent/test
python search_without_stream.py
```

**修改建议**：为了对比所有 Agent，取消注释所有 Agent 实例：

```python
agents = [
    {"name": "NaiveRagAgent", "instance": NaiveRagAgent()},
    {"name": "GraphAgent", "instance": GraphAgent()},
    {"name": "HybridAgent", "instance": HybridAgent()},
    {"name": "DeepResearchAgent", "instance": DeepResearchAgent(use_deeper_tool=True)},
    {"name": "FusionGraphRAGAgent", "instance": FusionGraphRAGAgent()}
]
```

**注意**：全部 Agent 同时测试会很慢（总计可能需要 5-10 分钟），建议一开始只启用 1-2 个 Agent。

**Step 2：流式测试**

```bash
cd /home/wkt/project/graph-rag-agent/test
python search_with_stream.py
```

**观察指标**：
```
[完成] 流式查询完成
- 总耗时: 12.34秒
- 首块延迟: 3.21秒      ← 用户感知到的"等待时间"
- 数据块数: 45个
- 总字符数: 2340字符
```

👉 **如果需要现场演示展示项目的全貌，详见 `day8_note_QA.md` Q11: 最应该展示什么？**

### 5.3 推荐的测试问题及分析

| 测试问题 | 难度 | 测试目标 | 预期差异 |
|----------|------|----------|----------|
| "优秀学生的申请条件是什么？" | 简单 | 基础事实检索 | 所有 Agent 应该都能正确回答 |
| "学业奖学金有多少钱？" | 中等 | 数值精准度 | 测试 Agent 是否能准确提取数字 |
| "大学英语考试的标准是什么？" | 中等 | 跨文档检索 | GraphAgent 应比 NaiveRAG 更全面 |
| 小明旷课+私藏+殴打，能否评奖学金？ | 复杂 | 多条件推理 | DeepResearch/Fusion 显著优于其他 |

**第 4 个问题是关键测试**：
- NaiveRAG：可能只找到"旷课"的规定，遗漏"殴打"和"私藏"
- GraphAgent：通过图邻居扩展，可能找到"违纪处分 → 不能评奖学金"的路径
- DeepResearch：多轮迭代，分别搜索三个违规行为，最后综合判断
- FusionAgent：将问题分解为 3 个子任务，分别验证后汇总

---

## 六、Debug 模式：透视 Agent 的"大脑"

### 6.1 如何开启 Debug 模式

👉 **关于开启 Debug 模式为什么会自动并且必须关闭流式输出，详见 `day8_note_QA.md` Q5。**

**方法 1：通过前端 UI**
1. 打开 Streamlit 前端（http://localhost:8501）
2. 在左侧边栏找到"Debug Mode"开关
3. 打开后，界面变为双栏布局

**方法 2：通过环境变量**
```bash
# 在 .env 中设置默认开启
FRONTEND_DEFAULT_DEBUG=true
```

**方法 3：通过 API 直接调用**
```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "优秀学生的申请条件是什么？",
    "session_id": "test-debug",
    "debug": true,
    "agent_type": "graph_agent"
  }'
```

### 6.2 Debug 面板包含什么？

Debug 面板有 **5 个标签页**：

```
┌────────────────────────────────────────────────────┐
│  执行轨迹 │ 知识图谱 │ 源内容 │ 图谱管理 │ 性能监控 │
├────────────────────────────────────────────────────┤
│                                                    │
│  Tab 1: 执行轨迹                                    │
│  ├─ 普通 Agent: LangGraph 节点执行记录              │
│  │   node: "agent" → input/output                  │
│  │   node: "tools" → tool_calls/results            │
│  │   node: "generate" → final_answer               │
│  │                                                  │
│  └─ DeepResearch Agent:                             │
│      ├─ 工具类型指示（标准版/增强版）                 │
│      ├─ 迭代轮次选择器                               │
│      ├─ 每轮详情：                                   │
│      │   ├─ 执行的搜索查询                           │
│      │   ├─ 发现的有用信息                           │
│      │   ├─ 知识库检索结果                           │
│      │   └─ 详细日志（可展开）                       │
│      └─ 增强功能详情（社区感知、证据链等）             │
│                                                    │
│  Tab 2: 知识图谱                                    │
│  ├─ PyVis 交互式图谱可视化                           │
│  ├─ 物理引擎参数调节                                 │
│  └─ 节点颜色编码（中心/源/目标/1跳/2跳）             │
│                                                    │
│  Tab 3: 源内容                                      │
│  └─ 检索到的原始文本块展示                           │
│                                                    │
│  Tab 4: 图谱管理（懒加载）                           │
│  ├─ 实体 CRUD 操作                                   │
│  └─ 关系 CRUD 操作                                   │
│                                                    │
│  Tab 5: 性能监控                                    │
│  ├─ 应用运行时间                                     │
│  ├─ API 延迟                                     │
│  └─ API 调用次数                                 │
└────────────────────────────────────────────────────┘
```

**加餐知识**：
- **关于用户反馈如何影响缓存**，👉 **详见 `day8_note.md` Q9**
- **关于由于这 5 个 Tab 原生带来了前端性能监控的实现**，👉 **详见 `day8_note.md` Q10**

---

### 6.3 执行轨迹的捕获原理

**BaseAgent 中的日志记录机制**：

```python
# graphrag_agent/agents/base.py
class BaseAgent:
    def __init__(self):
        self.execution_log = []  # 执行轨迹列表

    def _log_execution(self, node_name, input_data, output_data):
        """记录节点执行"""
        self.execution_log.append({
            "node": node_name,
            "timestamp": time.time(),
            "input": input_data,
            "output": output_data
        })

    def ask_with_trace(self, query, thread_id, recursion_limit=5):
        """带执行轨迹的查询"""
        self.execution_log = []  # 重置日志
        # ... 执行 LangGraph 状态图 ...
        return {
            "answer": answer,
            "execution_log": self.execution_log  # 返回完整轨迹
        }
```

**日志记录点**：
1. **缓存检查阶段**：记录缓存命中/未命中及耗时
2. **关键词提取**：记录提取的低级/高级关键词
3. **Agent 节点**：记录 LLM 的工具调用决策
4. **检索节点**：记录检索到的文档数量和内容
5. **生成节点**：记录最终回答生成

**DeepResearch 的特殊日志**：
```python
# 日志格式示例
"[深度研究] 开始第1轮迭代"
"[深度研究] 执行查询: 优秀学生的申请条件"
"[KB检索] 开始搜索: 申请条件"
"[深度研究] 发现有用信息: 根据学生手册第三章..."
"[深度研究] 第1轮迭代完成，继续下一轮"
```

### 6.4 Debug 模式的面试价值

**面试话术**：
"在开发和调试过程中，我们设计了一个完整的 Debug 面板，包含 5 个维度的可观测性：执行轨迹追踪（每个 LangGraph 节点的输入输出）、知识图谱可视化（命中的实体和关系）、源内容展示（检索到的原始文本块）、图谱管理（运行时修改图谱）、以及性能监控（API 调用次数和响应时间趋势）。这个工具帮助我们快速定位了多个 Agent 推理质量问题。"

---

## 七、缓存系统在查询中的完整交互

### 7.1 三级缓存架构

```
查询请求到达
    │
    ▼
┌── 第 1 级：全局缓存（Global Cache）─────────────────┐
│ 键策略：GlobalCacheKeyStrategy（仅基于 query）        │
│ 存储：内存 500 条 + 磁盘 5000 条                      │
│ 作用：跨会话去重，相同问题秒回                         │
│ 命中 → 直接返回（毫秒级）                             │
└────────────────┬──────────────────────────────────────┘
                 │ 未命中
                 ▼
┌── 第 2 级：快速缓存（Fast Cache）──────────────────┐
│ 基于关键词匹配的高速缓存路径                         │
│ 作用：相似问题的快速匹配                             │
│ 命中 → 返回 + 同步写入全局缓存                       │
└────────────────┬──────────────────────────────────────┘
                 │ 未命中
                 ▼
┌── 第 3 级：会话缓存（Session Cache）───────────────┐
│ 键策略：ContextAwareCacheKeyStrategy               │
│        （基于 query + thread_id + 上下文）           │
│ 存储：内存 200 条 + 磁盘 2000 条                     │
│ 作用：同一对话内的上下文相关缓存                      │
│ 命中 → 返回 + 同步写入全局缓存                       │
└────────────────┬──────────────────────────────────────┘
                 │ 全部未命中
                 ▼
        执行 LangGraph 完整流程
                 │
                 ▼
        结果同时写入会话缓存和全局缓存
```

### 7.2 向量语义缓存

除了精确匹配，缓存系统还支持**向量语义匹配**：

```python
# 精确匹配未命中后，尝试向量相似度匹配
# 使用 FAISS 索引
similarity = cosine_similarity(query_embedding, cached_key_embedding)
if similarity > CACHE_SIMILARITY_THRESHOLD:  # 默认 0.9
    return cached_value  # 语义命中
```

**实际效果**：
- "旷课会被退学吗" 和 "旷课多少学时会退学" → 可能语义命中（相似度 > 0.9）
- "奖学金申请条件" 和 "处分规定" → 不会命中（完全不同的问题）

### 7.3 缓存质量控制

```python
# 用户反馈影响缓存质量
# 正向反馈
agent.mark_answer_quality(query, is_positive=True, thread_id)
# → quality_score += 1，标记为 fast_path_eligible

# 负向反馈
agent.clear_cache_for_query(query, thread_id)      # 清除会话缓存
global_cache_manager.delete(query)                  # 清除全局缓存
# → 下次同样的问题会重新执行完整流程
```

---

## 八、流式输出机制深度解析

### 8.1 当前实现：伪流式（Pseudo-Streaming）

**重要事实**：当前所有 Agent 的流式输出都是**伪流式**。

```
真流式：LLM 生成一个 token → 立即发送给前端 → 用户看到逐字出现
伪流式：LLM 完整生成回答 → 按句子切分 → 逐块发送给前端 → 用户看到逐句出现
```

**为什么是伪流式？**
- LangGraph 的 `ToolNode` 必须等工具执行完毕才能继续
- LangChain 版本限制，`generate` 节点的 `astream()` 支持不完善
- 当前实现选择了稳定性优先

### 8.2 伪流式的实现细节

```python
# BaseAgent.ask_stream()
async def ask_stream(self, query, thread_id, recursion_limit=5):
    # 1. 缓存检查（与 ask() 相同）
    cached = self._check_all_caches(query, thread_id)

    if cached:
        # 缓存命中：将完整回答按句子拆分后逐块返回
        sentences = re.split(r'([.!?。！？]\s*)', cached)
        buffer = ""
        for sentence in sentences:
            buffer += sentence
            if len(buffer) >= 50 or is_sentence_end(sentence):
                yield buffer
                buffer = ""
                await asyncio.sleep(0.01)  # 模拟流式效果
    else:
        # 缓存未命中：执行完整流程后切分
        answer = await self._stream_process(query, thread_id)
        # 同样按句子切分后 yield
```

### 8.3 SSE 事件流格式

```
# 前端接收的 SSE 事件序列
data: {"status": "start"}

data: {"status": "token", "content": "根据《学生手册》第三章，"}

data: {"status": "token", "content": "优秀学生的申请条件包括："}

data: {"status": "token", "content": "1. 学业成绩排名前10%..."}

data: {"status": "execution_log", "content": {...}}  // Debug 模式

data: {"status": "done"}
```

### 8.4 面试追问：真流式怎么实现？

**回答要点**：
"要实现真流式，需要重构 `_generate_node` 节点，将其改为 AsyncGenerator。具体来说，需要用 `self.stream_llm.astream()` 替代 `self.llm.invoke()`，在 LLM 生成每个 token 时立即通过 SSE 推送给前端。但这需要 LangGraph 支持节点级别的 streaming，目前的 LangChain 版本（0.2.x）对此支持有限。项目中已经预留了 `self.stream_llm` 实例，等框架升级后可以无缝切换。"

---

## 九、配置调优实验

### 9.1 影响 Agent 性能的关键参数

```bash
# .env 中的关键调优参数

# === Agent 行为控制 ===
AGENT_RECURSION_LIMIT=5              # LangGraph 最大递归次数
AGENT_CHUNK_SIZE=4                   # 流式输出块大小
AGENT_STREAM_FLUSH_THRESHOLD=40      # 普通 Agent 刷新阈值（字符数）
DEEP_AGENT_STREAM_FLUSH_THRESHOLD=80 # DeepResearch 刷新阈值
FUSION_AGENT_STREAM_FLUSH_THRESHOLD=60 # FusionAgent 刷新阈值

# === 搜索参数 ===
SIMILARITY_THRESHOLD=0.9             # 向量相似度阈值
SEARCH_VECTOR_LIMIT=5                # 向量检索 Top-K
NAIVE_SEARCH_TOP_K=3                 # NaiveRAG 检索数量
LOCAL_SEARCH_TOP_CHUNKS=3            # 本地搜索文本块数
LOCAL_SEARCH_TOP_COMMUNITIES=3       # 本地搜索社区数
HYBRID_SEARCH_ENTITY_LIMIT=15        # 混合搜索实体上限

# === 多Agent编排 ===
MA_PLANNER_MAX_TASKS=6               # Planner 最大子任务数
MA_ENABLE_MAPREDUCE=true             # 是否启用 Map-Reduce 长文生成
MA_MAPREDUCE_THRESHOLD=20            # 触发 Map-Reduce 的证据阈值
MA_ENABLE_CONSISTENCY_CHECK=true     # 是否启用一致性校验
MA_SECTION_MAX_EVIDENCE=8            # 每段最大证据数
MA_WORKER_EXECUTION_MODE=sequential  # 任务执行模式（sequential/parallel）
```

### 9.2 实验 1：调整检索数量

```bash
# 实验：增加 NaiveRAG 的检索数量
NAIVE_SEARCH_TOP_K=3   # 默认值
NAIVE_SEARCH_TOP_K=10  # 增大后

# 预期效果：
# - 回答更全面（更多文本块提供更多信息）
# - 但也可能引入噪声（不相关的文本块干扰 LLM 判断）
# - LLM 输入 token 增加 → 费用增加
```

### 9.3 实验 2：调整向量相似度阈值

```bash
# 实验：降低相似度阈值
SIMILARITY_THRESHOLD=0.9   # 默认值（严格）
SIMILARITY_THRESHOLD=0.7   # 降低后（宽松）

# 预期效果：
# - 召回率提升（更多相关实体被检索到）
# - 准确率可能下降（不太相关的实体也被引入）
# - 对于模糊查询效果改善，对于精确查询可能变差
```

### 9.4 实验 3：修改 Entity Schema

```python
# graphrag_agent/config/settings.py
# 原始 Schema
entity_types = ["学生类型", "奖学金类型", "处分类型", "部门", "学生职责", "管理规定"]

# 实验：增加更细粒度的实体类型
entity_types = [
    "学生类型",
    "奖学金类型",
    "处分类型",
    "考试类型",      # 新增
    "课程类型",      # 新增
    "金额数值",      # 新增
    "部门",
    "学生职责",
    "管理规定"
]
```

**注意**：修改 Schema 后需要**重新构建知识图谱**才能生效：
```bash
python graphrag_agent/integrations/build/main.py
```

---

## 十、面试 STAR 话术准备

### Story 1：Agent 问答系统的全链路打通

**S（Situation）**：
知识图谱构建完成后，需要验证整个系统的端到端问答能力，并对比不同 Agent 策略的效果差异。

**T（Task）**：
1. 搭建 FastAPI 后端和 Streamlit 前端的完整服务
2. 设计测试用例，对比 5 种 Agent 的回答质量
3. 利用 Debug 模式分析 Agent 的推理过程
4. 通过配置调优提升问答效果

**A（Action）**：
1. **服务搭建**：启动 FastAPI（uvicorn 多 Worker）+ Streamlit 前端，配置 CORS 跨域
2. **Agent 对比测试**：设计了 4 个梯度难度的测试问题，从简单事实查询到多条件推理
3. **Debug 分析**：通过执行轨迹发现 GraphAgent 的文档质量评估节点会在关键词匹配率不足时触发 fallback 检索
4. **缓存调优**：发现全局缓存对追问场景存在误命中风险（"那上海市的奖学金呢？"会命中错误缓存），提出了意图识别前置路由的优化方案

**R（Result）**：
- 成功对比了 5 种 Agent 的性能差异：NaiveRAG（~3s）→ FusionAgent（~60s）
- 发现 DeepResearchAgent 在多条件推理问题上准确率显著高于 NaiveRAG
- 通过 Debug 面板识别并记录了 3 个可优化点

### Story 2：流式输出与并发控制

**S**：系统需要支持多用户并发问答，同时提供良好的用户体验（减少等待感）。

**T**：理解并验证系统的流式输出机制和并发控制策略。

**A**：
1. 分析了 SSE（Server-Sent Events）流式传输的实现
2. 发现当前是"伪流式"（完整生成后切分），记录了真流式的升级路径
3. 验证了会话锁机制：同一 session 的并发请求返回 429
4. 验证了 AgentManager 的实例隔离：不同 session 互不干扰

**R**：
- 清晰理解了伪流式 vs 真流式的技术差异和框架限制
- 掌握了 RLock + 会话锁的并发控制策略

---

## 十一、今日打卡清单

- [ ] 成功启动 FastAPI 后端和 Streamlit 前端
- [ ] 对同一问题测试至少 3 种 Agent，记录回答差异
- [ ] 开启 Debug 模式，观察至少一个 Agent 的完整执行轨迹
- [ ] 理解请求从前端到 Agent 的完整生命周期（能口述 6 个步骤）
- [ ] 执行一次缓存命中测试（同一问题问两次，观察第二次速度提升）
- [ ] 尝试修改一个 `.env` 参数，观察对回答的影响
- [ ] 填写 Agent 对比实验记录

---

## 十二、Agent 对比实验记录模板

| 测试项 | NaiveRAG | GraphAgent | HybridAgent | DeepResearch | FusionAgent |
|--------|----------|------------|-------------|--------------|-------------|
| 测试问题 | | | | | |
| 回答正确性 | /5 | /5 | /5 | /5 | /5 |
| 回答完整性 | /5 | /5 | /5 | /5 | /5 |
| 执行耗时(秒) | | | | | |
| 回答字符数 | | | | | |
| LLM 调用次数 | | | | | |
| 引用源数量 | | | | | |
| 是否有幻觉 | | | | | |
| 特殊发现 | | | | | |

---

## 十三、下一步：Day 9 预告

Day 9 将进入**面试 STAR 话术打磨**：
- 将 Day 1-8 的所有学习转化为面试叙事
- 准备 3 个 STAR 故事（项目整体、技术亮点、技术难点）
- 准备 7+ 个常见追问的回答
- 练习 2 分钟项目介绍（不看笔记）

---

*学习日期：2026-03-01 | 项目路径：`/home/wkt/project/graph-rag-agent`*
