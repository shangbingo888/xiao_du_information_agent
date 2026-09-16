# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## 常用命令

### 启动中间件（PostgreSQL / Redis / Milvus / ES / MinIO）
```bash
./start-services.sh start      # status | stop | restart | logs [服务名] | clean
# 等价写法：docker compose up -d
```
首次开发前必须执行。脚本封装了 `docker-compose.yml`（根目录），并做 pg_isready / redis-cli / Milvus 9091 / ES 1200 健康检查。ES 对外端口是 1200（容器内 9200）。

### 后端安装与启动
```bash
cd backend
pip install -r requirements.txt
python app/app_main.py     # 必须在 backend/ 目录下执行
```
`app_main.py` 以脚本方式运行，Python 会把 `backend/app` 加入 `sys.path`，**所有 import 都是顶层绝对导入**（`from router import ...`、`from service.xxx import ...`），在其它目录启动会 ModuleNotFoundError。服务监听 `http://localhost:8000`，Swagger 在 `/docs`。启动时 `Base.metadata.create_all` 自动建表。

### 前端安装与启动
```bash
cd frontend
npm install --legacy-peer-deps   # 必须带 --legacy-peer-deps，否则依赖冲突
npm run dev                      # http://localhost:5183（vite.config.ts 里是 5183，README 写的 5173 已过时）
```

### 前端构建与检查
```bash
npm run build     # vite build，产物到 frontend/dist
npm run lint      # eslint .（flat config: eslint.config.js）
npm run preview   # 预览 dist 产物
```
仓库没有配 `tsc --noEmit` 脚本；类型检查用 `npx tsc -p tsconfig.app.json --noEmit`。`tsconfig.app.json` 开了 `strict` + `noUnusedLocals` + `noUnusedParameters`，构建前注意未使用变量会报错（TS 层面，非 ESLint）。

### 后端测试
仓库**没有测试套件**（`backend/test/` 只有 fixture：`test_doc.pdf`、`数据.xlsx`、几张图表 PNG）。没有 pytest 配置。验证手段是手动打接口或用 `app/scripts/test_deep_research_v2.py` 直跑研究流水线：
```bash
cd backend && python app/scripts/test_deep_research_v2.py
```

### 冒烟接口（后端已启动）
```bash
curl -N -X POST http://localhost:8000/research/stream -H "Content-Type: application/json" \
  -d '{"query":"安责险在矿山行业的应用现状","max_iterations":2}'
curl -X POST http://localhost:8000/documents/upload -F "file=@./test/test_doc.pdf"
```

### 环境变量
`cp backend/.env.example backend/.env` 后填写。必填：`DASHSCOPE_API_KEY`（LLM + Embedding）、`BOCHA_API_KEY`（博查搜索）、`JWT_SECRET_KEY`。可选：`DOCMIND_*`（阿里云文档解析）、`BID_*`（招投标）、`JUHE_STOCK_API_KEY`、`OPENROUTER_API_KEY`。DB/Redis/Milvus 的默认值已与 `docker-compose.yml` 对齐，本地用 Docker 时无需改。

---

## 架构总览

这是一个 **AI 深度研究助手**（行业信息助手）：FastAPI 后端 + React 19 前端，后端核心是一套多智能体（multi-agent）研究流水线，通过 SSE 向前端流式推送研究过程。

### 后端分层（`backend/app/`）

`router/` → `service/` → `models/` + 外部中间件。注意**没有 repository 层**，`service/` 直接持有 SQLAlchemy / Milvus / Redis 客户端。

- **`app_main.py`**：入口。`load_dotenv()` → `Base.metadata.create_all(bind=engine)` → 注册 11 个 router → `lifespan` 里启动/停止 `scheduler_service`（新闻采集定时任务）。CORS 全开。
- **`router/`**：每个文件一个 `APIRouter`，被 `app_main.py` 显式 include。前缀：`/auth`、`/sessions`、`/chat`、`/research`、`/knowledge-bases`、`/documents`、`/search`、`/memories`、`/database`、`/news`、`/attachments`。
- **`models/`**（SQLAlchemy 声明式，`core/database.py` 的 `Base` + `get_db()` 依赖）：`User`；`ChatSession/ChatMessage/ChatAttachment/LongTermMemory`；`KnowledgeBase/Document`；`IndustryStats/CompanyData/PolicyData`；`ResearchCheckpoint`；`IndustryNews/BiddingInfo/NewsCollectionTask`。`docker/init-db/01-init.sql` 是容器首次启动的建表脚本，`create_all` 是运行时兜底——两者需保持一致。
- **`core/`**：`database.py`（engine/SessionLocal/get_db）、`redis_client.py`（连接池 + 全局 `cache` 单例）、`security.py`（python-jose JWT + passlib bcrypt）。
- **`config/`**：`llm_config.py` 是 **Agent 模型总配置**，dataclass 单例 `get_config()`，每个 Agent 一个 `AgentModelConfig(model, temperature, max_tokens)`，默认模型 `deepseek-v3.2`（scout 用 `qwen-plus`），`ResearchConfig` 控制 `max_iterations=1` / `quality_threshold=6.0` / `max_charts`。改模型或调参**只改这个文件**。`industry_config.py` 定义 4 个行业（智慧交通/金融科技/医疗健康/能源电力）的搜索关键词。

### 深度研究：V2 是当前主链路

**`service/deep_research_v2/`** 是核心，6 个专家 Agent 协作：

| Agent | 文件 | 职责 |
|---|---|---|
| `ChiefArchitect` | `agents/architect.py` | 拆解问题、生成大纲/研究问题/假设、构建知识图谱 |
| `DeepScout` | `agents/scout.py` | 网络搜索（博查）+ 本地知识库检索，产出 `Fact` |
| `DataAnalyst` | `agents/data_analyst.py` | 从原始材料抽取结构化 `DataPoint` |
| `CodeWizard` | `agents/wizard.py` | 生成并执行 Python，产出图表 |
| `CriticMaster` | `agents/critic.py` | 对抗式质检，产出 `CriticFeedback` |
| `LeadWriter` | `agents/writer.py` | 分段撰写与修订，输出最终报告 |

**关键实现细节（容易踩坑）**：`graph.py` 里 `_build_langgraph()` 声明了完整的 LangGraph 状态机（节点 `plan/research/analyze/write/review/revise`，条件边 `review --_should_revise--> revise|END`，`revise→review` 回环），但 **`run()` 中 `_run_with_langgraph()` 的调用被注释掉了，实际走 `_run_simplified()`**（原因：LangGraph 的 astream 批量吐消息，无法实时流式）。所以：

- 改流程逻辑要改 **`_run_simplified()`**，不是改图声明。实际顺序是 `architect → scout → data_analyst → wizard → writer`，然后 `while iteration < max_iterations` 循环 `critic`，按 `state["phase"]` 分流：`RE_RESEARCHING` → 再跑 scout + writer；`REVISING` → 只跑 writer；`COMPLETED` → break。
- 状态是 `state.py` 里的 `ResearchState`（TypedDict，全局工作记忆），`ResearchPhase` 是阶段枚举。共享状态靠 `state["messages"]`（SSE 事件）和 `state["_message_queue"]`（`asyncio.Queue`，`run_agent_with_streaming` 每 0.5s 轮询取出来 yield）。
- Agent 统一通过 `BaseAgent.add_message(state, event_type, content)` 写事件；消息体为 `{type, agent, timestamp, content}`。
- **取消机制**：`research_router` 用 Redis 键 `research:cancel:{session_id}`（TTL 300s）做标志，`graph` 在每个 Agent 前后检查并 `task.cancel()`。
- `service.py` 的 `DeepResearchV2Service.research()` 把事件格式化为 `data: {json}\n\n`，结束发 `data: [DONE]\n\n`。

### V1（legacy）仍然存在

`service/dr_g.py`（`ResearchService`）+ `react_controller.py`（`ReActController`：Plan→Execute并行→Reflect 循环）+ `tool_executor.py`（8 个工具：web_search / knowledge_search / text2sql / data_analyzer / chart_generator / stock_query / bidding_search / finish）。**V1 仍接线**：`research_router` 按 `version` 分发——`POST /research/stream` 默认 v2，`GET /research/stream` 默认 v1，`POST /research/resume/{id}` 强制 v2。V2 失败会降级吗？不会自动降级，但 V1 内部 `_research_with_react()` 失败会降级到 `_research_classic()`。新功能一律写 V2。

### 其它 Service

- `checkpoint_service.py`：`CheckpointService` 单例，**自建 `SessionLocal()`，不走 `get_db()`**。持久化 `ResearchCheckpoint`：`state_json`（清洗后的 ResearchState）+ `ui_state_json`（研究步骤/搜索结果/图表/知识图谱/流式报告）+ `final_report`。每阶段结束落盘，`POST /research/resume/{session_id}` 取回状态续跑（注意是阶段级恢复，非精确断点）。
- `milvus_service.py`：pymilvus 单例 `get_milvus_service()`，消费者为 `docmind_service`（写入）、`retrieval_service.retrieve_content`、`deep_research_v2/agents/scout.py`（本地知识库）、`memory_service`（集合 `long_term_memories`）。
- `news_collection_service.py` + `scheduler_service.py`：按 `industry_config` 的关键词定时采集行业新闻/招投标。
- `text2sql_service.py` / `database_explorer.py` / `chart_generator.py` / `smart_analyzer.py`：数据库问答与图表。

### 前端（`frontend/src/`）

React 19 + TypeScript + Vite 6 + Ant Design 5 + React Router 6（`createBrowserRouter`）+ **valtio** 状态管理。

- 路由：`router/routes.tsx`。`/login` 独立；其余路径包在 `AuthGuard`（读 `authState.isLoggedIn`，未登录跳 `/login`）→ `BaseLayout` → `Outlet`。页面：`/chat`（`:id` 是会话详情，空 path 是 `newchat`）、`/knowledge`、`/memory`、`/database`、`/news`、`/bidding`。
- 网络层：`api/request/` 是自研 axios 插件链（`installPlugins`），顺序为 `auth → service → loading → repeat → error-toast`。
  - **`service` 插件是后端契约**：响应体里若含 `status` 字段，则 `status !== 'success'` 一律 reject 成 `ResponseError`。后端新接口若返回 `{status: "success", ...}` 会被自动判定，不遵循则会漏判。
  - `auth` 插件注入 JWT；`repeat` 插件做重复请求去重。
  - SSE 请求统一用 `{ responseType: 'stream', adapter: 'fetch', headers: {Accept: 'text/event-stream'} }`——见 `api/session.ts` 的 `chat()` / `deepsearch()` / `chatWithAttachments()`。
- 状态：`store/` 下 valtio proxy（`auth.ts` / `session.ts` / `industry.ts` / `knowledge.ts` / `device.ts`），`valtio-persist.ts` 提供 localStorage 持久化包装，`auth.ts` 用 `subscribe` 自动落盘。
- **`pages/chat/index.tsx` 是一个 ~1100 行的巨型组件**，负责消费 SSE 并按 `json.type` 分发到 UI：V2 事件（`research_start` / `research_step` / `search_results` / `knowledge_graph` / `charts` / `phase` / `outline` / `report_draft` / `review` / `revision_complete` / `research_complete` / `research_cancelled` / `error`）与 V1 ReAct 事件（`react_start` / `plan` / `thought` / `action` / `observation` / `section_draft`）**两套都在这里处理**。加新事件类型要同步改这个文件。终止信号是字符串 `[DONE]`。
- 路径别名 `@/*` → `src/*`（`vite.config.ts` 的 alias + `tsconfig.app.json` 的 paths，两处都要有）。

### 开发约定

- 所有源文件顶部带版权头注释（`Copyright © 2026 深圳市深维智见教育科技有限公司 版权所有`），新增文件保持一致。
- 后端模块内 import 一律用顶层绝对路径（`from service.foo import ...`），不加 `app.` 前缀。
- `frontend/.env` 里 `VITE_API_BASE` / `VITE_API_PROXY` 决定 vite proxy 的前缀与目标，默认都指向 `http://localhost:8000/`。
