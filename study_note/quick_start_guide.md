# Graph-RAG-Agent 小白快速上手指南

> 跟着做，5 步跑起整个项目。预计耗时 **30~60 分钟**（含下载依赖）。

---

## 前置条件

在开始之前，确保你的机器上已安装以下软件：

| 软件 | 最低版本 | 检查命令 | 安装参考 |
|------|---------|----------|---------|
| Python | 3.10 | `python3 --version` | [python.org](https://www.python.org/downloads/) |
| Docker + Docker Compose | 20.x | `docker --version` | [docs.docker.com](https://docs.docker.com/get-docker/) |
| Git | 任意 | `git --version` | `sudo apt install git` |

> [!TIP]
> 推荐使用 conda 管理 Python 环境：`conda create -n graphrag python=3.10 && conda activate graphrag`

---

## 第 1 步：克隆项目 & 安装依赖

```bash
# 克隆项目
git clone <你的仓库地址> graph-rag-agent
cd graph-rag-agent

# （可选）创建 conda 虚拟环境
conda create -n graphrag python=3.10 -y
conda activate graphrag

# 安装 Linux 系统依赖（textract 需要）
sudo apt-get install -y python-dev-is-python3 libxml2-dev libxslt1-dev antiword unrtf poppler-utils

# 安装 Python 依赖
pip install -r requirements.txt

# 安装项目本身（开发模式）
pip install -e .
```

---

## 第 2 步：启动 Neo4j 数据库

```bash
# 在项目根目录执行
docker compose up -d
```

等待 **30 秒** 左右，检查 Neo4j 是否就绪：

```bash
docker compose logs neo4j | tail -5
# 看到 "Started." 字样即表示启动成功
```

验证连接：打开浏览器访问 **http://localhost:7474**，使用以下凭据登录：

| 字段 | 值 |
|------|-----|
| URI | `neo4j://localhost:7687` |
| 用户名 | `neo4j` |
| 密码 | `12345678` |

---

## 第 3 步：配置环境变量

```bash
# 复制示例配置
cp .env.example .env
```

打开 `.env` 文件，**必须修改**以下两项：

```ini
# 你的 OpenAI API Key（或兼容服务的 Key）
OPENAI_API_KEY = 'sk-你的真实Key'

# API 地址（如用官方 OpenAI 则改为：https://api.openai.com/v1）
OPENAI_BASE_URL = 'https://api.openai.com/v1'
```

> [!IMPORTANT]
> 如果你使用的是 OpenAI 兼容服务（如中转代理、本地部署的模型），请把 `OPENAI_BASE_URL` 改为对应地址。其余配置保持默认即可。

---

## 第 4 步：构建知识图谱

先把你的文档放入 `files/` 目录（支持 `.txt`、`.pdf`、`.docx` 等格式），然后执行：

```bash
python -m graphrag_agent.integrations.build.main
```

这会依次执行 3 个步骤：

| 步骤 | 做什么 | 大致耗时 |
|------|--------|---------|
| Step 1 | 读取文档 → 分块 → LLM 提取实体关系 → 存入 Neo4j | 3~5 分钟 |
| Step 2 | 实体向量索引 → 社区检测 → 社区摘要生成 | 2~3 分钟 |
| Step 3 | 文本块向量索引 | ~1 分钟 |

> [!NOTE]
> 耗时取决于文档数量和 LLM API 速度。3 个文档大约 8 分钟。构建完成后可在 Neo4j Browser（http://localhost:7474）中运行 `MATCH (n) RETURN labels(n), count(n)` 查看节点统计。

---

## 第 5 步：启动前后端服务

打开 **两个终端窗口**，分别启动后端和前端：

**终端 1 — 启动后端（FastAPI）：**

```bash
cd server
python main.py
# 看到 "Uvicorn running on http://0.0.0.0:8000" 即成功
```

**终端 2 — 启动前端（Streamlit）：**

```bash
cd frontend
streamlit run app.py
# 自动打开浏览器，地址：http://localhost:8501
```

---

## 🎉 开始使用

1. 浏览器打开 **http://localhost:8501**
2. 在左侧选择 Agent 类型（建议新手从 `naive_rag_agent` 开始）
3. 在输入框输入问题，点击发送
4. 等待回答生成

### 5 种 Agent 简介

| Agent | 特点 | 适合问题 | 速度 |
|-------|------|---------|------|
| **NaiveRAG** | 纯向量检索 | 简单事实查询 | ⚡ ~3s |
| **GraphAgent** | 知识图谱增强 | 需要关系推理的问题 | 🔄 ~8s |
| **HybridAgent** | 融合多种检索 | 通用问题 | 🔄 ~5s |
| **DeepResearchAgent** | 多轮迭代推理 | 需要深度分析的问题 | 🐢 ~30s |
| **FusionAgent** | 多智能体协作 | 复杂报告类需求 | 🐢 ~60s+ |

---

## 常见问题排查

### Q1: `docker compose up` 报错？
```bash
# 确认 Docker 正在运行
sudo systemctl start docker
# 确认当前用户在 docker 组中
sudo usermod -aG docker $USER
newgrp docker
```

### Q2: Neo4j 连接失败？
```bash
# 检查容器状态
docker compose ps
# 检查端口占用
lsof -i :7687
# 重启 Neo4j
docker compose restart neo4j
```

### Q3: `pip install` 报 textract 安装错误？
```bash
# 确保安装了系统依赖
sudo apt-get install -y python-dev-is-python3 libxml2-dev libxslt1-dev antiword unrtf poppler-utils
```

### Q4: 图构建时 LLM API 报错？
- 检查 `.env` 中 `OPENAI_API_KEY` 是否正确
- 检查 `OPENAI_BASE_URL` 是否可达：`curl <你的BASE_URL>/models`
- 确认账户余额充足

### Q5: 前端页面空白 / 报连接错误？
- 确认后端已启动（终端 1 无报错）
- 检查 `.env` 中 `FRONTEND_API_URL` 是否为 `http://localhost:8000`

---

## 项目目录结构速览

```
graph-rag-agent/
├── graphrag_agent/          # 核心代码
│   ├── agents/              #   Agent 实现（5 种 + 多智能体）
│   ├── cache_manager/       #   两层缓存系统
│   ├── community/           #   社区检测（Leiden / SLLPA）
│   ├── config/              #   配置与 Schema 定义
│   ├── evaluation/          #   评估系统（4 维度 18 指标）
│   ├── graph/               #   图构建（提取 / 消歧 / 对齐）
│   ├── integrations/build/  #   图构建入口（三步串行）
│   ├── pipelines/           #   数据管道（分块 / 摄取）
│   └── search/              #   搜索策略（本地 / 全局 / 混合 / 深度）
├── server/                  # FastAPI 后端
├── frontend/                # Streamlit 前端
├── files/                   # 📄 你的文档放这里
├── docker-compose.yaml      # Neo4j 容器配置
├── .env.example             # 环境变量模板
├── requirements.txt         # Python 依赖
└── study_note/              # 学习笔记（你现在在这里）
```

---

*最后更新：2026-03-01*
