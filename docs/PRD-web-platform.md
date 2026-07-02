# PRD：TradingAgents Web 平台（本地 Docker 优先）

| 字段 | 值 |
|------|-----|
| 版本 | 0.1.0-draft |
| 状态 | 待开发 |
| 目标读者 | 产品负责人、AI 编码 Agent、Reviewer |
| 关联文档 | [ARCH-web-platform.md](./ARCH-web-platform.md)、[TEST-automation-web-platform.md](./TEST-automation-web-platform.md) |
| 上游依赖 | TradingAgents v0.3.0（`TradingAgentsGraph.propagate` / `save_reports`） |
| 默认 LLM | **小米 MiMo（mimo）** — 见 §5.5 |
| 远期目标 | 懒猫微服 LPK 上架（Phase 2，本 PRD 不阻塞 Phase 1） |

---

## 1. 背景与问题

TradingAgents 当前是 **Rich CLI 交互式终端**（Typer + questionary），无法在浏览器中访问，不符合懒猫微服「HTTPS Web App」交付形态。

v0.3.0 已提供无头能力：

- `TradingAgentsGraph.propagate(ticker, trade_date, asset_type)` — 跑完整多 Agent 流水线
- `TradingAgentsGraph.save_reports(final_state, ticker)` — 写出与 CLI 一致的 Markdown 报告树

**缺口**：缺少 HTTP API、任务队列、Web UI、健康检查与 Docker Web 编排。

---

## 2. 产品愿景

> 在浏览器中提交一次股票/加密货币分析请求，异步等待多 Agent 完成，在线阅读或下载完整投研报告；本地 Docker 一键启动；后续同一套镜像可打包为懒猫 LPK。

---

## 3. 目标与非目标

### 3.1 Phase 1 目标（本 PRD 范围，必须交付）

| ID | 目标 | 验收标准 |
|----|------|----------|
| G1 | 本地 Docker 可启动 Web 全栈 | `docker compose -f docker-compose.web.yml up --build` 后浏览器可访问 |
| G2 | 提交分析任务 | 表单选 ticker、日期、分析师组合、LLM 配置（读 `.env`） |
| G3 | 异步执行 | API 立即返回 `job_id`；后台跑 `propagate`；前端轮询进度 |
| G4 | 报告展示 | 完成后展示 `complete_report.md` 及分章节 Markdown |
| G5 | 健康检查 | `GET /health` 返回 200，供 Docker / 未来 LPK 探针使用 |
| G6 | CLI 不回归 | 现有 `tradingagents` CLI 与 `docker compose run tradingagents` 行为不变 |
| G7 | 自动化测试门禁 | 见 [TEST-automation-web-platform.md](./TEST-automation-web-platform.md) 中 **P1 门禁** 全部通过 |

### 3.2 Phase 2 目标（本 PRD 仅引用，不实现）

- 懒猫 LPK：`package.yml` + `lzc-manifest.yml` + `lzc-build.yml`
- OIDC 免密 / `lzc-deploy-params.yml` 注入 API Key
- 多实例、资源限制、商店提审资料

### 3.3 非目标（明确不做）

- 不提供实盘下单、券商对接
- Phase 1 不做用户注册/多租户（单用户、单实例）
- Phase 1 不内嵌 Ollama 双容器（保留现有 `docker-compose.yml --profile ollama` 供 CLI 使用）
- 不替换 CLI 为 Web-only
- 不保证 LLM 输出可复现或投资建议准确性（沿用上游免责声明）

---

## 4. 用户画像与场景

| 角色 | 场景 |
|------|------|
| 开发者/量化爱好者 | 本地 NAS / 笔记本 Docker 跑分析，避免终端交互 |
| 懒猫微服用户（Phase 2） | 浏览器打开子域名，填 Key 即用 |
| AI Agent（实施者） | 按 ARCH + TEST 文档分 Task 实现，每 Task 有可验证 AC |

---

## 5. 功能需求

### 5.1 页面（Web UI）

| ID | 功能 | 优先级 | 说明 |
|----|------|--------|------|
| UI-01 | 首页 / 新建分析 | P0 | ticker、trade_date、asset_type(stock/crypto)、分析师多选 |
| UI-02 | 运行配置摘要 | P0 | 展示当前 provider、模型名；**默认小米 MiMo（mimo）**；不展示 Key |
| UI-03 | 任务列表 | P1 | 最近 N 条 job：状态、创建时间、ticker |
| UI-04 | 任务详情 / 进度 | P0 | pending → running → succeeded/failed；running 时展示阶段 hint |
| UI-05 | 报告阅读 | P0 | Markdown 渲染 `complete_report.md`；侧边栏章节导航 |
| UI-06 | 报告下载 | P1 | 下载 zip（报告目录）或单文件 `.md` |
| UI-07 | 错误页 / Toast | P0 | API 4xx/5xx、缺 Key、job failed 可读提示 |
| UI-08 | 免责声明 | P0 | 页脚固定展示研究用途声明（摘自 README） |

### 5.2 API（REST）

Base path：`/api/v1`

| ID | Method | Path | 优先级 | 说明 |
|----|--------|------|--------|------|
| API-01 | GET | `/health` | P0 | `{ "status": "ok", "version": "0.3.0" }` |
| API-02 | GET | `/ready` | P0 | 数据目录可写、至少一个 LLM provider 已配置 |
| API-03 | GET | `/config/public` | P0 | provider、模型、已选 analyst 默认值；**无密钥** |
| API-04 | POST | `/runs` | P0 | 创建 job，body 见 ARCH §4.2 |
| API-05 | GET | `/runs` | P1 | 分页列表 |
| API-06 | GET | `/runs/{job_id}` | P0 | 状态、progress、error、report_path |
| API-07 | GET | `/runs/{job_id}/report` | P0 | `complete_report.md` 正文 |
| API-08 | GET | `/runs/{job_id}/report/sections` | P1 | 章节索引 JSON |
| API-09 | GET | `/runs/{job_id}/artifacts/{path}` | P1 | 单文件下载（路径白名单） |
| API-10 | DELETE | `/runs/{job_id}` | P2 | 取消 running / 删除 completed |

**Job 状态机**

```
queued → running → succeeded
                 → failed
                 → cancelled（P2）
```

### 5.3 后台任务

| ID | 需求 | 说明 |
|----|------|------|
| JOB-01 | 单进程内存队列 | Phase 1 足够；并发默认 1（可 env 配置） |
| JOB-02 | 调用现有 Graph | `TradingAgentsGraph(...).propagate()` → `save_reports()` |
| JOB-03 | 持久化 job 元数据 | SQLite `jobs.db` 于 `TRADINGAGENTS_DATA_DIR` |
| JOB-04 | 进度回调 | 复用 CLI `StatsCallbackHandler` 或轻量 stage 枚举推送到 job 表 |
| JOB-05 | 超时 | 默认 30min，`TRADINGAGENTS_JOB_TIMEOUT_SEC` 可配 |
| JOB-06 | 失败可诊断 | 存 traceback 摘要到 job.error |

### 5.4 配置与环境

| 变量 | 必填 | 说明 |
|------|------|------|
| `MIMO_API_KEY` | **本地/联调默认必填** | 小米 MiMo API Key（[官方文档](https://mimo.mi.com/docs/zh-CN/quick-start/summary/first-api-call)） |
| `OPENAI_COMPATIBLE_API_KEY` | 否 | 与 `MIMO_API_KEY` 同步；TradingAgents `openai_compatible` 实际读取此变量 |
| `TRADINGAGENTS_LLM_PROVIDER` | 否 | 默认 `openai_compatible`（对接 MiMo） |
| `TRADINGAGENTS_DEEP_THINK_LLM` | 否 | 默认 `mimo-v2.5-pro` |
| `TRADINGAGENTS_QUICK_THINK_LLM` | 否 | 默认 `mimo-v2.5-pro` |
| `TRADINGAGENTS_LLM_BACKEND_URL` | 否 | 默认 `https://api.xiaomimimo.com/v1` |
| `TRADINGAGENTS_OUTPUT_LANGUAGE` | 否 | 默认 `Chinese` |
| `TRADINGAGENTS_DATA_DIR` | 否 | 默认 `/data`（Docker volume） |
| `TRADINGAGENTS_WEB_HOST` | 否 | 默认 `0.0.0.0` |
| `TRADINGAGENTS_WEB_PORT` | 否 | 默认 `8080` |
| `TRADINGAGENTS_JOB_CONCURRENCY` | 否 | 默认 `1` |
| `TRADINGAGENTS_MOCK_GRAPH` | 否 | 默认 `0`；CI/无 Key 冒烟设为 `1` |
| `TRADINGAGENTS_CORS_ORIGINS` | 否 | 本地 dev 默认 `http://localhost:5173` |
| 其他 `*_API_KEY` | 否 | 可选；切换 provider 时使用（见 `.env.example`） |

### 5.5 默认 LLM：小米 MiMo（mimo）

> **mimo** = **小米公司 MiMo 大模型 API**（[Xiaomi MiMo 开放平台](https://mimo.mi.com)），**不是** MiniMax（稀宇科技）。

TradingAgents 上游暂无独立 `xiaomi` provider，Web 层通过 **`openai_compatible`** + MiMo 官方 OpenAI 兼容端点接入。

| 项 | 默认值 | 理由 |
|----|--------|------|
| Provider | `openai_compatible` | MiMo 兼容 OpenAI Chat Completions API |
| Endpoint | `https://api.xiaomimimo.com/v1` | 按量付费默认 Base URL |
| 模型 | `mimo-v2.5-pro` | 官方当前推荐（MiMo-V2 系列已下线，须用 V2.5） |
| 报告语言 | `Chinese` | 国内测试可读 |
| 密钥 env | `MIMO_API_KEY` | 小米官方命名；webapi 同步至 `OPENAI_COMPATIBLE_API_KEY` |

**Token Plan 用户（`tp-` 密钥）**

| 项 | 值 |
|----|-----|
| `TRADINGAGENTS_LLM_BACKEND_URL` | `https://token-plan-cn.xiaomimimo.com/v1` |
| `MIMO_API_KEY` | Token Plan 专属 Key |

**升级路径（可选）**

| 场景 | 调整 |
|------|------|
| 国际节点 | 文档另有 `token-plan-sgp.xiaomimimo.com`（按需） |
| 换其他 LLM | 改 `TRADINGAGENTS_LLM_PROVIDER` + 对应 Key（见 `.env.example`） |

**Web `/api/v1/config/public` 必须回显（无密钥）**

```json
{
  "llm_provider": "openai_compatible",
  "llm_label": "小米 MiMo (mimo)",
  "deep_think_llm": "mimo-v2.5-pro",
  "quick_think_llm": "mimo-v2.5-pro",
  "backend_url": "https://api.xiaomimimo.com/v1",
  "output_language": "Chinese",
  "mock_graph": false
}
```

**AI 实施硬约束**

1. `webapi/config.py` 启动时：`MIMO_API_KEY` → 回填 `OPENAI_COMPATIBLE_API_KEY`（若后者为空）。
2. `webapi/config.py` 与 `docker-compose.web.yml` 默认值必须与上表一致。
3. MiMo 认证支持 `api-key` 与 `Authorization: Bearer` 两种头；若 LangChain 默认 Bearer 失败，在 `webapi` 层为 MiMo 增加 `api-key` 头注入（见 ARCH §4.0.5）。
4. CI 仍用 `TRADINGAGENTS_MOCK_GRAPH=1`，**不消耗** MiMo 额度。

---

## 6. 非功能需求

| 类别 | 要求 |
|------|------|
| 性能 | 创建 job API P99 < 500ms（不含 LLM）；报告读取 < 2s |
| 安全 | API Key 永不进响应/日志；artifact 路径防目录穿越 |
| 兼容 | Python 3.12（Docker 主镜像）；前端 ES2020+ |
| 可观测 | 结构化日志 JSON 可选；job_id 贯穿 request log |
| 国际化 | Phase 1 中英双语 UI 标签（分析语言仍走 `TRADINGAGENTS_OUTPUT_LANGUAGE`） |

---

## 7. 信息架构（IA）

```
/                     → 新建分析 + 最近任务
/runs                 → 任务列表
/runs/:id             → 进度 + 报告
/runs/:id/report      → 全屏报告阅读
```

---

## 8. 里程碑与 AI 实施顺序

> **AI Agent 必须按序完成；每里程碑跑 TEST 文档对应门禁后再进入下一步。**

| 里程碑 | 交付物 | 门禁 |
|--------|--------|------|
| M0 | 三文档评审通过（本文档集） | 人工 |
| M1 | `webapi/` FastAPI 骨架 + `/health` `/ready` | TEST §3.1 |
| M2 | Job 模型 + POST/GET runs + 内存 worker 调 Graph（mock LLM 测试） | TEST §3.2 |
| M3 | `webui/` 新建任务 + 轮询 + 报告页 | TEST §3.3 |
| M4 | `docker-compose.web.yml` 全栈本地跑通 | TEST §3.4 |
| M5 | CI 接入 api + e2e 流水线 | TEST §3.5 |
| M6（Phase 2） | LPK 三件套 + lazycat install | lazycat-porting skill |

---

## 9. 验收标准（Phase 1 完成定义）

1. 克隆仓库，复制 Web 默认环境：

   ```bash
   cp docs/env/web.defaults.env .env
   # 编辑 .env，填入 MIMO_API_KEY（小米 MiMo 控制台）
   ```

2. 执行：

   ```bash
   docker compose -f docker-compose.web.yml up --build -d
   curl -sf http://localhost:8080/health
   curl -sf http://localhost:8080/ready
   curl -sf http://localhost:8080/api/v1/config/public | grep openai_compatible
   ```

3. 浏览器打开 `http://localhost:8080`，确认配置摘要显示 **小米 MiMo (mimo)**；提交 `600519.SS`（或 `AAPL`）+ 今日日期，等待 job `succeeded`。
4. 页面可见 **中文** Markdown 报告，磁盘 `TRADINGAGENTS_DATA_DIR/reports/` 有对应目录。
5. `pytest tests/webapi` + `pytest tests/integration/web` + `cd webui && npm run test:e2e`（mock 模式）全绿。
6. 原 CLI：`docker compose run --rm tradingagents --help` 仍正常。

---

## 10. 风险与缓解

| 风险 | 缓解 |
|------|------|
| LLM 单次运行 >30min | 可调超时；UI 明确「长时间运行」预期 |
| 网关/proxy 超时（Phase 2） | 异步 job + 轮询/SSE；LPK 路由调大 timeout |
| Token 成本高 | 默认 mimo `mimo-v2.5-pro`；UI 展示「深度」说明；默认 analyst 可减至 `market+news` |
| Graph 非线程安全 | worker 并发=1；Phase 2 再考虑多 worker 进程 |

---

## 11. 开放问题（AI 实施时可默认）

| 问题 | 默认决策 |
|------|----------|
| 默认 LLM | **小米 MiMo（mimo）**（§5.5） |
| SSE vs 轮询 | Phase 1 **轮询 3s**；SSE 为 P2 |
| 前端框架 | React + Vite + TypeScript（见 ARCH） |
| 是否 SSR | 否，SPA + FastAPI 静态托管 |

---

## 12. 文档维护

- 功能变更先改 PRD AC，再改 ARCH，最后改 TEST。
- 每个 PR 描述须引用 PRD ID（如 `API-04`）并标注里程碑。
