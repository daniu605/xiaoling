# 小灵 · xiaoling

> 基于大模型的智能数据问答助手 —— 「你负责提问，小灵负责写 SQL」

小灵（xiaoling）是一个 **ChatBI / NL2SQL 智能问数系统**：用户用自然语言提问（如“华北地区销售总额”“各地区 iPhone 去年卖了多少钱”），小灵通过多轮检索与大模型推理，自动生成并执行 SQL，最终以表格形式返回查询结果，全程流式展示每一步执行进度。

## ✨ 核心特性

- **自然语言问数**：无需懂 SQL，用中文提问即可完成数据查询
- **元数据知识库**：自动将维度表、事实表、字段别名、业务指标（GMV / AOV 等）构建为可检索的元数据知识库
- **混合召回**：向量召回（Qdrant）+ 全文检索（Elasticsearch）双路召回表、字段、指标与维度值
- **流式执行过程**：SSE 实时推送“关键词提取 → 召回 → 过滤 → 生成 SQL → 校验 → 执行”每一步状态
- **SQL 自动纠错**：生成的 SQL 先校验，失败自动进入纠错节点重试，提高成功率
- **过程可视化**：聊天式界面，实时展示执行步骤与结果表格

## 🏗️ 架构与执行流程

后端基于 **LangGraph** 状态图编排，一次问答经过以下节点：

```
                    ┌─────────────────┐
                    │ extract_keywords │ 关键词提取
                    └────────┬────────┘
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │ recall_column│    │recall_metric│    │ recall_value│   三路并行召回
   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
          └──────────────────┼──────────────────┘
                             ▼
                  ┌────────────────────┐
                  │ merge_retrieved_info│ 合并召回结果
                  └─────────┬──────────┘
                     ┌──────┴───────┐
                     ▼              ▼
             ┌──────────────┐ ┌──────────────┐
             │ filter_metric│ │ filter_table │  指标/表过滤
             └──────┬───────┘ └──────┬───────┘
                    └───────┬────────┘
                            ▼
                  ┌───────────────────┐
                  │  add_extra_context │ 补充上下文
                  └─────────┬─────────┘
                            ▼
                  ┌───────────────────┐
                  │    generate_sql    │ 生成 SQL
                  └─────────┬─────────┘
                            ▼
                  ┌───────────────────┐
        否         │    validate_sql    │ SQL 校验
        ┌─────────┤                   │
        │         └─────────┬─────────┘
        ▼                   │ 校验通过
 ┌─────────────┐            ▼
 │ correct_sql │ ────► ┌─────────────┐
 │  SQL 纠错    │       │ execute_sql │ ───► 返回结果表格
 └─────────────┘       └─────────────┘
```

## 🧰 技术栈

| 层 | 技术 |
| --- | --- |
| 前端 | Vue 3（`<script setup>`）+ Vite 7 |
| 后端 | Python 3.12+ · FastAPI · Uvicorn |
| Agent 编排 | LangGraph · LangChain（DeepSeek LLM） |
| 检索 | Qdrant（向量）· Elasticsearch（全文）· jieba 分词 |
| 向量化 | BAAI/bge-large-zh-v1.5（HuggingFace Embedding） |
| 数据库 | MySQL（业务数仓 dw + 元数据库 meta）· SQLAlchemy 2 |
| 中间件部署 | Docker Compose（MySQL · Qdrant · Elasticsearch · Embedding 服务） |

## 📁 项目结构

```
xiaoling/
├── data-agent/                  # 后端服务
│   ├── main.py                  # FastAPI 启动入口（端口 8000）
│   ├── app/
│   │   ├── agent/               # LangGraph 状态图与各执行节点
│   │   │   ├── graph.py         # 状态图编排（召回→过滤→生成→校验→执行）
│   │   │   └── nodes/           # extract_keywords / recall_* / generate_sql ...
│   │   ├── api/                 # 路由与 Schema（SSE 流式问答接口 /api/query）
│   │   ├── clients/             # MySQL / ES / Qdrant / Embedding 客户端管理
│   │   ├── conf/                # 应用与元数据配置（YAML）
│   │   ├── models/              # MySQL / ES / Qdrant 数据模型
│   │   ├── prompt/              # Prompt 加载器
│   │   ├── repositories/        # 各存储的仓储层
│   │   ├── services/            # 问数服务 & 元数据知识库构建服务
│   │   └── scripts/             # build_meta_knowledge.py 知识库构建脚本
│   ├── prompts/                 # LLM 提示词模板（生成/校验/纠错 SQL 等）
│   └── pyproject.toml           # 依赖管理（uv）
└── date-agent-frontend/         # 前端服务（Vue 3 + Vite，端口 5173）
    └── src/
        └── App.vue              # 聊天式问答界面（步骤流 + 结果表格）
```

## 🚀 快速开始

### 1. 启动中间件（Docker）

```bash
cd docker
docker compose up -d
# 包含：MySQL（meta/dw 元数据与数仓）、Qdrant（向量库）、
#       Elasticsearch（全文索引）、Embedding 推理服务
```

### 2. 构建元数据知识库

```bash
cd data-agent
python app/scripts/build_meta_knowledge.py
```

### 3. 启动后端

```bash
cd data-agent
python main.py        # FastAPI 监听 0.0.0.0:8000
```

### 4. 启动前端

```bash
cd date-agent-frontend
npm install
npm run dev           # Vite 开发服务器 http://127.0.0.1:5173/
```

### 5. 试试这些问题

- 华北地区销售总额
- 2025 年各地区平均销售额
- 各地区 iPhone 去年卖了多少钱
- 各地区销量排名前三的商品

## 📌 说明

- `conf/app_config.yaml` 中的数据库、向量库、ES、LLM 等连接信息按本地环境修改
- 出于安全考虑，仓库中的 LLM `api_key` 已替换为占位符，请自行填写
- Docker 中间件首次启动耗时较长；Embedding 模型下载可能需要网络代理；如本机已有 MySQL 服务需先停止以避免端口冲突
