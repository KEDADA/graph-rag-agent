# Day 3 学习笔记 — Agent 架构：从 LangGraph 到完整问答流程

> 日期：2026-02-25（重构版）
> 
> **阅读前提**：已完成 Day 1-2，理解图构建全流程。本文从零讲解 LangGraph，然后逐步拆解项目中 Agent 的完整工作原理。

---

## 写在前面：Day 3 要解决的核心问题

Day 2 我们把知识图谱建好了。但图谱只是"数据"——用户问了一个问题，系统如何**决定怎么搜、搜完怎么答**？

这就是 **Agent（智能体）** 的工作。项目有 5 种 Agent，但它们的核心逻辑都是同一套：

```
用户问题 → [决策：需不需要搜索？搜哪种？] → [搜索] → [基于搜索结果生成回答] → 返回
```

这套"决策→执行→生成"的流程，项目用了一个框架来实现——**LangGraph**。

所以 Day 3 的学习路径是：

```
① 先学 LangGraph 是什么（工具层面）
② 再看 BaseAgent 怎么用 LangGraph 搭了一个通用骨架
③ 最后看两个具体 Agent（NaiveRagAgent、GraphAgent）怎么"填"骨架
```

---

## 一、LangGraph 入门：用状态图编排 AI 的思考过程

> ❗❗ **延伸阅读**：LangGraph vs LangChain 的区别→见 day3_note_QA.md → QA-1

### 1.1 一句话理解 LangGraph

> **LangGraph = 用"流程图"来控制 LLM 的行为。**

你可以想象画了一张流程图：起点是用户提问，终点是返回答案，中间有若干个"处理站"（节点），每个站做一件事。站与站之间有"路线"（边），有些路线是固定的，有些是根据条件动态选择的。

LangGraph 做的事情就是：**让你用代码把这张流程图画出来，然后它自动按图执行**。

### 1.2 LangGraph 的三个核心概念

| 概念 | 类比 | 在代码中 |
|------|------|---------|
| **节点（Node）** | 流程图中的"处理站" | `workflow.add_node("名称", 处理函数)` |
| **边（Edge）** | 站与站之间的连线 | `workflow.add_edge("A", "B")` 或条件边 |
| **状态（State）** | 在各站之间传递的"文件袋" | 一个字典，所有节点共享读写 |

### 1.3 最小示例：理解执行过程

假设我们要做一个最简单的 AI 助手：用户提问 → LLM 回答。

```python
from langgraph.graph import StateGraph, START, END

# 第 1 步：定义"文件袋"里装什么
class MyState(TypedDict):
    messages: list     # 对话消息列表

# 第 2 步：画流程图
workflow = StateGraph(MyState)

# 添加一个节点：调用 LLM
workflow.add_node("llm", 调用LLM的函数)

# 添加边：起点→llm→终点
workflow.add_edge(START, "llm")
workflow.add_edge("llm", END)

# 第 3 步：编译成可执行的图
graph = workflow.compile()

# 第 4 步：执行
graph.invoke({"messages": [用户的提问]})
# → 自动走 START → llm → END，返回 LLM 的回答
```

这就是 LangGraph 的全部核心思路。接下来的复杂性只是：**多几个节点、加几个条件分支**。

### 1.4 条件边：让 AI 自己选路

真实场景中，LLM 不是每次都用同一条路。比如：

- 用户问"你好" → LLM 直接回答，不需要搜索
- 用户问"旷课退学规定是什么" → LLM 需要先搜索知识库

怎么实现？用**条件边**：

```python
workflow.add_conditional_edges(
    "agent",                    # 从哪个节点出发
    判断函数,                    # 这个函数返回值决定走哪条路
    {
        "tools": "retrieve",   # 返回"tools" → 走 retrieve 节点
        END: END               # 返回 END → 直接结束
    }
)
```

LangGraph 内置了一个判断函数 `tools_condition`，它检查 LLM 的回复中有没有"我要调用工具"的标记（`tool_calls`）。有就走搜索，没有就结束。

### 1.5 状态传递：节点之间怎么通信？

所有节点共享同一个 State 字典。但有个关键设计——**消息是追加的，不是覆盖的**：

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    #                                          ^^^^^^^^^^^
    #                                          这个 add_messages 是"累加器"
```

什么意思？假设 messages 最初是 `[A]`，节点返回 `[B]`，那么：
- **没有 `add_messages`**：messages 变成 `[B]`（覆盖了 A）
- **有 `add_messages`**：messages 变成 `[A, B]`（A 保留，B 追加）

这样每个节点的输入输出都保存在 messages 列表中，后续节点可以回溯查看之前的信息。

> **小结**：LangGraph = 画流程图（节点+边） + 共享状态（messages 列表） + 编译执行。理解了这些，就能看懂项目怎么用它了。

---

## 二、BaseAgent：项目用 LangGraph 搭的通用骨架

> 对应源码：`graphrag_agent/agents/base.py`（781 行）
>
> 不要被行数吓到——大部分是缓存和流式输出的代码。核心骨架只有 `__init__` + `_setup_graph` 两个方法，约 100 行。

### 2.1 先看全貌：BaseAgent 到底做了什么

BaseAgent 是一个**抽象基类**（不能直接使用，只能通过子类使用），它替所有 Agent 统一搞定了以下事情：

```
BaseAgent 统一提供的能力：
  ✅ LangGraph 状态图（决策→搜索→生成的流程框架）
  ✅ 两层缓存系统（避免重复调用 LLM）
  ✅ 流式输出（前端实时展示回答）
  ✅ 对话记忆（同一会话内记住上下文）
  ✅ 性能日志（记录每步耗时）

子类只需要"填空"告诉 BaseAgent：
  ❓ 用什么搜索工具？（_setup_tools）
  ❓ 搜完之后怎么走？（_add_retrieval_edges）
  ❓ 怎么生成回答？（_generate_node）
  ❓ 怎么提取关键词？（_extract_keywords）
```

这就是**模板方法模式**：父类定好流程骨架，子类只实现变化的部分。

> **为什么叫"抽象基类"？**
> 
> ```python
> from abc import ABC, abstractmethod
> 
> class BaseAgent(ABC):   # ABC = Abstract Base Class，Python 标准库
> ```
> 
> `ABC` 的作用：
> 1. 禁止直接 `BaseAgent()` 实例化——你必须写子类
> 2. `@abstractmethod` 标记的方法，子类**必须**实现，否则实例化时 Python 直接报 TypeError
> 3. 类似 Java 的 `abstract class`

### 2.2 初始化：BaseAgent 启动时做了哪些准备

```python
def __init__(self, cache_dir="./cache"):
    # ① 准备两个 LLM
    self.llm = get_llm_model()           # "思考用"的 LLM
    self.stream_llm = get_stream_llm_model()  # "打字机输出用"的 LLM

    # ② 安全限制
    self.default_recursion_limit = 5  # ❗❗ 详解见 day3_note_QA.md → QA-7
    #   ↑ LangGraph 状态图最多走几步（不是 LLM 调用次数！）
    #   正常 agent→retrieve→generate 只走 3 步
    #   设 5 是防止 agent→retrieve→agent→retrieve→... 的死循环

    # ③ 对话记忆  ❗❗ 详解见 day3_note_QA.md → QA-10
    self.memory = MemorySaver()
    #   ↑ LangGraph 的 checkpointer，按 thread_id 保存每个会话的消息历史
    #   注意：仅内存存储，服务重启后丢失  ❗❗ 持久化方案见 day3_note_QA.md → QA-2

    # ④ 两层缓存（后面 §四 详细讲）
    self.cache_manager = CacheManager(...)        # 会话级
    self.global_cache_manager = CacheManager(...)  # 全局级

    # ⑤ 构建 LangGraph 状态图
    self.tools = self._setup_tools()   # 子类告诉我用什么工具
    self._setup_graph()                # 画流程图
```

**为什么需要两个 LLM？** ❗❗ `bind_tools` **详解见 day3_note_QA.md → QA-11**

| | `self.llm` | `self.stream_llm` |
|---|---|---|
| 用途 | 决策（要不要调工具）+ 生成回答 | 前端流式展示 |
| 调用方式 | `.invoke()` 整体返回 | `.astream()` 逐 token 返回 |
| 为什么分开 | `bind_tools()` 要求 LLM 返回完整的 JSON（tool\_calls），流式模式下 JSON 逐字返回时会解析失败 |

### 2.3 `_setup_graph`：画流程图（最关键的 30 行代码）

```python
def _setup_graph(self):
    # ── 定义 State ──
    class AgentState(TypedDict):
        messages: Annotated[Sequence[BaseMessage], add_messages]
        # 这行拆开看：
        #   Sequence[BaseMessage]
        #     └ Sequence = 有序序列（类似 list），来自 Python 的 typing 模块
        #     └ BaseMessage = LangChain 的消息基类（HumanMessage、AIMessage 等的父类）
        #     └ 合起来 = "messages 是一个 BaseMessage 的列表"
        #
        #   Annotated[..., add_messages]
        #     └ Annotated = Python 3.9+ 的类型注解，给类型附加额外信息
        #     └ add_messages = LangGraph 的 reducer 函数
        #     └ 合起来 = "告诉 LangGraph：更新 messages 时用 add_messages 策略（追加而不是覆盖）"
        #
        # 最终含义：messages 是一个消息列表，每个节点的返回值会被【追加】到列表末尾

    # ── 画图 ──
    workflow = StateGraph(AgentState)

    #   三个节点
    workflow.add_node("agent", self._agent_node)        # 节点1: LLM 决策
    workflow.add_node("retrieve", ToolNode(self.tools))  # 节点2: 执行搜索工具
    workflow.add_node("generate", self._generate_node)   # 节点3: 生成回答

    #   连线
    workflow.add_edge(START, "agent")         # 起点 → 节点1
    workflow.add_conditional_edges(           # 节点1 → 条件分支
        "agent", tools_condition,
        {"tools": "retrieve", END: END}
    )
    self._add_retrieval_edges(workflow)       # 节点2 → ???（子类决定）
    workflow.add_edge("generate", END)        # 节点3 → 终点

    # ── 编译 ──
    self.graph = workflow.compile(checkpointer=self.memory)
```

画出来的流程图：

```
                    ┌───────────────────────────────────┐
                    │          BaseAgent 状态图           │
                    │                                    │
                    │   START                            │
                    │     │                              │
                    │     ▼                              │
                    │  ┌────────┐                        │
                    │  │ agent  │  ← LLM 决策节点        │
                    │  └───┬────┘                        │
                    │      │                             │
                    │      ├── LLM 说"需要搜索"           │
                    │      │   （返回 tool_calls）        │
                    │      │         │                   │
                    │      │         ▼                   │
                    │      │   ┌──────────┐              │
                    │      │   │ retrieve │  ← 执行工具   │
                    │      │   └────┬─────┘              │
                    │      │        │                    │
                    │      │        ▼                    │
                    │      │   ┌──────────┐              │
                    │      │   │ generate │  ← 生成回答   │
                    │      │   └────┬─────┘              │
                    │      │        │                    │
                    │      │        ▼                    │
                    │      │      END                    │
                    │      │                             │
                    │      └── LLM 说"我直接答"           │
                    │          （无 tool_calls）          │
                    │               │                    │
                    │               ▼                    │
                    │             END                    │
                    └───────────────────────────────────┘
```

> **retrieve → generate 中间的连线（`_add_retrieval_edges`）为什么是子类决定？** 因为不同 Agent 的策略不同——NaiveRagAgent 直接连过去，GraphAgent 中间还要根据"用的是本地搜索还是全局搜索"做条件路由。这是唯一需要子类自定义的流程部分。

---

## 三、一个问题的完整旅程（以 GraphAgent 为例）

理解了流程图，现在让我们跟踪一个真实问题，看它是怎么在状态图中一步步被处理的。

**问题**：*"旷课多少学时会被退学？"*

### Step 1：进入 `agent` 节点 —— LLM 做决策

```python
def _agent_node(self, state):
    messages = state["messages"]
    # 此时 messages = [HumanMessage("旷课多少学时会被退学？")]

    # 1) 提取关键词（子类实现） ❗❗ 为什么在这里提取？见 day3_note_QA.md → QA-8
    query = messages[-1].content
    keywords = self._extract_keywords(query)
    # → {"low_level": ["旷课", "学时", "退学"], "high_level": ["学籍管理"]}

    # 2) 把关键词"藏"进消息的 metadata 中（后续节点可以读取）
    enhanced_message = HumanMessage(
        content=query,
        additional_kwargs={"keywords": keywords}
    )

    # 3) 让 LLM 做决定：要不要调用搜索工具？
    model = self.llm.bind_tools(self.tools)  # ❗❗ 详解见 day3_note_QA.md → QA-3 和 QA-11
    #       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    #       bind_tools 的作用：
    #       把工具的"使用说明"注入 LLM 的 system prompt，例如：
    #       "你有以下工具可用：
    #        1. local_retriever: 用于查找具体规定和条款...
    #        2. global_retriever: 用于总结归纳整体框架..."
    #       LLM 根据问题和工具描述，自主决定用哪个（或不用）

    response = model.invoke(messages)
    # → LLM 返回：AIMessage(tool_calls=[{name: "local_retriever", args: {query: "旷课学时退学"}}])
    #   意思是："这个问题问的是具体规定，我要用本地搜索"

    return {"messages": [response]}
    # → messages 变成 [HumanMessage, AIMessage(tool_calls=[...])]
```

**此时 messages 列表**：
```
[0] HumanMessage("旷课多少学时会被退学？")         ← 用户问题
[1] AIMessage(tool_calls=[{name: "local_retriever", ...}])  ← LLM 的决策
```

### Step 2：`tools_condition` 条件路由

LangGraph 检查 `messages[-1]`（即 LLM 的返回）：
- 有 `tool_calls` → 走 `retrieve` 节点 ✅
- 没有 → 走 `END`

### Step 3：进入 `retrieve` 节点 —— 执行搜索

```python
# ToolNode 是 LangGraph 的预建节点，自动完成：
# 1. 读 messages[-1] 中的 tool_calls → 发现要调 local_retriever
# 2. 找到 LocalSearchTool，执行 search("旷课学时退学")
# 3. 把搜索结果包装成 ToolMessage 追加到 messages
```

搜索过程（Local Search，Day 4 会详细学）：
```
"旷课学时退学" → 向量检索匹配实体 → 找到实体"退学"
→ 沿图谱扩展 1-2 跳邻居 → 得到相关的 Chunk 和实体描述
→ 拼装成一段文本上下文
```

**此时 messages 列表**：
```
[0] HumanMessage("旷课多少学时会被退学？")                    ← 用户问题
[1] AIMessage(tool_calls=[{name: "local_retriever", ...}])     ← LLM 决策
[2] ToolMessage("根据《华东理工大学本科生学籍管理办法》第X条...") ← 搜索结果
```

### Step 4：路由判断（GraphAgent 特有）

GraphAgent 在 `retrieve → generate` 之间加了一个条件路由 `_grade_documents`：

```python
def _grade_documents(self, state):
    messages = state["messages"]

    # 检查 Step 1 中 LLM 调用的是哪个工具
    tool_name = messages[-2].tool_calls[0]["function"]["name"]

    if tool_name == "global_retriever":
        return "reduce"    # → 走 reduce 节点
    else:
        return "generate"  # → 走 generate 节点
```

> **为什么全局搜索要走 `reduce` 而不是直接 `generate`？**
>
> 关键在于**两种搜索返回的结果格式不同**：
>
> - **本地搜索（local）** 返回的是：一段拼接好的文本（实体描述+相关 Chunk），可以直接当 context 塞给 LLM 生成回答
> - **全局搜索（global）** 返回的是：**多个社区的独立摘要**（每个社区是图谱中一组紧密关联的实体的总结）
>
> 比如问"学校的学生管理制度整体框架？"，全局搜索可能返回：
> ```
> 社区1摘要："奖学金管理方面，学校设有国家奖学金、上海市奖学金..."
> 社区2摘要："纪律处分方面，学生违反校规可能受到警告、严重警告..."
> 社区3摘要："学籍管理方面，学生入学后需注册学籍..."
> ```
>
> 这些摘要之间**互相独立、可能有重复**，不能直接拼在一起。`reduce` 节点的工作就是把这些散碎的社区摘要**合并成一个连贯的、去重的完整回答**——这就是 Map-Reduce 中的 Reduce 步骤。

本例中用的是 `local_retriever` → 走 `generate`。

### Step 5：进入 `generate` 节点 —— 生成回答

```python
def _generate_node(self, state):
    messages = state["messages"]
    question = messages[-3].content   # ❗❗ 为什么是 [-3]？见 day3_note_QA.md → QA-4
    docs = messages[-1].content       # → 搜索结果文本（第2条）

    # 构建 RAG Prompt
    prompt = ChatPromptTemplate.from_messages([
        ("system", "你是华东理工大学的学生事务助手，请基于以下资料回答问题..."),
        ("human", "资料：{context}\n\n问题：{question}"),
    ])

    # Prompt → LLM → 字符串
    rag_chain = prompt | self.llm | StrOutputParser()
    response = rag_chain.invoke({
        "context": docs,
        "question": question,
    })
    # → "根据华东理工大学的相关规定，一学期累计旷课超过该学期教学计划总学时三分之一者，应予退学处理。"

    return {"messages": [AIMessage(content=response)]}
```

### Step 6：到达 END → 返回给用户

**最终 messages 列表**：
```
[0] HumanMessage("旷课多少学时会被退学？")                           ← 用户问题
[1] AIMessage(tool_calls=[{name: "local_retriever", ...}])            ← LLM 决策
[2] ToolMessage("根据《华东理工大学本科生学籍管理办法》第X条...")        ← 搜索结果
[3] AIMessage("根据华东理工大学的相关规定，一学期累计旷课超过...")      ← 最终回答
```

`ask()` 方法从 `messages[-1].content` 取出最终回答，返回给用户。

> **核心理解**：整个过程就是消息列表不断"变长"。每个节点读前面的消息，做处理，追加新消息。最终回答就是列表中最后一条 AIMessage。这就是 LangGraph 状态图的全部执行逻辑。

---

## 四、NaiveRagAgent vs GraphAgent：对比理解

有了"一个问题的旅程"作为基础，现在对比两个 Agent 的差异就很清晰了。

### 4.1 NaiveRagAgent（167 行）—— 最简单的基线

```python
class NaiveRagAgent(BaseAgent):
    def __init__(self):
        self.search_tool = NaiveSearchTool()  # 唯一的工具：纯 Chunk 向量搜索
        super().__init__(cache_dir="./cache/naive_agent")
```

**它"填"了 BaseAgent 的哪些空？**

| 抽象方法 | NaiveRagAgent 的实现 | 效果 |
|---------|---------------------|------|
| `_setup_tools()` | `[NaiveSearchTool]` | 只有 1 个工具 |
| `_add_retrieval_edges()` | `retrieve → generate`（直连） | 无条件分支 |
| `_extract_keywords()` | 返回空列表 | 不做关键词提取 |
| `_generate_node()` | 标准 RAG Chain | context + question → LLM |

**NaiveRagAgent 的状态图（最简）**：

```
START → agent → retrieve → generate → END
                    ↑           
          NaiveSearchTool  
        （在 Chunk 向量索引上做余弦相似度搜索）
```

**搜索原理**：直接把用户问题转成向量，在 Chunk 向量索引中找 top-K 个最相似的文本块，拼在一起作为 context。**不经过知识图谱**。

### 4.2 GraphAgent（528 行）—— 图结构 Agent

```python
class GraphAgent(BaseAgent):
    def __init__(self):
        self.local_tool = LocalSearchTool()   # 工具1：图谱本地搜索
        self.global_tool = GlobalSearchTool()  # 工具2：社区摘要全局搜索
        super().__init__(cache_dir="./cache/graph_agent")
```

**它"填"了 BaseAgent 的哪些空？**

| 抽象方法 | GraphAgent 的实现 | 效果 |
|---------|-------------------|------|
| `_setup_tools()` | `[LocalSearchTool, GlobalSearchTool]` | 2 个工具，LLM 自主选择 |
| `_add_retrieval_edges()` | 条件路由（grade_documents） | 根据工具类型走不同分支 |
| `_extract_keywords()` | 调用 LLM 提取 low\_level + high\_level 关键词 | 增强搜索和缓存精度 |
| `_generate_node()` | 同上，但 Prompt 针对图搜索结果优化 | —— |

**GraphAgent 的状态图**：

```
START → agent → retrieve ─┬─ local搜索 → generate → END
                          │
                          └─ global搜索 → reduce → END
                                           ↑
                                    Map-Reduce 聚合
                                 （多个社区摘要 → 合并成统一回答）
```

**GraphAgent 多了什么？**

1. **两个搜索工具**：LLM 在 agent 节点根据问题类型自主选择
   - "旷课退学规定" → 具体细节 → `local_retriever`
   - "学校管理制度整体框架" → 宏观概括 → `global_retriever`

2. **条件路由 `_grade_documents`**：retrieve 之后判断用的是哪个工具
   - 本地搜索 → 直接走 `generate`
   - 全局搜索 → 走 `reduce`（因为全局搜索返回的是多个社区的摘要，需要再做一次 Map-Reduce 合并）

3. **`reduce` 节点**：将全局搜索返回的多段社区摘要汇总成一个连贯的回答

4. **LLM 关键词提取**：提取 `low_level`（具体实体词）和 `high_level`（抽象概念），用于增强搜索查询和缓存键

### 4.3 一张表总结差异

| 维度 | NaiveRagAgent | GraphAgent |
|------|--------------|------------|
| 搜索工具 | 1 个（Chunk 向量搜索） | 2 个（本地图搜索 + 全局搜索） |
| 使用知识图谱？ | ❌ 不用 | ✅ 核心依赖 |
| retrieve 后的路由 | 直连 generate | 条件分支（generate / reduce） |
| 关键词提取 | 不做 | LLM 提取两级关键词 |
| 适合的问题类型 | "XX 规定是什么" | 需要关系推理 / 全局概括的复杂问题 |
| 代码量 | 167 行 | 528 行 |
| 角色 | baseline 基线 | **项目主力 Agent** |

---

## 五、两层缓存：为什么要缓存？怎么缓存？

### 5.1 为什么需要缓存

每次回答一个问题，系统要：
1. 调用 LLM 提取关键词（1 次 API 调用）
2. 调用 LLM 做工具决策（1 次）
3. 执行图搜索（数据库查询）
4. 调用 LLM 生成回答（1 次）

如果多个用户问了同一个问题，每次都走完 3 次 LLM 调用，既浪费钱又慢。**缓存就是"记住"之前回答过的问题，下次直接返回。**

### 5.2 为什么要两层，不能一层搞定？

```
场景1：用户A 在会话中先问"国家奖学金申请条件"，再问"那上海市奖学金呢？"
场景2：用户B（独立会话）也问了"国家奖学金申请条件"
```

- **"那上海市奖学金呢？"** 这个问题的含义取决于上一轮对话——如果用全局缓存，另一个也问"那xxx呢"的用户会拿到错误答案
- **"国家奖学金申请条件"** 则是独立问题，任何人问都该返回相同答案

因此需要两层：

| | 会话缓存 | 全局缓存 |
|---|---|---|
| 作用域 | 同一对话内（`thread_id` 隔离） | 所有用户共享 |
| 键生成 | `thread_id + query + keywords` 三者联合哈希 | 仅 `query` 的 MD5 |
| 解决问题 | 同一对话中的追问和反复确认 | 不同用户问的高频相同问题 |
| 内存/磁盘 | 200条/2000条 | 500条/5000条 |

### 5.3 查询时的缓存检查顺序

```python
def _check_all_caches(self, query, thread_id):
    # ① 先查全局缓存（命中率最高、最便宜）
    result = self.global_cache_manager.get(query)
    if result: return result

    # ② 再查快速路径缓存
    result = self.check_fast_cache(query, thread_id)
    if result: return result

    # ③ 最后查会话缓存
    result = self.cache_manager.get(query, thread_id=thread_id)
    if result: return result

    # ④ 全部未命中 → 执行 LLM 推理
    return None
```

### 5.4 写入时同时更新两层

```python
# LLM 推理完成后
if answer and len(answer) > 10:    # 过滤太短的垃圾回答
    self.cache_manager.set(query, answer, thread_id=thread_id)   # 写会话缓存
    self.global_cache_manager.set(query, answer)                 # 写全局缓存
```

### 5.5 语义缓存：改了措辞也能命中  ❗❗ 详解见 day3_note_QA.md → QA-5、QA-6

普通缓存是精确匹配——"国家奖学金申请条件"和"怎么申请国家奖学金"会被当成两个不同问题。

项目用 **SentenceTransformer**（本地小模型，`all-MiniLM-L6-v2`）做向量语义匹配：

```
缓存中已有："国家奖学金的申请条件是什么？"
新查询：    "怎么申请国家奖学金？"
→ 语义相似度 = 0.92（≥ 阈值 0.9）→ 命中！直接返回缓存结果
```

阈值为什么设 0.9（很高）？因为"国家奖学金申请条件"和"国家**助**学金申请条件"的相似度可能是 0.88，但答案完全不同。宁可多调一次 LLM，也不返回错误答案。

### 5.6 MemorySaver vs CacheManager — 容易混淆的两个概念

| | MemorySaver | CacheManager |
|---|---|---|
| 来自 | LangGraph 框架 | 项目自研 |
| 保存什么 | 完整消息列表 `[HumanMsg, AIMsg, ToolMsg, ...]` | 问答映射 `{query → answer}` |
| 用途 | 恢复对话上下文（多轮对话） | 跳过重复的 LLM 推理 |
| 按什么隔离 | `thread_id` | 会话缓存按 `thread_id`，全局缓存不隔离 |
| 持久化 | ❌ 纯内存，重启丢失 | ✅ 磁盘 pickle 文件持久化 |

---

## 六、流式输出：前端怎么实时显示回答

### 6.1 当前实现——"伪流式"

项目的流式输出**不是**真正的逐 token 输出（LLM 一个字一个字返回），而是：

```
LLM 先完整生成整个回答
→ 按句子切分
→ 攒够 threshold 个字符后推送一批给前端
→ 前端呈现"打字机"效果
```

```python
# 伪代码
response = llm.invoke(...)              # 先完整生成
sentences = split_by_punctuation(response)  # 按 。！？ 切分
buffer = ""
for sentence in sentences:
    buffer += sentence
    if len(buffer) >= 40:               # stream_flush_threshold = 40
        yield buffer                    # 推送给前端
        buffer = ""
        await asyncio.sleep(0.01)       # 微延迟，模拟流畅感
```

### 6.2 `stream_flush_threshold` 是什么

控制**每攒够多少字符才推送一次**给前端：

- `threshold = 40`（普通 Agent）：推送频繁，像打字机
- `threshold = 80`（DeepResearch）：推送较慢但每次内容更完整

### 6.3 三种调用方式对比  ❗❗ 详解见 day3_note_QA.md → QA-9

| 方法 | 返回值 | 适用场景 |
|------|--------|---------|
| `ask(query)` | `str`（完整回答） | 后端 API / 测试脚本 |
| `ask_stream(query)` | `AsyncGenerator[str]`（流式字符块） | 前端 Streamlit 实时展示 |
| `ask_with_trace(query)` | `{answer, execution_log}`（回答+每步日志） | Debug 模式 |

---

## 七、面试要点总结

### STAR 片段：Agent 架构设计

**S**：项目需要支持多种检索策略（纯向量/本地图搜索/全局搜索/深度研究），不同策略的处理流程不同。

**T**：设计一套可扩展的 Agent 架构，新增策略时不需要修改核心代码。

**A**：
1. 使用 **LangGraph StateGraph** 构建三节点状态机（agent→retrieve→generate），将 LLM 决策、工具执行、回答生成解耦为独立节点
2. 设计 **BaseAgent 抽象基类**（模板方法模式），统一提供缓存/流式/日志基础设施，子类只需实现 4 个抽象方法
3. 实现**两层语义缓存**（会话级+全局级+SentenceTransformer 向量匹配），避免重复 LLM 调用

**R**：成功支撑 5 种 Agent 复用同一套基础设施，新增 Agent 只需约 150 行代码

### 高频面试问答

| 问题 | 要点 |
|------|------|
| 为什么选 LangGraph 而不是手写？ | 原生支持条件路由、状态持久化、递归保护，比手写 if-else 更健壮 |
| `tools_condition` 怎么判断要不要调工具？ | 检查 LLM 返回的 AIMessage 里有没有 `tool_calls` 字段 |
| `bind_tools()` 做了什么？ | 把工具名称+描述+参数 Schema 注入 LLM 的 prompt，让 LLM 通过 Function Calling 协议返回结构化的工具调用指令 |
| 两层缓存为什么要分开？ | 会话缓存按 `thread_id` 隔离上下文相关的回答，全局缓存跨会话复用高频独立问题 |
| MemorySaver 和 CacheManager 区别？ | MemorySaver 保存对话状态轨迹（messages列表），CacheManager 保存问答结果映射（query→answer） |
| 流式输出是真流式吗？ | 当前是"伪流式"——LLM 完整生成后按句子切片推送。受限于 `bind_tools` 模式下的流式不稳定 |
| `recursion_limit` 是什么？ | LangGraph 状态图的节点执行步数上限（默认5），防止 agent↔retrieve 的死循环 |

---

## 八、自测清单

- [ ] 能用自己的话解释 LangGraph 的三个核心概念（节点、边、状态）
- [ ] 能在白纸上画出 BaseAgent 的状态图（3 节点 + 条件边）
- [ ] 能画出 GraphAgent 的状态图（多了 reduce 节点和 grade_documents 路由）
- [ ] 能跟踪一个问题在 messages 列表中的完整旅程（4 条消息的含义）
- [ ] 能解释两层缓存的设计动机（举出"那上海市奖学金呢？"这个反例）
- [ ] 能区分 MemorySaver 和 CacheManager 的不同职责

---

## 九、Day 3 涉及的代码文件索引

| 文件 | 核心类 | 行数 | 本文对应章节 |
|------|-------|------|------------|
| `agents/base.py` | `BaseAgent` | 781 | §二、§三、§五、§六 |
| `agents/naive_rag_agent.py` | `NaiveRagAgent` | 167 | §四 |
| `agents/graph_agent.py` | `GraphAgent` | 528 | §三、§四 |
| `config/settings.py` | `AGENT_SETTINGS` | 351 | §二 |
| `cache_manager/manager.py` | `CacheManager` | 395 | §五 |
| `cache_manager/strategies/global_strategy.py` | `GlobalCacheKeyStrategy` | 25 | §五 |

---

*完成日期：2026-02-25 | 对应计划：Day 3 — Agent 基础架构深挖*
