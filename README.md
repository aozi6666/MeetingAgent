# Meeting Multi-Agent · Review Decision Agent

> 面向研发评审场景的 **离线会议决策提取与检索平台**。上传评审录音 → 转写 → Multi-Agent 并行提取纪要/行动项/风险/决策 → 决策入库向量化 → AI 对话双路 RAG 召回。

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                           前端（React）                           │
│  会议管理 │ 纪要详情 │ 决策库 │ AI 对话 │ 知识库 │ Agent 监控     │
└──────────────┬───────────────────────────────────┬───────────────┘
               │ HTTP / SSE                         │
┌──────────────▼───────────────────────────────────▼───────────────┐
│                        后端（FastAPI）                            │
├───────────┬───────────┬───────────┬───────────┬──────────────────┤
│ meetings  │ summaries │ decisions │   chat    │  agent_runs API  │
├───────────┴───────────┴───────────┴───────────┴──────────────────┤
│                         Service Layer                            │
│  MeetingService │ TranscriptionService（ASR + Mock 降级）         │
│  SummaryService │ DecisionGraphService │ ChatService              │
│  KnowledgeService（向量 + 全文 + RRF）│ EmbeddingService          │
├──────────────────────────────────────────────────────────────────┤
│                  Multi-Agent（LangGraph v2）                      │
│  [planner] → [budget_check] → fan-out（动态路由）                 │
│    ├─ summary_agent      ┐                                        │
│    ├─ action_items_agent │ 并行（套 Harness）                     │
│    ├─ risks_agent        │                                        │
│    └─ decision_extractor ┘                                        │
│  → [output_validator] → [human_review] → [persist]               │
├──────────────────────────────────────────────────────────────────┤
│                    决策抽取两步流水线                              │
│  DecisionDetector（全量扫描定位）→ OptionExtractor（结构化抽取）  │
├──────────────────────────────────────────────────────────────────┤
│                    双路 RAG 召回（AI 对话）                       │
│  文档路（knowledge_service）+ 决策路（decision_graph_service）     │
│  → RRF 融合 → top-5 上下文 → LLM 流式输出                         │
└──────────────┬───────────────────────────────────┬───────────────┘
               │                                    │
┌──────────────▼──────────────────┐ ┌──────────────▼───────────────┐
│   PostgreSQL + pgvector         │ │           Redis              │
│ meetings / transcripts /        │ │ session / query cache        │
│ summaries / action_items /      │ │ transcribe task state        │
│ risks / decisions /             │ │ agent run metrics            │
│ decision_options /              │ │ rate limit / lock            │
│ decision_relations /            │ │                              │
│ knowledge_documents /           │ │                              │
│ agent_runs / chat_sessions      │ │                              │
└─────────────────────────────────┘ └──────────────────────────────┘
```

---

## Core Capabilities

### Offline Meeting Pipeline

```
音频上传 → TranscriptionService（DashScope ASR / Mock 降级）
         → 存储 Transcript（含说话人 + 时间戳）
         → Multi-Agent 并行（LangGraph）
            ├─ 摘要 Agent        → Summary
            ├─ 行动项 Agent      → ActionItem[]
            ├─ 风险 Agent        → Risk[]
            └─ 决策抽取 Agent    → Decision[]（两步流水线）
         → 落库 + 决策向量化 + 即时关联 top-3 历史决策
```

### Two-Stage Decision Extraction Pipeline

```mermaid
flowchart LR
    A["完整 Transcript"] --> B["Step 1 · DecisionDetector<br/>LLM 全量扫描定位决策段<br/>区分 决策 / 提议 / 推迟<br/>confidence ≥ 0.7 过滤"]
    B --> C["候选决策段<br/>（0~N 个）"]
    C --> D["Step 2 · OptionExtractor<br/>逐段结构化抽取"]
    D --> E["结构化 Decision<br/>标题 / 背景 / 候选方案<br/>已选方案 / 理由 / 反对意见 / 决策人"]
```

### Harness Constraint Framework

每个 Agent 节点由 `harness_wrap` 包裹，提供：

- **BudgetGuard** — Token / 成本预算，超限抛 `BudgetExceededError`
- **CircuitBreaker** — 连续失败熔断
- **RetryPolicy** — 可配置重试次数与退避
- **OutputValidator** — 结构化校验 + 回灌重试
- **AgentRun** — 节点耗时 / Token / 成本 / Tool 调用全生命周期记录

ContextVar 跨节点传递 `run_id` / `budget_guard`，不侵入 LangGraph state。

### Dual-Path RAG Retrieval

AI 对话检索时并行召回两路结果，RRF 融合后送入 LLM：

```
用户提问
  ├─ 文档路：knowledge_service.search()（纪要 + 知识文档，向量 + 全文 + RRF）
  └─ 决策路：decision_graph_service.search()（决策库 pgvector 语义检索）
  → RRF 融合（k=60）→ top-5 上下文
  → LLM 流式输出（SSE）
```

---

## Feature Modules

| 模块 | 能力 |
|------|------|
| 会议管理 | 创建会议、音频上传、自动转写 |
| 智能纪要 | Multi-Agent 生成纪要 / 行动项 / 风险 |
| 决策库 | 结构化决策 + 语义搜索 + 历史关联 |
| AI 对话 | 双路 RAG 流式问答 |
| 知识库 | 文档解析 + 向量检索 |
| Agent 监控 | 运行统计 + 预算 + 审批 |

---

## Tech Stack

### Backend

| 领域 | 技术 |
|---|---|
| Web 框架 | FastAPI 0.115 + Uvicorn |
| 数据库 | PostgreSQL 16 + pgvector（cosine, ivfflat, 1024 维） |
| ORM | SQLAlchemy 2.0（async）+ Alembic |
| 缓存 | Redis 7 |
| Agent 编排 | LangGraph 0.2（StateGraph + 条件路由 + 并行 fan-out） |
| LLM | 通义千问 qwen-plus（兼容 OpenAI 接口） |
| Embedding | text-embedding-v3（1024 维） |
| ASR | DashScope Paraformer-v2（录音文件识别，OSS 中转） |
| 文档解析 | pypdf / python-docx |

### Frontend

| 领域 | 技术 |
|---|---|
| 框架 | React 19 + TypeScript |
| 构建 | Vite 8 |
| 服务端状态 | TanStack Query 5 |
| UI 状态 | Zustand 5 |
| 路由 | React Router 7（懒加载） |
| 样式 | Tailwind CSS 4 |
| Markdown | react-markdown + remark-gfm + rehype-highlight |
| 流式通信 | @microsoft/fetch-event-source（SSE） |
| 虚拟滚动 | @tanstack/react-virtual |

### Infrastructure

| 领域 | 技术 |
|---|---|
| 容器化 | Docker Compose（PostgreSQL + pgvector + Redis） |
| 向量数据库 | pgvector（PostgreSQL 扩展） |

---

## Project Structure

```
MeetingAgent/
├── backend/
│   ├── app/
│   │   ├── agents/
│   │   │   ├── harness/              # Harness 约束框架
│   │   │   │   ├── budget.py         # BudgetGuard（Token/成本预算）
│   │   │   │   ├── circuit_breaker.py# CircuitBreaker（熔断）
│   │   │   │   ├── retry.py          # RetryPolicy（重试退避）
│   │   │   │   ├── validator.py      # OutputValidator（结构校验）
│   │   │   │   └── wrap.py           # harness_wrap 装饰器 + ContextVar
│   │   │   ├── nodes/
│   │   │   │   ├── planner.py        # Planner（会议类型识别 + 动态调度）
│   │   │   │   ├── budget_check.py   # 文本压缩 + 预算预检
│   │   │   │   ├── decision_detector.py  # Step 1：决策段定位
│   │   │   │   ├── option_extractor.py   # Step 2：结构化选项抽取
│   │   │   │   ├── decision_extractor.py # 两步流水线组合节点
│   │   │   │   ├── output_validator.py   # 校验 + 回灌重试
│   │   │   │   └── human_review.py       # 高风险人工审批
│   │   │   ├── tools/                # Tool Registry（Agent 工具调用）
│   │   │   ├── meeting_graph.py      # v1 三 Agent 并行图
│   │   │   └── meeting_graph_v2.py   # v2 Harness 版（四路并行）
│   │   ├── api/
│   │   │   ├── meetings.py           # 会议 CRUD + 音频上传
│   │   │   ├── summaries.py          # 纪要生成 + 行动项 + 风险
│   │   │   ├── decisions.py          # 决策库（列表/详情/搜索）
│   │   │   ├── chat.py               # AI 对话（SSE 流式）
│   │   │   ├── knowledge.py          # 知识库检索 + 文档管理
│   │   │   ├── agent_runs.py         # Agent 运行监控 + 审批
│   │   │   └── rooms.py              # 实时房间管理（对接 mediasoup SFU）
│   │   ├── services/
│   │   │   ├── transcription_service.py  # 转写（DashScope + Mock 降级）
│   │   │   ├── dashscope_asr_service.py  # DashScope 原生 ASR
│   │   │   ├── oss_service.py            # 阿里云 OSS（音频中转）
│   │   │   ├── summary_service.py         # Agent 工作流协调 + 落库
│   │   │   ├── decision_graph_service.py  # 决策入库 + 向量关联 + 检索
│   │   │   ├── chat_service.py            # AI 对话（双路 RAG + RRF）
│   │   │   ├── knowledge_service.py       # 知识检索（向量 + 全文 + RRF）
│   │   │   ├── embedding_service.py       # 文本向量化
│   │   │   ├── agent_run_service.py       # AgentRun 生命周期
│   │   │   ├── document_parser.py         # 文档解析（PDF/DOCX）
│   │   │   └── document_chunker.py        # 文档分块
│   │   ├── models/
│   │   │   ├── decision.py           # Decision / DecisionOption / DecisionRelation
│   │   │   ├── agent_run.py          # AgentRun（含 steps / node_usage）
│   │   │   ├── meeting.py / transcript.py / summary.py
│   │   │   ├── action_item.py / risk.py
│   │   │   ├── knowledge_doc.py / chat.py
│   │   │   └── room.py / realtime_session.py
│   │   ├── schemas/                  # Pydantic 响应模型
│   │   ├── db/                       # 数据库连接
│   │   └── config.py                 # 配置管理
│   ├── alembic/versions/             # 数据库迁移
│   ├── scripts/                      # 测试脚本
│   ├── seed_data.py                  # 示例数据
│   ├── requirements.txt
│   └── .env.example
├── frontend/
│   └── src/
│       ├── features/
│       │   ├── meetings/             # 会议管理（列表/详情/音频上传/转写）
│       │   ├── summaries/            # 纪要（列表/详情/行动项/风险）
│       │   ├── decisions/            # 决策库（列表/详情/语义搜索/分页）
│       │   ├── chat/                 # AI 对话（流式 + 虚拟滚动）
│       │   ├── knowledge/            # 知识库（上传/检索/管理）
│       │   └── agent-runs/           # Agent 监控（统计/步骤/审批）
│       ├── components/
│       │   ├── layout/               # AppLayout / Sidebar / Header
│       │   └── ui/                   # Button / Card / Badge / Markdown 等
│       ├── api/                      # ky HTTP 客户端
│       ├── hooks/                    # 虚拟列表 / 语音
│       ├── lib/                      # constants / utils / query-client
│       ├── router/                   # 路由配置（懒加载）
│       └── types/                    # TypeScript 类型定义
├── sfu/                              # mediasoup SFU（实时音视频信令 + 媒体转发）
├── docker-compose.yml                # PostgreSQL + pgvector + Redis
└── README.md
```

---

## Data Model

### Decision Tables (Core)

```mermaid
erDiagram
    MEETINGS ||--o{ DECISIONS : "一次会议产出多个决策"
    DECISIONS ||--o{ DECISION_OPTIONS : "一个决策含多个候选方案"
    DECISIONS ||--o{ DECISION_RELATIONS : "决策间语义关联(source/target)"

    DECISIONS {
        uuid id PK
        uuid meeting_id FK "可空,支持跨会议决策"
        varchar title "决策标题"
        text context "决策背景"
        text snippet "决策段原文"
        varchar chosen_option "已选方案"
        jsonb reasons "选择理由"
        jsonb objections "反对意见"
        jsonb decided_by "决策人"
        float confidence "置信度"
        vector embedding "1024维向量,ivfflat cosine"
    }

    DECISION_OPTIONS {
        uuid id PK
        uuid decision_id FK "级联删除"
        varchar name "方案名"
        jsonb pros "优点"
        jsonb cons "缺点"
        varchar proposed_by "提议人"
        boolean is_chosen "是否被选"
    }

    DECISION_RELATIONS {
        uuid id PK
        uuid source_decision_id FK "源决策"
        uuid target_decision_id FK "目标决策"
        varchar relation_type "默认relates"
        text context "关联说明"
        float similarity_score "向量相似度"
    }
```

- Other Core Tables： `meetings` / `transcripts` / `summaries` / `action_items` / `risks` / `knowledge_documents` / `agent_runs` / `chat_sessions` / `chat_messages`

---

## Quick Start

### Prerequisites

- Python 3.11+
- Node.js 18+
- Docker（用于 PostgreSQL + Redis）

### 1. Start Infrastructure

```bash
docker compose up -d
```

### 2. Start Backend

```bash
cd backend

# 创建虚拟环境
python -m venv .venv
source .venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env，填入 OPENAI_API_KEY（通义千问 DashScope）

# 数据库迁移
PYTHONPATH=. alembic upgrade head

# 灌入示例数据（可选）
python seed_data.py

# 启动服务
uvicorn app.main:app --reload --port 8787
```

### 3. Start Frontend

```bash
cd frontend
npm install
npm run dev
```

### 4. Access the Application

- 前端：`npm run dev` 后浏览器打开 5173 端口
- API 文档：后端启动后访问 `/docs`

---

## Configuration

编辑 `backend/.env`：

```bash
# 数据库
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/yuan_meet

# LLM（通义千问，兼容 OpenAI 接口）
OPENAI_API_KEY=your-dashscope-api-key
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL=qwen-plus
EMBEDDING_MODEL=text-embedding-v3
EMBEDDING_DIMENSIONS=1024

# 音频转写
# auto: 优先真实转写，未配置 OSS 时自动降级 Mock（默认）
# mock: 仅 Mock（零配置本地开发）
# dashscope: 仅真实转写
TRANSCRIPTION_PROVIDER=auto

# 阿里云 OSS（真实转写需要，留空自动降级 Mock）
# OSS_ACCESS_KEY_ID=
# OSS_ACCESS_KEY_SECRET=
# OSS_ENDPOINT=https://oss-cn-hangzhou.aliyuncs.com
# OSS_BUCKET_NAME=
```

**音频转写**：额外配置 4 个 OSS 环境变量，音频上传 OSS → DashScope ASR 公网回调 → 轮询解析（含说话人 + 时间戳）→ 清理 OSS。

---

## License

MIT
