# GitHub Fork & PR 操作完整指南

> 本指南以 `graph-rag-agent` 项目为例，记录从 Fork 到提交 Issue/PR 的完整流程。

---

## 一、Fork 项目（GitHub 网页操作）

1. 登录 GitHub（https://github.com）
2. 访问原项目：**https://github.com/1517005260/graph-rag-agent**
3. 点击页面右上角 **「Fork」** 按钮
4. 选择你自己的账号，Repository name 保持默认
5. 点击 **「Create fork」**
6. 完成后你拥有：`https://github.com/你的用户名/graph-rag-agent`

---

## 二、关联本地项目

你本地已经 clone 了原仓库，需要把远程地址切换到自己的 fork：

```bash
# ① 把原来的 origin（原作者）改名为 upstream
git remote rename origin upstream

# ② 添加你的 fork 作为新的 origin
git remote add origin https://github.com/你的用户名/graph-rag-agent.git

# ③ 验证
git remote -v
# 期望输出：
# origin    https://github.com/你的用户名/graph-rag-agent.git (fetch)
# origin    https://github.com/你的用户名/graph-rag-agent.git (push)
# upstream  https://github.com/1517005260/graph-rag-agent.git (fetch)
# upstream  https://github.com/1517005260/graph-rag-agent.git (push)

# ④ 推送到你的 fork，确认连通
git push origin master
```

> **本项目的主分支是 `master`，不是 `main`。**

---

## 三、认证方式（首次推送时需要）

GitHub 已不支持密码推送，需要用 **Personal Access Token**：

1. 打开 https://github.com/settings/tokens
2. 点击 **「Generate new token (classic)」**
3. Note 填 `graph-rag-agent`，勾选 **`repo`** 权限
4. 点击 **「Generate token」** → **立刻复制**（只显示一次！）
5. 推送时，密码处粘贴这个 token

---

## 四、提交 Issue

### 在原仓库提交（不是你的 fork）

1. 访问 https://github.com/1517005260/graph-rag-agent/issues
2. 点击 **「New issue」**
3. 填写标题和描述（参考下方模板）
4. 点击 **「Submit new issue」**

### Issue 模板示例

```markdown
### 标题
[Bug] `_generate_node` 中 `messages[-3]` 硬编码索引在多工具调用场景下可能取到错误内容

### 描述
`naive_rag_agent.py` 和 `graph_agent.py` 的 `_generate_node` 中
使用 `messages[-3].content` 获取用户原始问题。

该写法假定 messages 结构固定为 3 条：
`[HumanMessage, AIMessage(tool_calls), ToolMessage]`

### 潜在问题
当 LLM 在单次调用中返回多个 tool_calls 时，ToolNode 会为每个
tool_call 生成独立的 ToolMessage，导致 messages 长度 > 3，
此时 `messages[-3]` 不再指向 HumanMessage。

### 建议修复

用类型匹配代替硬编码索引：

    question = "未找到问题"
    for msg in reversed(messages):
        if isinstance(msg, HumanMessage):
            question = msg.content
            break

### 影响范围
- `graphrag_agent/agents/naive_rag_agent.py` L57
- `graphrag_agent/agents/graph_agent.py` 对应位置
```

---

## 五、提交 PR（Pull Request）

### 5.1 本地创建修复分支

```bash
# ① 确保在最新的 master 上
git checkout master
git pull upstream master   # 从原作者仓库拉取最新代码并合并，确保你的修复基于最新版本

# ② 创建新分支（命名规范：类型/简短描述）
git checkout -b fix/hardcoded-message-index
```

> **为什么要创建新分支？不能直接在 master 上改吗？**
>
> **① `master` 要保持和原项目同步**
> `master` 分支的职责是：随时能用 `git pull upstream master` 拉取原作者的最新代码。如果你在 master 上改了东西，下次同步时就可能产生冲突。
>
> **② 一个分支 = 一个 PR = 一件事**
> 每个 PR 应该只做一件事（比如"修复 messages[-3] 问题"）。如果你在 master 上改了 A 又改了 B，提 PR 时 A 和 B 会混在一起，原作者很难审查。新建分支让每个修复独立、干净。
>
> **③ 可以同时做多件事**
> ```
> master                    ← 干净的，随时同步原仓库
>   ├── fix/message-index   ← PR1：修复索引问题
>   ├── feat/add-logging    ← PR2：添加日志功能
>   └── docs/update-readme  ← PR3：更新文档
> ```
> 三个分支互不影响，可以分别提三个 PR。
>
> **类比**：master 是"正式文件柜"，新分支是"草稿纸"。你在草稿纸上改好、确认没问题后，再通过 PR 把草稿纸的内容合并到正式文件柜里。

### 5.2 修改代码

修改 `naive_rag_agent.py` 和 `graph_agent.py` 中的 `messages[-3]`，改为类型查找。

### 5.3 提交并推送

```bash
# 查看改了什么
git diff

# 暂存
git add -A

# 提交（commit message 规范：类型: 描述）
git commit -m "fix: use type-based lookup instead of hardcoded messages[-3] index"

# 推送到你的 fork
git push origin fix/hardcoded-message-index
```

### 5.4 在 GitHub 上创建 PR

1. 打开你的 fork 页面：`https://github.com/你的用户名/graph-rag-agent`
2. 页面顶部会出现黄色提示条：**「Compare & pull request」**
3. 点击按钮
4. **Base repository** 选 `1517005260/graph-rag-agent`，**base** 选 `master`
5. **Head repository** 选你的 fork，**compare** 选 `fix/hardcoded-message-index`
6. 填写 PR 标题和描述
7. 点击 **「Create pull request」**

---

## 六、合并分支到 master

### 常见困惑：切回 master 后文件"消失"了？

**这不是 bug，是 Git 分支的正常行为！**

如果你在 `test-branch` 上新建了文件并 commit，切回 `master` 后这些文件不会出现在 master 上——因为它们只存在于 `test-branch` 中。文件并没有丢，切回 `test-branch` 就能看到。

### 把分支内容合并到 master

```bash
# ① 切到 master
git checkout master

# ② 把目标分支的改动合并过来
git merge test-branch
# → master 现在拥有了 test-branch 中的所有文件和改动

# ③ 合并完成后，可以删掉不再需要的分支
git branch -d test-branch

# ④ 推送更新后的 master 到远程
git push origin master
```

> **`merge` 做了什么？**
>
> 把目标分支的所有 commit "搬"到当前分支上。合并后 master 就和 test-branch 拥有相同的文件内容了。
>
> ```
> 合并前：
> master:      A → B → C
> test-branch: A → B → C → D（你的新文件）
>
> git merge test-branch 后：
> master:      A → B → C → D（新文件也到 master 了）
> ```

---

## 七、同步原仓库更新

原作者更新了代码后，你需要把更新拉到本地：

```bash
# ① 拉取原作者最新代码
git fetch upstream

# ② 切到 master 并合并
git checkout master
git merge upstream/master

# ③ 推送到你的 fork（保持同步）
git push origin master
```

---

## 八、常见问题 FAQ

### Q1：`git push origin master` 中的 `origin` 是什么？

`origin` 是远程仓库 URL 的**别名**，因为每次输入完整 URL 太长了：

```
git push origin master
         ↑       ↑
         │       └── 推哪个分支
         └── 推到哪个远程仓库（别名）
```

| 别名 | 实际地址 | 角色 |
|------|---------|------|
| `origin` | `https://github.com/KEDADA/graph-rag-agent.git` | 你的 fork |
| `upstream` | `https://github.com/1517005260/graph-rag-agent.git` | 原作者 |

这些别名通过 `git remote add` 自定义，叫什么都行。`origin`/`upstream` 只是社区约定。

### Q2：分支名 `fix/hardcoded-message-index` 中的 `/` 是文件夹吗？

**不是文件夹**，`/` 只是分支名中的普通字符。这是一种命名规范，让分支用途一目了然：

| 前缀 | 含义 | 示例 |
|------|------|------|
| `fix/` | 修复 bug | `fix/message-index` |
| `feat/` | 新功能 | `feat/add-streaming` |
| `docs/` | 文档更新 | `docs/update-readme` |
| `refactor/` | 重构 | `refactor/cache-system` |

Git 本身不关心分支名，叫 `abc123` 也能跑。但在团队协作中 `fix/xxx` 一眼就知道这个分支在做什么。

### Q3：push 需要每个分支都推吗？

**不用**，只推你改了的分支：

```bash
git push origin master                        # 只推 master
git push origin fix/hardcoded-message-index   # 只推这个分支
```

每个分支是独立的，互不影响。

### Q4：原作者更新了代码，怎么同步而不影响我的修改？

```bash
# ① 下载原作者最新代码（不改任何文件）
git fetch upstream

# ② 只合并到 master（你的修改分支不受影响）
git checkout master
git merge upstream/master

# ③ 推到你的 fork
git push origin master
```

你的修改分支（如 `fix/xxx`）完全不受影响。如果后续想让修改分支也获得最新代码：

```bash
git checkout fix/hardcoded-message-index
git merge master    # 把更新后的 master 合并进来
```

### Q5：push 会不会把 `.env` 文件推上去导致泄露？

**不会！** 项目的 `.gitignore` 第 9 行已配置排除 `.env`：

```
# .gitignore
.env    ← Git 完全忽略此文件
```

`git add -A` 时自动跳过 `.env`，push 不会包含它。

确认安全的命令：
```bash
git log --all -- .env
# 无输出 → 从未被提交过 ✅
# 有输出 → 需要清除历史记录（联系我处理）
```

---

## 九、常用命令速查

| 操作 | 命令 |
|------|------|
| 查看远程地址 | `git remote -v` |
| 查看所有分支 | `git branch -a` |
| 切换分支 | `git checkout 分支名` |
| 创建并切换分支 | `git checkout -b 新分支名` |
| 查看改动 | `git diff` |
| 暂存所有改动 | `git add -A` |
| 提交 | `git commit -m "描述"` |
| 推送到 fork | `git push origin 分支名` |
| 拉取原作者更新 | `git fetch upstream && git merge upstream/master` |
| 删除本地分支 | `git branch -d 分支名` |
| 确认 .env 安全 | `git log --all -- .env` |

---

## 当前项目远程配置

```
origin    https://github.com/KEDADA/graph-rag-agent.git    ← 你的 fork
upstream  https://github.com/1517005260/graph-rag-agent.git ← 原作者
主分支名：master
```

*创建日期：2026-02-26*
