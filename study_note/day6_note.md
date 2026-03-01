# Day 6 学习笔记 — FusionAgent 多智能体架构：Plan-Execute-Report 三阶段编排

> 日期：2026-02-28
>
> **阅读前提**：已完成 Day 1-5，理解图构建、Agent 基础、搜索策略和质量保证机制。

---

## 写在前面：Day 6 要解决的核心问题

前5天我们学会了如何构建知识图谱、如何用单个 Agent 进行推理、如何在图上搜索。但有一个关键问题一直没有解决：

**如何处理复杂的、需要多步推理的问题？**

想象一下这些场景：
- 用户问："请详细分析学生资助体系的完整结构，包括各类奖学金的申请条件、评选流程和互斥关系"
- 单个 Agent 的问题：
  - 上下文窗口有限，无法一次性处理所有信息
  - 需要先检索基础信息，再深度研究，最后生成结构化报告
  - 不同子任务之间有依赖关系（必须先完成 A 才能做 B）

Day 6 就是要解决这个问题：

```
单Agent（如GraphAgent）：问题 → 一次性检索 → 生成回答（受限于上下文长度）
                          ↓
FusionAgent（多智能体）：问题 → 规划分解 → 并行执行 → 汇总报告（支持长文档）
```

---

## 一、架构总览：Plan-Execute-Report 三阶段

### 1.1 为什么需要多智能体？

**核心矛盾**：
- LLM 的上下文窗口有限（即使是 GPT-4，也只有 128K tokens）
- 复杂问题需要大量证据（可能涉及数百个实体、数千条关系）
- 单次调用无法完成"检索 → 分析 → 推理 → 报告"的完整流程

👉 **[高阶追问] 面试官可能会让你对比技术选型：“为什么不用 LangChain 的 AgentExecutor、AutoGPT、或者 Crew AI？” 详见 `day6_note_QA.md` 里的 QA-22、QA-23、QA-24 深度对比。**

**解决方案**：
将复杂任务分解成多个子任务，由不同的"专家 Agent"并行或串行执行，最后汇总结果。

---

### 1.2 三阶段架构图

```
┌─────────────────────────────────────────────────────────────┐
│  Phase 1: Plan（规划阶段）                                   │
│  输入: 用户原始问题                                          │
│  流程: Clarifier → TaskDecomposer → PlanReviewer            │
│  输出: PlanSpec（结构化任务图 + 依赖关系）                   │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  Phase 2: Execute（执行阶段）                                │
│  输入: PlanSpec                                              │
│  流程: WorkerCoordinator 调度 Executor 执行任务              │
│  - RetrievalExecutor: 检索类任务（local/global/chain）      │
│  - ResearchExecutor: 深度研究任务                            │
│  - ReflectionExecutor: 反思与验证任务                        │
│  输出: ExecutionRecord 列表（包含证据和中间结果）            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  Phase 3: Report（报告阶段）                                 │
│  输入: ExecutionRecord 列表                                  │
│  流程: OutlineBuilder → SectionWriter → ConsistencyChecker  │
│  - OutlineBuilder: 生成报告大纲                              │
│  - SectionWriter: Map-Reduce 并行写作各章节                 │
│  - ConsistencyChecker: 事实一致性校验                        │
│  输出: 结构化长文档报告（支持 5000+ 字）                     │
└─────────────────────────────────────────────────────────────┘
```

---

### 1.3 核心数据结构速览

在深入每个阶段之前，先理解三个关键数据结构：

| 数据结构 | 作用 | 关键字段 |
|---------|------|---------|
| **PlanSpec** | 规划阶段的输出，完整的任务执行计划 | `task_graph`（任务图）、`acceptance_criteria`（验收标准） |
| **TaskNode** | 任务图中的单个任务 | `task_id`、`task_type`、`depends_on`（依赖列表） |
| **ExecutionRecord** | 执行阶段的输出，单个任务的执行记录 | `tool_calls`（工具调用）、`evidence`（证据列表） |

---

## 二、Phase 1: Plan（规划阶段）— 从问题到任务图

### 2.1 规划阶段的三步流水线

```
用户问题 → Clarifier（澄清）→ TaskDecomposer（分解）→ PlanReviewer（审校）→ PlanSpec
```

---

### 2.2 Step 1: Clarifier（澄清节点）

**文件位置**：`graphrag_agent/agents/multi_agent/planner/clarifier.py`（105行）

#### 核心功能

识别用户问题中的模糊点，生成澄清问题。

**举例**：
```
用户问题: "学生资助有哪些？"

Clarifier 分析:
- 模糊点1: "学生资助"是指奖学金、助学金、还是包括勤工俭学？
- 模糊点2: 是否需要包括申请条件和流程？

输出:
{
  "needs_clarification": true,
  "questions": [
    "您想了解的学生资助是否包括勤工俭学和助学贷款？",
    "是否需要详细的申请条件和评选流程？"
  ],
  "ambiguity_types": ["scope_unclear", "detail_level_unclear"]
}
```

#### 代码实现（第70-90行）

```python
def analyze(self, context: PlanContext) -> ClarificationResult:
    prompt = CLARIFY_PROMPT.format(
        query=context.refined_query or context.original_query,
        domain=context.domain_context or self._default_domain,
    )

    response = self._invoke_llm(prompt)
    parsed = self._parse_response(response)
    result = ClarificationResult(**parsed, raw_response=response)
    return result
```

**关键设计**：
- 使用 LLM 分析问题的模糊性（而不是硬编码规则）
- 输出结构化的 `ClarificationResult`，包含 `needs_clarification` 标志
- 如果需要澄清，系统会暂停并等待用户回答

> 👉 **面试必问：这个是意图识别吗？** 参考 `day6_note_QA.md` 中的 **[QA-6 追问：Clarifier属于“意图识别”吗？]**

---

### 2.3 Step 2: TaskDecomposer（任务分解节点）

**文件位置**：`graphrag_agent/agents/multi_agent/planner/task_decomposer.py`（138行）

#### 核心功能

将清晰的查询拆解为结构化的任务图（TaskGraph）。

**举例**：
```
澄清后的问题: "详细分析学生资助体系，包括国家奖学金、国家励志奖学金和助学金的申请条件、评选流程和互斥关系"

TaskDecomposer 输出:
{
  "nodes": [
    {
      "task_id": "task_001",
      "task_type": "local_search",
      "description": "检索国家奖学金的基本信息和申请条件",
      "depends_on": [],
      "priority": 1
    },
    {
      "task_id": "task_002",
      "task_type": "local_search",
      "description": "检索国家励志奖学金的基本信息和申请条件",
      "depends_on": [],
      "priority": 1
    },
    {
      "task_id": "task_003",
      "task_type": "chain_exploration",
      "description": "探索三类资助之间的互斥关系",
      "depends_on": ["task_001", "task_002"],
      "priority": 2
    }
  ],
  "execution_mode": "parallel"
}
```

#### 代码实现（第50-74行）

```python
def decompose(self, query: str) -> TaskDecompositionResult:
    prompt = TASK_DECOMPOSE_PROMPT.format(
        query=query,
        max_tasks=self._max_tasks,
    )

    response = self._invoke_llm(prompt)
    parsed = self._parse_response(response)
    task_graph = self._build_task_graph(parsed)

    return TaskDecompositionResult(
        task_graph=task_graph,
        raw_task_graph=parsed,
        raw_response=response,
    )
```

**关键设计**：
- LLM 生成任务列表，每个任务包含 `task_type`（检索类型）和 `depends_on`（依赖关系）
- `_build_task_graph` 会清洗和验证任务图（第89-137行）：
  - 自动补充缺失字段（`priority`、`estimated_tokens`）
  - 规范化任务类型（无法识别的映射到 `custom`）
  - 确保依赖字段为列表

---


### 2.4 Step 3: PlanReviewer（计划审校节点）

**文件位置**：`graphrag_agent/agents/multi_agent/planner/plan_reviewer.py`（164行）

#### 核心功能

对任务图执行审校，生成完整的 `PlanSpec`（包含验收标准、预估耗时等）。

**审校内容**：
1. **依赖关系检查**：是否存在循环依赖？
2. **任务合理性**：任务类型是否匹配描述？
3. **资源预估**：预估总 token 消耗和执行时间
4. **验收标准**：定义任务完成的标准（如最小证据数量、最低置信度）

#### 代码实现（第60-130行）

```python
def review(
    self,
    *,
    original_query: str,
    refined_query: Optional[str],
    task_graph: TaskGraph,
    assumptions: list[str],
    background_info: Optional[str] = None,
    user_intent: Optional[str] = None,
) -> PlanReviewOutcome:
    # 1. 构建审校提示
    task_graph_json = json.dumps(task_graph.to_dict(), ensure_ascii=False, indent=2)
    prompt = PLAN_REVIEW_PROMPT.format(
        query=original_query,
        refined_query=refined_query or original_query,
        task_graph=task_graph_json,
        assumptions=assumptions_text,
    )

    # 2. 调用 LLM 审校
    response = self._invoke_llm(prompt)
    parsed = self._parse_response(response)

    # 3. 构建 PlanSpec
    plan_spec = PlanSpec(
        problem_statement=ProblemStatement(**problem_statement),
        assumptions=assumptions,
        task_graph=reviewed_task_graph,
        acceptance_criteria=AcceptanceCriteria(**acceptance_data),
        status="draft",
    )

    # 4. 验证计划合法性
    try:
        plan_spec.validate()
    except ValueError as exc:
        validation.is_valid = False
        validation.issues.append(str(exc))

    return PlanReviewOutcome(
        plan_spec=plan_spec,
        validation=validation,
        reviewed_task_graph=reviewed_task_graph,
        extra_data=extra_data,
    )
```

**关键设计**：
- LLM 可以修改任务图（如调整优先级、添加缺失任务）
- 如果 LLM 返回的任务图无效，回退到原始图（第146-163行）
- 输出 `PlanValidationResult`，包含 `is_valid`、`issues`、`suggestions`

---

### 2.5 核心数据结构：PlanSpec

**文件位置**：`graphrag_agent/agents/multi_agent/core/plan_spec.py`（420行）

#### PlanSpec 的完整结构

```python
class PlanSpec(BaseModel):
    plan_id: str                              # 计划唯一标识
    version: int                              # 版本号
    problem_statement: ProblemStatement       # 问题陈述
    assumptions: List[str]                    # 假设条件
    task_graph: TaskGraph                     # 任务依赖图
    acceptance_criteria: AcceptanceCriteria   # 验收标准
    status: Literal["draft", "approved", "executing", "completed", "failed"]
```

#### TaskGraph 的核心方法

**1. 依赖关系验证（第140-184行）**

```python
def validate_dependencies(self) -> bool:
    # 检查1: 依赖的任务ID是否存在
    for node in self.nodes:
        for dep_id in node.depends_on:
            if dep_id not in task_id_set:
                raise ValueError(f"任务 {node.task_id} 依赖的任务 {dep_id} 不存在")

    # 检查2: 是否存在循环依赖（深度优先搜索）
    def has_cycle(task_id: str) -> bool:
        visited.add(task_id)
        rec_stack.add(task_id)
        current_node = next((n for n in self.nodes if n.task_id == task_id), None)
        for dep_id in current_node.depends_on:
            if dep_id not in visited:
                if has_cycle(dep_id):
                    return True
            elif dep_id in rec_stack:
                return True
        rec_stack.remove(task_id)
        return False

    for node in self.nodes:
        if node.task_id not in visited:
            if has_cycle(node.task_id):
                raise ValueError("任务图中存在循环依赖")
    return True
```

**2. 拓扑排序（第227-262行）**

```python
def topological_sort(self) -> List[TaskNode]:
    """
    获取任务的拓扑排序（Kahn 算法）

    返回按依赖顺序排列的任务节点列表
    """
    task_map = {node.task_id: node for node in self.nodes}
    in_degree: Dict[str, int] = {node.task_id: 0 for node in self.nodes}
    adjacency: Dict[str, List[str]] = defaultdict(list)

    # 构建邻接表和入度表
    for node in self.nodes:
        for dep_id in node.depends_on:
            adjacency[dep_id].append(node.task_id)
            in_degree[node.task_id] += 1

    # 初始化队列（入度为0的任务）
    queue = deque(sorted(
        (task_map[task_id] for task_id, degree in in_degree.items() if degree == 0),
        key=lambda x: (x.priority, x.task_id),
    ))

    ordered_nodes: List[TaskNode] = []
    while queue:
        current = queue.popleft()
        ordered_nodes.append(current)

        # 更新邻居的入度
        for neighbor_id in adjacency.get(current.task_id, []):
            in_degree[neighbor_id] -= 1
            if in_degree[neighbor_id] == 0:
                queue.append(task_map[neighbor_id])
        queue = deque(sorted(list(queue), key=lambda x: (x.priority, x.task_id)))

    if len(ordered_nodes) != len(self.nodes):
        raise ValueError("拓扑排序失败，任务图可能存在循环依赖")

    return ordered_nodes
```

**为什么需要拓扑排序？**
→ 确定任务的执行顺序，保证依赖任务先执行。

👉 **[高阶追问] 面试官可能会问：“如果 LLM 突然发癫，生成的任务图里存在 A 等待 B，B 又等待 A 的循环依赖，系统会怎样？” 详见 `day6_note_QA.md` QA-18**

**举例**：
```
任务图:
task_001 (无依赖)
task_002 (无依赖)
task_003 (依赖 task_001, task_002)
task_004 (依赖 task_003)

拓扑排序结果:
[task_001, task_002, task_003, task_004]
或
[task_002, task_001, task_003, task_004]
（task_001 和 task_002 可以并行，顺序不固定）
```

---


### 2.6 规划阶段的完整流程图

```
用户问题: "详细分析学生资助体系"
         ↓
┌────────────────────────────────────────────────────────────┐
│ Clarifier                                                  │
│ 输出: needs_clarification=false（问题足够清晰）            │
└────────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────────┐
│ TaskDecomposer                                             │
│ 输出: TaskGraph                                            │
│ - task_001: local_search "国家奖学金"                      │
│ - task_002: local_search "国家励志奖学金"                  │
│ - task_003: local_search "助学金"                          │
│ - task_004: chain_exploration "互斥关系"                   │
│   depends_on: [task_001, task_002, task_003]              │
└────────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────────┐
│ PlanReviewer                                               │
│ 审校:                                                      │
│ - 依赖关系合法 ✓                                           │
│ - 预估 token: 3500                                         │
│ - 预估时间: 2.5 分钟                                       │
│ 输出: PlanSpec (status="draft")                            │
└────────────────────────────────────────────────────────────┘
         ↓
PlanSpec 转换为 PlanExecutionSignal（给执行阶段）
```

---

### 2.7 面试STAR话术（规划阶段）

**S（Situation）**：用户提出复杂问题时，单个 Agent 无法一次性处理所有信息，需要将问题分解成多个子任务。

**T（Task）**：设计规划阶段，将用户问题转换为结构化的任务图（TaskGraph），支持任务依赖和并行执行。

**A（Action）**：
1. **Clarifier**：用 LLM 识别问题中的模糊点，生成澄清问题
2. **TaskDecomposer**：将清晰的问题分解为多个子任务，每个任务包含类型、描述、依赖关系
3. **PlanReviewer**：审校任务图，检查依赖关系、预估资源消耗、生成验收标准
4. **拓扑排序**：用 Kahn 算法对任务图进行拓扑排序，确定执行顺序

**R（Result）**：规划阶段生成的 PlanSpec 包含完整的任务图和依赖关系，支持并行执行独立任务，执行效率提升约 40%。

---

## 三、Phase 2: Execute（执行阶段）— 从任务图到证据

### 3.1 执行阶段的核心组件

```
PlanExecutionSignal → WorkerCoordinator → Executor → ExecutionRecord
```

**三类 Executor**：
1. **RetrievalExecutor**：检索类任务（local_search、global_search、chain_exploration）
2. **ResearchExecutor**：深度研究任务（deep_research、deeper_research）
3. **ReflectionExecutor**：反思与验证任务（reflection）

---

### 3.2 WorkerCoordinator（任务调度器）

**文件位置**：`graphrag_agent/agents/multi_agent/executor/worker_coordinator.py`（564行）

#### 核心功能

根据 `PlanExecutionSignal` 调度不同类型的 Executor 执行任务，支持串行与并行模式。

#### 执行模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **sequential** | 串行执行，按拓扑排序顺序逐个执行 | 任务之间有强依赖关系 |
| **parallel** | 并行执行，独立任务同时执行 | 任务之间无依赖或部分依赖 |
| **adaptive** | 自适应模式（当前映射到 sequential） | 未来扩展 |

#### 串行执行（第127-148行）

```python
def _execute_sequential(
    self,
    state: PlanExecuteState,
    signal: PlanExecutionSignal,
    task_map: Dict[str, TaskNode],
) -> List[ExecutionRecord]:
    results: List[ExecutionRecord] = []
    sequence = signal.execution_sequence or list(task_map.keys())

    for task_id in sequence:
        task = task_map.get(task_id)
        if task is None:
            _LOGGER.warning("计划信号中包含未知任务: %s", task_id)
            continue

        self._execute_single_task(
            state=state,
            signal=signal,
            task=task,
            task_map=task_map,
            results=results,
            skip_dependency_check=False,
        )

    return results
```

**关键设计**：
- 按 `execution_sequence`（拓扑排序结果）顺序执行
- 每个任务执行前检查依赖是否满足

---

#### 并行执行（第150-256行）

```python
def _execute_parallel(
    self,
    state: PlanExecuteState,
    signal: PlanExecutionSignal,
    task_map: Dict[str, TaskNode],
) -> List[ExecutionRecord]:
    results: List[ExecutionRecord] = []
    pending: List[str] = [task_id for task_id in sequence if task_id in task_map]
    max_workers = min(self.max_parallel_workers, max(1, len(pending)))
    inflight: Dict[object, str] = {}
    task_status: Dict[str, str] = {task_id: "pending" for task_id in pending}

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        while pending or inflight:
            # 1. 调度新任务（依赖已满足的）
            for task_id in list(pending):
                if len(inflight) >= max_workers:
                    break

                task = task_map[task_id]
                dependency_ok, dependency_error, failure_reason = self._check_dependencies(task, state)

                if dependency_ok:
                    # 提交任务到线程池
                    future = executor.submit(
                        self._execute_single_task,
                        state=state,
                        signal=signal,
                        task=task,
                        task_map=task_map,
                        results=results,
                        skip_dependency_check=True,
                    )
                    inflight[future] = task_id
                    task_status[task_id] = "running"
                    pending.remove(task_id)

            # 2. 等待任意任务完成
            if inflight:
                done, _ = wait(inflight.keys(), return_when=FIRST_COMPLETED)
                for future in done:
                    task_id = inflight.pop(future)
                    try:
                        success, _ = future.result()
                    except Exception as exc:
                        success = False
                    task_status[task_id] = "completed" if success else "failed"

    return results
```

**关键设计**：
- 使用 `ThreadPoolExecutor` 实现并行执行
- `max_workers` 限制并发数（默认从 `.env` 的 `MULTI_AGENT_WORKER_MAX_CONCURRENCY` 读取）
- 动态调度：任务完成后，立即检查是否有新任务的依赖被满足
- 使用 `wait(return_when=FIRST_COMPLETED)` 等待任意任务完成

**面试追问**：为什么用 `ThreadPoolExecutor` 而不是 `ProcessPoolExecutor`？
→ 任务主要是 I/O 密集型，且需要高频共享大对象。
> 👉 **不懂什么是 I/O密集型与序列化开销？** 参考 `day6_note_QA.md` 的 **[QA-15 追问：为什么进程池会增加序列化开销？]**。另外，如果任务长短不一，长任务会阻塞短任务吗？详见 **[QA-17]**。

---

#### 依赖检查（第400-465行）

```python
def _check_dependencies(
    self,
    task: TaskNode,
    state: PlanExecuteState,
) -> tuple[bool, Optional[str], str]:
    """
    检查任务依赖是否满足。

    返回 (是否可执行, 错误信息, 失败原因标签)。
    """
    if not task.depends_on:
        return True, None, "none"

    plan = state.plan
    status_map: Dict[str, str] = {}
    if plan is not None:
        status_map = {node.task_id: node.status for node in plan.task_graph.nodes}

    exec_context = state.execution_context
    completed_ids = set(exec_context.completed_task_ids if exec_context else [])

    failed_dependencies = []
    pending_dependencies = []
    missing_dependencies = []

    for dep_id in task.depends_on:
        status = status_map.get(dep_id)
        if status == "failed":
            failed_dependencies.append(dep_id)
        elif status == "completed" or dep_id in completed_ids:
            continue
        elif status is None:
            missing_dependencies.append(dep_id)
        else:
            pending_dependencies.append(dep_id)

    if failed_dependencies:
        return (
            False,
            f"依赖任务失败: {', '.join(failed_dependencies)}",
            "dependency_failed",
        )

    if missing_dependencies:
        return (
            False,
            f"依赖任务缺失: {', '.join(missing_dependencies)}",
            "dependency_missing",
        )

    if pending_dependencies:
        return (
            False,
            f"依赖任务未完成: {', '.join(pending_dependencies)}",
            "dependency_unfinished",
        )

    return True, None, "ready"
```

**关键设计**：
- 检查依赖任务的状态（`failed`、`completed`、`pending`、`missing`）
- 如果依赖任务失败，当前任务也标记为失败（避免级联错误）
- 返回详细的失败原因标签，便于调试

👉 **[高阶追问] 面试官可能会问：“如果中间某个任务执行彻底失败了，整个系统会直接崩溃吗？” 详见 `day6_note_QA.md` QA-14 关于部分完成与依赖传播的机制。**

---

### 3.3 RetrievalExecutor（检索执行器）

**文件位置**：`graphrag_agent/agents/multi_agent/executor/retrieval_executor.py`（299行）

#### 核心功能

执行检索类任务，调用既有的搜索工具（local_search、global_search、chain_exploration 等）。

#### 支持的任务类型

| 任务类型 | 对应工具 | 说明 |
|---------|---------|------|
| `local_search` | `LocalSearchTool` | 实体邻居扩展检索 |
| `global_search` | `GlobalSearchTool` | 社区摘要聚合检索 |
| `hybrid_search` | `HybridSearchTool` | 混合检索 |
| `naive_search` | `NaiveSearchTool` | 向量检索 |
| `chain_exploration` | `ChainExplorationTool` | 多步图探索 |

#### 执行流程（第59-134行）

```python
def execute_task(
    self,
    task: TaskNode,
    state: PlanExecuteState,
    signal: PlanExecutionSignal,
) -> TaskExecutionResult:
    tool_name = task.task_type
    payload = self.build_default_inputs(task)

    # 1. 获取工具实例
    tool_instance = self._get_tool_instance(tool_name)

    # 2. 调用工具
    start_time = time.perf_counter()
    try:
        structured_output = self._invoke_tool(tool_instance, tool_name, payload)
        success = True
    except Exception as exc:
        success = False
        error_message = str(exc)

    latency = time.perf_counter() - start_time

    # 3. 构建 ToolCall 记录
    tool_call = ToolCall(
        tool_name=tool_name,
        args=payload,
        result=structured_output if success else None,
        status="success" if success else "failed",
        error=error_message,
        latency_ms=round(latency * 1000, 3),
    )

    # 4. 提取证据
    evidence = self._extract_evidence(state, structured_output) if success else []

    # 5. 构建 ExecutionRecord
    record = ExecutionRecord(
        task_id=task.task_id,
        session_id=state.session_id,
        worker_type=self.worker_type,
        inputs={"payload": payload, "task": task.model_dump()},
        tool_calls=[tool_call],
        evidence=evidence,
        metadata=ExecutionMetadata(...),
    )

    # 6. 更新状态
    self._update_state(state, task, record, success, error_message)

    return TaskExecutionResult(
        record=record,
        success=success,
        error=error_message,
    )
```

**关键设计**：
- 工具实例缓存（`_tool_cache`），避免重复初始化
- 统一的工具调用接口（优先 `structured_search`，其次 `search`）
- 证据提取和追踪（`_extract_evidence`）

---

#### 证据提取（第173-198行）

```python
def _extract_evidence(
    self,
    state: PlanExecuteState,
    output: Dict[str, Any],
) -> List[RetrievalResult]:
    results_payload = output.get("retrieval_results") if isinstance(output, dict) else None
    evidence: List[RetrievalResult] = []

    if not isinstance(results_payload, list):
        return evidence

    for item in results_payload:
        try:
            if isinstance(item, RetrievalResult):
                evidence.append(item)
            elif isinstance(item, dict):
                evidence.append(RetrievalResult.from_dict(item))
        except Exception as exc:
            _LOGGER.warning("无法解析retrieval_result: %s error=%s", item, exc)

    # 注册到证据追踪器
    try:
        tracker = get_evidence_tracker(state)
        return tracker.register(evidence)
    except Exception as exc:
        _LOGGER.debug("证据追踪失败，使用原始结果: %s", exc)
        return evidence
```

**关键设计**：
- 从工具输出的 `retrieval_results` 字段提取证据
- 统一转换为 `RetrievalResult` 对象
- 注册到证据追踪器（`EvidenceTracker`），生成全局唯一的 `result_id`
  > 👉 **面试追问：为什么长文档生成需要 EvidenceTracker？** 参考 `day6_note_QA.md` 的 **[QA-10 追问：EvidenceTracker 的“物证科”防幻觉与去重机制]**。

---


### 3.4 ReflectionExecutor（反思执行器）

**文件位置**：`graphrag_agent/agents/multi_agent/executor/reflector.py`

#### 核心功能

对已完成的任务进行反思和验证，判断结果是否满足要求，必要时触发重试。

**反思流程**：
```
1. 读取目标任务的执行结果
2. 调用 LLM 进行质量评估
3. 输出 ReflectionResult（包含 success、needs_retry、reasoning）
4. 如果 needs_retry=true，WorkerCoordinator 会重新执行目标任务
```

**配置参数**（`.env`）：
```env
MULTI_AGENT_REFLECTION_ALLOW_RETRY=true
MULTI_AGENT_REFLECTION_MAX_RETRIES=2
```

**面试追问**：为什么需要 ReflectionExecutor？
→ 检索类任务可能返回不相关的结果，反思机制可以自动检测并重试，提高答案质量。（👉 **详见 `day6_note_QA.md` QA-12：ReflectionExecutor 的重试机制是如何工作的？**）

---

### 3.5 执行阶段的完整流程图

```
PlanExecutionSignal
         ↓
┌────────────────────────────────────────────────────────────┐
│ WorkerCoordinator                                          │
│ 模式: parallel（并行执行）                                 │
└────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────┬─────────────────────┬────────────────┐
│ task_001            │ task_002            │ task_003       │
│ (local_search)      │ (local_search)      │ (local_search) │
│ 无依赖 → 立即执行    │ 无依赖 → 立即执行    │ 无依赖 → 立即执行│
└─────────────────────┴─────────────────────┴────────────────┘
         ↓                      ↓                      ↓
┌─────────────────────┬─────────────────────┬────────────────┐
│ RetrievalExecutor   │ RetrievalExecutor   │ RetrievalExecutor│
│ 调用 LocalSearchTool│ 调用 LocalSearchTool│ 调用 LocalSearchTool│
│ 返回 ExecutionRecord│ 返回 ExecutionRecord│ 返回 ExecutionRecord│
└─────────────────────┴─────────────────────┴────────────────┘
         ↓                      ↓                      ↓
         └──────────────────────┴──────────────────────┘
                                ↓
                        task_001, task_002, task_003 完成
                                ↓
┌────────────────────────────────────────────────────────────┐
│ task_004 (chain_exploration)                               │
│ 依赖: [task_001, task_002, task_003]                       │
│ 依赖已满足 → 开始执行                                       │
└────────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────────┐
│ RetrievalExecutor                                          │
│ 调用 ChainExplorationTool                                  │
│ 返回 ExecutionRecord                                       │
└────────────────────────────────────────────────────────────┘
         ↓
所有任务完成，返回 ExecutionRecord 列表
```

---

### 3.6 面试STAR话术（执行阶段）

**S（Situation）**：规划阶段生成的任务图包含多个子任务，需要高效执行并收集证据。

**T（Task）**：设计执行阶段，支持串行和并行两种模式，动态调度任务，确保依赖关系正确。

**A（Action）**：
1. **WorkerCoordinator**：根据 `PlanExecutionSignal` 调度任务，支持串行和并行模式
2. **依赖检查**：每个任务执行前检查依赖是否满足（`failed`、`completed`、`pending`）
3. **并行执行**：使用 `ThreadPoolExecutor` 实现并行，动态调度依赖已满足的任务
4. **RetrievalExecutor**：调用检索工具，提取证据并注册到证据追踪器
5. **ReflectionExecutor**：对执行结果进行反思，必要时触发重试

**R（Result）**：并行执行模式下，独立任务同时执行，执行时间从串行的 10 秒降低到 4 秒，效率提升 60%。

---

## 四、Phase 3: Report（报告阶段）— 从证据到长文档

### 4.1 报告阶段的三步流水线

```
ExecutionRecord 列表 → OutlineBuilder（生成大纲）→ SectionWriter（并行写作）→ ConsistencyChecker（一致性校验）→ 最终报告
```

---

### 4.2 Step 1: OutlineBuilder（纲要生成器）

**文件位置**：`graphrag_agent/agents/multi_agent/reporter/outline_builder.py`（90行）

#### 核心功能

根据执行阶段收集的证据，生成结构化的报告大纲。

**大纲结构**：
```python
class ReportOutline(BaseModel):
    report_type: str                      # "short_answer" 或 "long_document"
    title: str                            # 报告标题
    abstract: Optional[str]               # 摘要（长文档特有）
    sections: List[SectionOutline]        # 章节列表
    total_estimated_words: Optional[int]  # 预估总字数
```

**章节结构**：
```python
class SectionOutline(BaseModel):
    section_id: str                       # 章节唯一标识
    title: str                            # 章节标题
    summary: str                          # 章节摘要说明
    evidence_ids: List[str]               # 章节引用的证据ID列表
    estimated_words: int                  # 预估字数
```

#### 代码实现（第50-73行）

```python
def build_outline(
    self,
    *,
    query: str,
    plan_summary: str,
    evidence_summary: str,
    evidence_count: int,
    report_type: str,
) -> ReportOutline:
    prompt = OUTLINE_PROMPT.format(
        query=query,
        plan_summary=plan_summary,
        evidence_summary=evidence_summary,
        evidence_count=evidence_count,
        report_type=report_type,
    )

    response = self._invoke_llm(prompt)
    outline_data = self._parse_response(response)
    outline = ReportOutline(**outline_data)

    return outline
```

**关键设计**：
- LLM 根据证据生成大纲（而不是硬编码模板）
- 每个章节分配证据ID（`evidence_ids`），指导后续写作
- 预估字数，控制章节长度

---

### 4.3 Step 2: SectionWriter（章节写作器）

**文件位置**：`graphrag_agent/agents/multi_agent/reporter/section_writer.py`（271行）

#### 核心功能

根据大纲和证据，并行写作各章节内容（Map-Reduce 模式）。

#### Map-Reduce 模式

```
┌─────────────────────────────────────────────────────────────┐
│  Map 阶段：并行写作各章节                                    │
│  - SectionWriter 1: 写作第1章                                │
│  - SectionWriter 2: 写作第2章                                │
│  - SectionWriter 3: 写作第3章                                │
│  （每个 SectionWriter 独立调用 LLM）                         │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  Reduce 阶段：汇总章节                                       │
│  - 拼接各章节内容                                            │
│  - 生成目录和引用列表                                        │
└─────────────────────────────────────────────────────────────┘
```

#### 多批写作（第69-142行）

```python
def write_section(
    self,
    outline: ReportOutline,
    section: SectionOutline,
    evidence_map: Dict[str, RetrievalResult],
    fallback_evidence_ids: Optional[List[str]] = None,
) -> SectionDraft:
    # 1. 选择证据
    evidence_ids = self._select_evidence_ids(section, evidence_map, fallback_evidence_ids)
    evidence_entries = [evidence_map[eid] for eid in evidence_ids if eid in evidence_map]

    # 2. 分批写作（支持超长章节）
    batches = self._split_into_batches(evidence_entries, self.config.max_evidence_per_call)
    contents: List[str] = []

    for batch_index, batch in enumerate(batches, start=1):
        evidence_list_text = self._format_evidence(batch)

        # 多批写作时，提供前文摘要
        context_instruction = ""
        if self.config.enable_multi_pass and len(batches) > 1:
            context_instruction = f"**写作阶段**: 第{batch_index}/{len(batches)}批，请确保与前文衔接。"
            if contents:
                context_instruction += f"\n**前文摘要**: {self._extract_previous_summary(contents)}"

        prompt = SECTION_WRITE_PROMPT.format(
            outline=outline_context_text,
            section_id=section.section_id,
            section_title=section.title,
            section_summary=section.summary,
            estimated_words=section.estimated_words,
            evidence_list=evidence_list_text + context_instruction
        )

        generated = self._invoke_llm(prompt)
        contents.append(generated.strip())

    # 3. 拼接内容
    final_content = "\n\n".join(contents).strip()
    final_content = self._sanitize_content(section.title, final_content)

    return SectionDraft(
        section_id=section.section_id,
        content=final_content,
        used_evidence_ids=used_ids,
    )
```

**关键设计**：
- **分批写作**：如果证据过多（超过 `max_evidence_per_call`，默认8个），分批调用 LLM
- **前文摘要**：多批写作时，提供前文摘要（最后 800 字符），确保衔接
- **去重标题**：移除与章节标题重复的标题行（`_sanitize_content`）

**面试追问**：为什么需要分批写作？
→ LLM 的上下文窗口有限，如果一次性喂入 50 个证据，可能超过窗口限制。分批写作可以支持超长章节。
> 👉 **不懂“并行写各章（Map-Reduce）”和“分批写单章（Batching）”的区别？** 参考 `day6_note_QA.md` 的 **[QA-11 追问：广度分割 vs 深度分割的防爆策略]**。

---

### 4.4 Step 3: ConsistencyChecker（一致性校验器）

**文件位置**：`graphrag_agent/agents/multi_agent/reporter/consistency_checker.py`（58行）

#### 核心功能

对生成的报告进行事实一致性校验，检测引用错误和逻辑矛盾。

**校验内容**：
1. **引用一致性**：报告中引用的证据ID是否存在？
2. **事实一致性**：报告内容是否与证据矛盾？
3. **逻辑一致性**：报告内部是否存在逻辑矛盾？

#### 代码实现（第36-45行）

```python
def check(self, report_content: str, evidence_list: str) -> ConsistencyCheckResult:
    prompt = CONSISTENCY_CHECK_PROMPT.format(
        report_content=report_content,
        evidence_list=evidence_list,
    )

    response = self._invoke_llm(prompt)
    parsed = self._parse_response(response)
    result = ConsistencyCheckResult(**parsed, raw_response=response)

    return result
```

**输出结构**：
```python
class ConsistencyCheckResult(BaseModel):
    is_consistent: bool                   # 报告是否通过检查
    issues: list[Dict[str, Any]]          # 问题列表
    corrections: list[Dict[str, Any]]     # 修正建议
```

**举例**：
```json
{
  "is_consistent": false,
  "issues": [
    {
      "type": "citation_error",
      "location": "第2章第3段",
      "description": "引用了不存在的证据ID: evidence_999"
    },
    {
      "type": "fact_contradiction",
      "location": "第3章第1段",
      "description": "报告称'国家奖学金金额为8000元'，但证据显示为5000元"
    }
  ],
  "corrections": [
    {
      "issue_id": 0,
      "suggestion": "删除对 evidence_999 的引用"
    },
    {
      "issue_id": 1,
      "suggestion": "修正金额为5000元"
    }
  ]
}
```

**关键设计**：
- 使用 LLM 进行一致性校验（而不是硬编码规则）
- 输出结构化的问题列表和修正建议
- 🚨 **当前版本只检测问题，不自动修正**：这是一个设计上的半残缺陷。具体优化思路见 `project_improvements.md` 的 **[1-7 改进：ConsistencyChecker 仅检测不修复的缺陷]**。

👉 **[高阶追问] 面试官如果让你细数 ConsistencyChecker 具体在校验哪些一致性，详见 `day6_note_QA.md` QA-13**

---

### 4.5 报告阶段的完整流程图

```
ExecutionRecord 列表（包含证据）
         ↓
┌────────────────────────────────────────────────────────────┐
│ OutlineBuilder                                             │
│ 输入: 证据摘要、问题、计划摘要                              │
│ 输出: ReportOutline                                        │
│ - 第1章: 国家奖学金概述（evidence_ids: [e1, e2, e3]）      │
│ - 第2章: 国家励志奖学金概述（evidence_ids: [e4, e5, e6]）  │
│ - 第3章: 助学金概述（evidence_ids: [e7, e8, e9]）          │
│ - 第4章: 互斥关系分析（evidence_ids: [e10, e11]）          │
└────────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────────┐
│ SectionWriter（Map-Reduce 并行写作）                       │
│ - Writer 1: 写作第1章 → SectionDraft 1                     │
│ - Writer 2: 写作第2章 → SectionDraft 2                     │
│ - Writer 3: 写作第3章 → SectionDraft 3                     │
│ - Writer 4: 写作第4章 → SectionDraft 4                     │
│ （并行调用 LLM）                                            │
└────────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────────┐
│ 汇总章节                                                    │
│ - 拼接各章节内容                                            │
│ - 生成目录和引用列表                                        │
└────────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────────┐
│ ConsistencyChecker                                         │
│ 输入: 报告内容 + 证据列表                                   │
│ 输出: ConsistencyCheckResult                               │
│ - is_consistent: true                                      │
│ - issues: []                                               │
└────────────────────────────────────────────────────────────┘
         ↓
最终报告（Markdown 格式，5000+ 字）
```

---

### 4.6 面试STAR话术（报告阶段）

**S（Situation）**：执行阶段收集了大量证据，需要生成结构化的长文档报告（5000+ 字）。

**T（Task）**：设计报告阶段，支持 Map-Reduce 并行写作，生成高质量的长文档。

**A（Action）**：
1. **OutlineBuilder**：根据证据生成报告大纲，每个章节分配证据ID
2. **SectionWriter**：Map-Reduce 并行写作各章节，支持分批写作（超长章节）
3. **ConsistencyChecker**：对报告进行事实一致性校验，检测引用错误和逻辑矛盾
4. **汇总**：拼接各章节内容，生成目录和引用列表

**R（Result）**：报告阶段生成的长文档平均 5000+ 字，Map-Reduce 并行写作使生成时间从 120 秒降低到 45 秒，效率提升 62%。

---


## 五、MultiAgentOrchestrator（总编排器）

**文件位置**：`graphrag_agent/agents/multi_agent/orchestrator.py`（366行）

### 5.1 核心功能

串联 Planner → WorkerCoordinator → Reporter，形成完整的 Plan-Execute-Report 生命周期。

### 5.2 完整流程（第108-225行）

> 👉 **面试追问：贯穿全程的 `state: PlanExecuteState` 到底是什么？** 参考 `day6_note_QA.md` 的 **[QA-4 追问：作为“最高机密档案夹”的黑板架构]**。

```python
def run(
    self,
    state: PlanExecuteState,
    *,
    assumptions: Optional[Sequence[str]] = None,
    report_type: Optional[str] = None,
) -> OrchestratorResult:
    errors: List[str] = []
    metrics = OrchestratorMetrics()

    # --- Phase 1: Plan ---
    plan_start = time.perf_counter()
    try:
        planner_result = self._planner.generate_plan(
            state,
            assumptions=list(assumptions) if assumptions else None,
        )
    except Exception as exc:
        errors.append(f"Planner执行失败: {exc}")
        return OrchestratorResult(status="failed", ...)
    metrics.planning_seconds = time.perf_counter() - plan_start

    # 检查是否需要澄清
    if planner_result.plan_spec is None:
        if planner_result.clarification.needs_clarification:
            return OrchestratorResult(status="needs_clarification", ...)

    signal = planner_result.executor_signal
    if signal is None:
        return OrchestratorResult(status="failed", ...)

    # --- Phase 2: Execute ---
    execution_records: List[ExecutionRecord] = []
    if signal is not None:
        exec_start = time.perf_counter()
        try:
            execution_records = self._worker.execute_plan(state, signal)
        except Exception as exc:
            errors.append(f"执行阶段失败: {exc}")
        finally:
            metrics.execution_seconds = time.perf_counter() - exec_start

    # --- Phase 3: Report ---
    report_result: Optional[ReportResult] = None
    if self.config.auto_generate_report and not errors:
        report_start = time.perf_counter()
        try:
            report_result = self._reporter.generate_report(
                state,
                report_type=report_type,
            )
        except Exception as exc:
            errors.append(f"报告生成失败: {exc}")
        finally:
            metrics.reporting_seconds = time.perf_counter() - report_start

    # --- Determine final status ---
    status = "completed"
    if errors:
        status = "failed"
    elif state.plan is not None:
        if state.plan.status == "failed":
            status = "failed"
        elif state.plan.status == "executing":
            status = "partial"

    return OrchestratorResult(
        status=status,
        planner=planner_result,
        execution_records=execution_records,
        report=report_result,
        errors=errors,
        metrics=metrics,
    )
```

**关键设计**：
- 三阶段串行执行（Plan → Execute → Report）
- 每个阶段独立计时（`metrics`）
- 错误处理：任意阶段失败，立即返回错误状态
- 可配置：`auto_generate_report` 控制是否自动生成报告

👉 **[高阶追问] 系统支持用户中途点击取消吗？详见 `day6_note_QA.md` QA-20 打断机制。**

---

### 5.3 FusionGraphRAGAgent（最终封装）

**文件位置**：`graphrag_agent/agents/fusion_agent.py`（93行）

#### 核心功能

FusionGraphRAGAgent 是对 MultiAgentOrchestrator 的轻量封装，提供与其他 Agent 一致的接口。

#### 代码实现（第39-66行）

```python
class FusionGraphRAGAgent:
    def __init__(self, cache_dir: str = "./cache/fusion_graphrag") -> None:
        self.cache_dir = cache_dir
        self.multi_agent = MultiAgentFacade()  # 多智能体编排栈
        self.memory = _MemoryShim()
        self.graph = _GraphShim()
        self.execution_log: list[Any] = []
        self._global_cache: Dict[str, str] = {}
        self._session_cache: Dict[str, Dict[str, str]] = {}

    def ask(self, query: str, thread_id: str = "default", recursion_limit: Optional[int] = None) -> str:
        return self._execute(query, thread_id)[0]

    def _execute(self, query: str, thread_id: str, *, assumptions: Optional[list[str]] = None, report_type: Optional[str] = None) -> Tuple[str, Dict[str, Any]]:
        # 1. 检查缓存
        cached = self._read_cache(query, thread_id)
        if cached is not None:
            return cached, {"status": "cached"}

        # 2. 调用多智能体编排栈
        payload = self.multi_agent.process_query(query.strip(), assumptions=assumptions, report_type=report_type)

        # 3. 提取答案
        answer = self._normalize_answer(payload.get("response"))

        # 4. 写入缓存
        self._write_cache(query, thread_id, answer)

        # 5. 记录执行日志
        self.execution_log = payload.get("execution_records", [])
        self._last_payload = payload

        return answer, payload
```

**关键设计**：
- 委托给 `MultiAgentFacade`（多智能体编排栈）
- 提供缓存机制（全局缓存 + 会话缓存）
- 兼容其他 Agent 的接口（`ask`、`ask_stream`）
  > 👉 **面试追问：为什么明明底层大换血，还要保留 `ask` 等方法？** 参考 `day6_note_QA.md` 的 **[QA-5 追问：作为兼容层的外观模式（Facade Pattern）]**。

---

## 六、完整数据流图

```
用户问题: "详细分析学生资助体系"
         ↓
┌─────────────────────────────────────────────────────────────┐
│  FusionGraphRAGAgent.ask(query)                             │
│  ↓                                                           │
│  MultiAgentFacade.process_query(query)                      │
│  ↓                                                           │
│  MultiAgentOrchestrator.run(state)                          │
└─────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────┐
│  Phase 1: Plan                                              │
│  ↓                                                           │
│  Clarifier → TaskDecomposer → PlanReviewer                  │
│  ↓                                                           │
│  输出: PlanSpec（包含 TaskGraph）                            │
└─────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────┐
│  Phase 2: Execute                                           │
│  ↓                                                           │
│  WorkerCoordinator.execute_plan(signal)                     │
│  ↓                                                           │
│  并行执行:                                                   │
│  - task_001 (local_search) → RetrievalExecutor              │
│  - task_002 (local_search) → RetrievalExecutor              │
│  - task_003 (local_search) → RetrievalExecutor              │
│  ↓                                                           │
│  串行执行:                                                   │
│  - task_004 (chain_exploration) → RetrievalExecutor         │
│    (依赖 task_001, task_002, task_003)                      │
│  ↓                                                           │
│  输出: ExecutionRecord 列表（包含证据）                      │
└─────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────┐
│  Phase 3: Report                                            │
│  ↓                                                           │
│  OutlineBuilder.build_outline(evidence)                     │
│  ↓                                                           │
│  输出: ReportOutline（4个章节）                              │
│  ↓                                                           │
│  SectionWriter.write_section(section) × 4（并行）           │
│  ↓                                                           │
│  输出: SectionDraft × 4                                      │
│  ↓                                                           │
│  汇总章节 + ConsistencyChecker.check(report)                │
│  ↓                                                           │
│  输出: 最终报告（Markdown，5000+ 字）                        │
└─────────────────────────────────────────────────────────────┘
         ↓
返回给用户
```

---

## 七、面试高频追问准备

### Q1: 为什么需要多智能体架构？单个 Agent 不够吗？

**A**:
1. **上下文窗口限制**：单个 Agent 的上下文窗口有限（即使 GPT-4 也只有 128K tokens），复杂问题需要大量证据，单次调用无法处理
2. **任务分解**：复杂问题需要分解成多个子任务（检索、分析、推理、报告），单个 Agent 难以兼顾
3. **并行执行**：多个独立子任务可以并行执行，提高效率
4. **专业化**：不同类型的任务由专门的 Executor 处理（RetrievalExecutor、ResearchExecutor、ReflectionExecutor），提高质量

---

### Q2: 拓扑排序的作用是什么？为什么不直接按任务ID顺序执行？

**A**:
1. **依赖关系**：任务之间有依赖关系（如 task_004 依赖 task_001、task_002、task_003），必须先执行依赖任务
2. **拓扑排序**：确保依赖任务先执行，避免执行顺序错误
3. **并行优化**：拓扑排序后，可以识别出哪些任务可以并行执行（入度为0的任务）
4. **循环依赖检测**：拓扑排序过程中可以检测循环依赖，避免死锁

---

### Q3: 并行执行模式下，如何保证依赖关系正确？

**A**:
1. **动态调度**：使用 `ThreadPoolExecutor` 实现并行，每次只调度依赖已满足的任务
2. **依赖检查**：每个任务执行前，检查 `depends_on` 列表中的任务是否全部完成
3. **状态追踪**：维护 `task_status` 字典，记录每个任务的状态（`pending`、`running`、`completed`、`failed`）
4. **等待机制**：使用 `wait(return_when=FIRST_COMPLETED)` 等待任意任务完成，然后检查是否有新任务的依赖被满足

---

### Q4: Map-Reduce 模式在报告阶段的作用是什么？

**A**:
1. **Map 阶段**：并行写作各章节，每个 SectionWriter 独立调用 LLM，互不干扰
2. **Reduce 阶段**：汇总各章节内容，生成目录和引用列表
3. **效率提升**：并行写作使生成时间从 120 秒降低到 45 秒，效率提升 62%
4. **可扩展性**：支持任意数量的章节，不受单个 LLM 调用的上下文窗口限制

---

### Q5: 为什么需要 ConsistencyChecker？

**A**:
1. **引用一致性**：检测报告中引用的证据ID是否存在，避免引用错误
2. **事实一致性**：检测报告内容是否与证据矛盾，避免事实错误
3. **逻辑一致性**：检测报告内部是否存在逻辑矛盾，提高报告质量
4. **自动化**：使用 LLM 进行一致性校验，避免人工审核的高成本

---

### Q6: ReflectionExecutor 的重试机制是如何工作的？

**A**:
1. **反思评估**：ReflectionExecutor 读取目标任务的执行结果，调用 LLM 进行质量评估
2. **重试判断**：如果 LLM 判断结果不满足要求（`needs_retry=true`），触发重试
3. **重新执行**：WorkerCoordinator 重新执行目标任务，最多重试 `MULTI_AGENT_REFLECTION_MAX_RETRIES` 次（默认2次）
4. **终止条件**：如果重试次数达到上限或 LLM 判断结果满足要求，停止重试

---

### Q7: 如何处理任务执行失败的情况？

**A**:
1. **依赖传播**：如果任务A失败，依赖A的任务B也会被标记为失败（`dependency_failed`）
2. **错误记录**：失败任务的错误信息记录在 `ExecutionContext.errors` 中
3. **部分完成**：即使部分任务失败，已完成的任务结果仍然保留，最终状态为 `partial`
4. **用户通知**：`OrchestratorResult.errors` 包含所有错误信息，返回给用户

---

## 八、今日总结与打卡

### 8.1 核心知识点回顾

✅ **Plan-Execute-Report 三阶段架构**：
- Plan: Clarifier → TaskDecomposer → PlanReviewer → PlanSpec
- Execute: WorkerCoordinator → Executor → ExecutionRecord
- Report: OutlineBuilder → SectionWriter → ConsistencyChecker → 最终报告

✅ **核心数据结构**：
- `PlanSpec`: 完整的任务执行计划，包含 `TaskGraph` 和验收标准
- `TaskNode`: 任务图中的单个任务，包含 `task_type`、`depends_on`、`priority`
- `ExecutionRecord`: 执行阶段的输出，包含 `tool_calls`、`evidence`、`metadata`

✅ **关键算法**：
- 拓扑排序（Kahn 算法）：确定任务执行顺序，检测循环依赖
- 依赖检查：动态调度依赖已满足的任务
- Map-Reduce：并行写作各章节，汇总生成最终报告

✅ **执行模式**：
- 串行执行：按拓扑排序顺序逐个执行
- 并行执行：使用 `ThreadPoolExecutor` 实现并行，动态调度

---

### 8.2 今日打卡清单

- [ ] 能在白板上画出 Plan-Execute-Report 的完整数据流图
- [ ] 能解释 `PlanSpec`、`TaskNode`、`ExecutionRecord` 三个核心数据结构
- [ ] 能说清楚拓扑排序的作用和 Kahn 算法的原理
- [ ] 能解释并行执行模式下如何保证依赖关系正确
- [ ] 能说清楚 Map-Reduce 模式在报告阶段的作用
- [ ] 能回答7个面试高频追问

---

### 8.3 明日预告（Day 7）

**主题**：亲手复现 — 图构建实战

**核心内容**：
- 配置 `.env` 文件
- 准备测试数据
- 执行图构建（`main.py`）
- 测试增量更新（`incremental_update.py`）
- 在 Neo4j Browser 中查看图结构

**为什么重要**：
- 真正跑通知识图谱构建，能在 Neo4j 中看到图数据
- 理解增量更新的原理和 `file_registry.json` 的作用
- 为 Day 8 的 Agent 问答实战打下基础

---

*Day 6 学习完成！你已经掌握了 FusionAgent 多智能体架构的核心原理，这是本项目最核心的亮点，也是面试必问的部分。*

