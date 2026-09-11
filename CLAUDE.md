# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

「心语」是一个面向情感支持场景的开源 AI 对话应用：FastAPI 后端 + React 18 前端，集成情绪/意图识别、长期记忆、RAG 知识库（ChromaDB）、Agent/Skills、流式响应与自动评估。项目文档为中文优先，详细规范见 [`AGENTS.md`](AGENTS.md)（本文件的权威补充，改动前应先读它）。

**最关键的认知**：这是一个多轮演进的老项目，同一能力可能同时存在「当前分层实现 / 历史单体实现 / 实验性实现」。`backend/app.py` 的装配结果才是唯一权威的线上路径 —— 不要仅凭文件名或 README 描述判断代码是否真实运行。开始修改前先确认真实 import 链。

## 常用命令

在项目根目录执行（Windows PowerShell；项目已有 `.venv`，可用 `.venv\Scripts\python.exe` 替代 `python`）：

```powershell
# 准备本地配置（不要覆盖已有 config.env）
Copy-Item config.env.example config.env

# 安装依赖
.venv\Scripts\python.exe -m pip install -r requirements.txt
Set-Location frontend; npm install; Set-Location ..

# 启动：同时起后端 + React 开发服务器（Ctrl+C 同时停止）
.venv\Scripts\python.exe main.py

# 仅启动后端（含知识库初始化）
.venv\Scripts\python.exe run_backend.py

# 仅启动当前 FastAPI app（不做 run_backend.py 的昂贵初始化）
.venv\Scripts\python.exe -m uvicorn backend.app:app --host 127.0.0.1 --port 8000
```

测试与静态检查：

```powershell
# 单个测试文件 / 单个用例
.venv\Scripts\python.exe -m pytest backend/tests/unit/test_memory_safety.py -q
.venv\Scripts\python.exe -m pytest path/to/test_file.py::test_name -q

# 单元 / 集成 / 全部后端测试
.venv\Scripts\python.exe -m pytest backend/tests/unit -q
.venv\Scripts\python.exe -m pytest backend/tests/integration -q
.venv\Scripts\python.exe -m pytest backend/tests -q

# 语法 / 静态检查（只对改动文件跑，不顺手格式化全仓库）
.venv\Scripts\python.exe -m py_compile backend/app.py
.venv\Scripts\python.exe -m ruff check backend
.venv\Scripts\python.exe -m black --check backend
.venv\Scripts\python.exe -m mypy backend --ignore-missing-imports
```

前端：

```powershell
Set-Location frontend
$env:CI = "true"; npm test -- --watchAll=false
npm run build
```

数据库迁移（`scripts/db_manager.py` 封装 Alembic；`make db-*` 依赖 GNU Make，PowerShell 下不可原样使用）：

```powershell
python scripts/db_manager.py upgrade    # 升级到最新版本
python scripts/db_manager.py check      # 检查连接
python scripts/db_manager.py reset      # 危险：重置数据库，未经用户明确要求不要执行
```

**重要提示**：`PYTHONUTF8=1` 是 Windows 下排查应用导入/MySQL fallback 的必备环境变量（GBK 控制台可能因 Unicode 输出抛 `UnicodeEncodeError`）。

## 架构

### 权威入口

- 正式后端目标是 **`backend.app:app`**（`backend/app.py` 的 `create_app()`）。`backend/main.py` 是历史单体应用，不要当作当前应用或往上面加功能。
- `main.py`（根目录）= 本地开发编排入口，只做进程编排；`run_backend.py` = 独立后端入口（检查依赖 + 初始化 RAG 后启动）。
- `backend/app.py` 在模块导入时即执行 `app = create_app()`，会初始化聊天/RAG/意图/Agent 等服务。**测试阶段不能依赖真实 LLM/MySQL/Redis/外网**，可选依赖失败应安全降级。

### 分层结构

```
React useChat → frontend/src/services/ChatAPI.js → POST /chat/stream (multipart + SSE)
  → backend/routers/chat.py → ChatService → 情绪分析 → 意图/危机标记
  → ContextService + MemoryService → (可选 RAG) → 聊天引擎
  → MySQL/SQLite 消息 + Chroma 记忆 → SSE token/done/error
```

- `backend/routers/`：HTTP/SSE 路由，只做协议层（参数、状态码、调 service）。
- `backend/services/`：当前主聊天链路所在（ChatService/MemoryService/ContextService 等）。
- `backend/modules/`：RAG、LLM、Intent、多模态、模块化 Agent（各自有 models/routers/services）。
- `backend/agent/` 与 `backend/modules/agent/`：并存的两套 Agent 实现，改前先追踪当前 router 的真实 import。
- `backend/runtime/`：新一代 Runtime + Skills / 会话 / 策略 / 工具协议（并非全部接入主链路）。
- `backend/core/`：通用配置、异常、接口、校验工具。

### 路由地图（当前 `backend/app.py` 实际注册）

必选：`/chat`、`/memory`、`/feedback`、`/evaluation`、`/api/emotion`、`/api/personalization`、`/api/rag`；可选（import 成功时）：`/enhanced-chat`、`/agent`、`/hermes`、`/intent`、`/performance`、`/streaming`。`/ab-testing`、`/multimodal/*` 等**未注册** —— 涉及这些接口时先决定是显式注册、删除死入口还是继续维护旧单体，不要悄悄走第三条。

### 需要注意的同名/重复实现

- 根 `config.py` 与 `backend/core/config.py` 是两套配置体系（变量命名不同：`MYSQL_*` vs `DB_*`、`LLM_API_KEY` vs `OPENAI_API_KEY`）。
- `backend/models.py` 与 `backend/schemas/` 都有 Pydantic 模型；主聊天路由从 `backend.models` 导入 `ChatRequest`/`ChatResponse`。
- `backend/routers/agent.py` 与 `backend/modules/agent/routers/agent_router.py` 都声明 `/agent`，当前注册的是前者。
- `alembic/versions/` 与 `backend/migrations/*.sql` 并存；正式增量迁移优先 Alembic。

不要为了「消除重复」直接删除其中一套 —— 先证明没有导入方/脚本/文档依赖，再单独做迁移型重构。

### 版本边界（不可顺手升级）

- Pydantic **v1**（`<2.0.0`，用 `validator`，不要用 `field_validator`/`model_dump`）、SQLAlchemy **1.4**（`<2.0.0`）、ChromaDB 0.4、LangChain 0.2。
- Python 3.10/3.11 为开发目标，但 Dockerfile 与 CI 仍用 3.9（版本漂移，勿在普通功能改动中统一）。
- `requirements.txt` 是现有安装/Docker 的实际依赖来源；`pyproject.toml` 同时是元数据与工具配置 —— 增删依赖时两者都要核对。

## 不可破坏的底线

- 这是情感支持产品，**不是医疗诊断/心理治疗**。不把模型输出包装成诊断、处方或治疗结论。
- **危机安全优先**：不弱化自伤/自杀/暴力/危机意图的识别、升级与兜底逻辑（安全策略资料在 `knowledge_base/organization_policy/crisis_intervention_protocol.md`）。
- 记忆、会话、情绪、向量数据必须按 `user_id`/`session_id` 隔离，任何读写删都要校验归属。
- 外部 URL、上传文件、Hermes 工作区、工具调用都是不可信输入（防 SSRF、路径穿越、shell 注入）。
- 不提交真实对话、用户画像、密钥、数据库凭据、附件、日志、生成物（`config.env`、`.venv/`、`chroma_db/`、`data/`、`uploads/`、`log/` 等都不属于源码改动）。

## 参考文档

- [`AGENTS.md`](AGENTS.md) —— 权威、完整的工程规范（阅读顺序、安全细节、典型改动路径、完成定义）。
- [`docs/`](docs/) —— 架构与功能专题文档（Agent 架构、记忆系统、RAG、意图引擎、安全机制、生产部署等）。
- [`README.md`](README.md) —— 快速开始与环境配置说明。
