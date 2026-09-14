# MVP 信息架构与 API 清单（Codex 多租户 Web 平台）

> 状态：已拍板，**先设计、后脚手架**。  
> 原则：Web 控制面学 WeKnora；Coding 执行用 Codex；Penguin 工场二期；不整仓 fork WeKnora。

## 1. 产品一句话

给团队用的 **多租户 Coding Agent 平台**：在浏览器里开空间、起任务，后端用 **Docker 容器跑 Codex**，网页看事件流与结果；平台统一管模型网关与配额。

交付物：**网址**，不是桌面 App。

## 2. MVP 范围（做 / 不做）

### 做

| 模块 | 说明 |
|------|------|
| 登录 | **邮箱 + 密码**（OIDC / GitHub OAuth 二期） |
| 空间（Workspace/Tenant） | 一用户可多空间；当前空间上下文 |
| RBAC | `viewer` / `contributor` / `admin` / `owner` |
| Coding 任务 | 指定仓库/目录说明 + prompt → 创建 Job |
| Worker | 每 Job 一个 Docker 容器，内跑 `codex exec` 或 `app-server` 短会话 |
| 事件流 | WebSocket/SSE 推送 turn/item（对齐 Codex JSONL / app-server 事件） |
| 模型 | 平台级上游（如胜算云）；空间可覆盖 model id |
| 配额 | 每空间：任务数 / token 粗计量 |
| 审计 | 谁在何时创建/取消了任务、角色变更 |

### 明确不做（MVP）

- 知识库 / RAG / Wiki / 图谱  
- 嵌入站点 widget、IM 通道  
- Penguin「一句话生成应用 / 自进化」  
- E2B / Cube 沙箱集群（仅 Docker）  
- 共享空间（Organization）跨租户共享  
- 计费收银台（只留配额计数）

## 3. 角色矩阵（抄 WeKnora，收窄到 Coding）

| 能力 | viewer | contributor | admin | owner |
|------|:------:|:-----------:|:-----:|:-----:|
| 看任务 / 事件流 / 产物 | ✓ | ✓ | ✓ | ✓ |
| 创建 / 取消自己的任务 | | ✓ | ✓ | ✓ |
| 取消他人任务 | | | ✓ | ✓ |
| 管理空间模型与配额配置 | | | ✓ | ✓ |
| 邀请成员、改角色 | | | ✓ | ✓ |
| 删除空间 | | | | ✓ |

资源归属：`jobs.creator_id` —— Contributor 默认可管自己的 Job；Admin+ 可管全部。

## 4. 信息架构（页面）

```text
/login
/workspaces                         空间列表
/w/:slug                            空间首页（任务概览）
/w/:slug/jobs                       任务列表
/w/:slug/jobs/new                   新建任务
/w/:slug/jobs/:id                   任务详情（事件流 + 日志 + 产物）
/w/:slug/settings/members           成员与角色
/w/:slug/settings/models            模型与配额（Admin+）
/w/:slug/settings/audit             审计日志（Admin+）
/account                            账号
```

可选后续：`/w/:slug/knowledge/*`（WeKnora 知识面）、`/factory/*`（Penguin 工场）。

## 5. 逻辑架构

```text
Browser
  │  HTTPS + SSE/WebSocket
  ▼
API Gateway / BFF
  │  AuthN + Workspace RBAC
  ▼
Control Plane (新仓库，如 codex-platform)
  ├─ tenants / members / rbac
  ├─ jobs / job_events
  ├─ quotas / audit
  └─ model_config
        │  enqueue
        ▼
   Job Queue (Redis/NATS/Postgres SKIP LOCKED)
        │
        ▼
   Worker
        │  docker run --network=... \
        │    -e SHENGSUANYUN_API_KEY=... \
        │    -v workspace_data:/work
        ▼
   Container: Codex (exec --json 或 app-server)
        │  stdout JSONL / RPC events
        ▼
   Event Ingest → job_events → 推给浏览器
        │
        ▼
   Model Gateway（胜算云等）
```

## 6. 核心数据模型（最小表）

| 表 | 关键字段 |
|----|----------|
| `users` | id, email, github_id?, ... |
| `workspaces` | id, slug, name, owner 约束 |
| `workspace_members` | workspace_id, user_id, role |
| `jobs` | id, workspace_id, creator_id, status, prompt, repo_url?, image, started_at, finished_at, error |
| `job_events` | id, job_id, seq, type, payload_json, created_at |
| `job_artifacts` | id, job_id, kind (log/patch/file), uri |
| `workspace_model_config` | workspace_id, provider, model, base_url_ref |
| `quota_counters` | workspace_id, period, jobs_used, tokens_used |
| `audit_logs` | id, workspace_id, actor_id, action, target, meta, created_at |

状态机建议：`queued → starting → running → succeeded|failed|cancelled`。

## 7. API 清单（REST 草图）

前缀：`/api/v1`。除登录外均需 Bearer；写操作带 workspace 上下文（路径或头 `X-Workspace-Id`）。

### Auth
- `POST /auth/register` `{ email, password }`
- `POST /auth/login` `{ email, password }`
- `GET /auth/me`
- ~~`POST /auth/oauth/github/callback`~~（二期）

### Workspaces
- `GET /workspaces`
- `POST /workspaces`
- `GET /workspaces/:id`
- `PATCH /workspaces/:id`（admin+）
- `DELETE /workspaces/:id`（owner）

### Members
- `GET /workspaces/:id/members`
- `POST /workspaces/:id/members` `{ user_email|user_id, role }`（admin+）
- `PATCH /workspaces/:id/members/:userId` `{ role }`（admin+；不可移除最后一位 owner）
- `DELETE /workspaces/:id/members/:userId`

### Jobs
- `GET /workspaces/:id/jobs?status=&mine=`
- `POST /workspaces/:id/jobs`  
  body: `{ prompt, repo_url?, branch?, model?, sandbox: "docker" }`
- `GET /workspaces/:id/jobs/:jobId`
- `POST /workspaces/:id/jobs/:jobId/cancel`
- `GET /workspaces/:id/jobs/:jobId/events?after_seq=`
- `GET /workspaces/:id/jobs/:jobId/artifacts`

### Realtime
- `GET /workspaces/:id/jobs/:jobId/stream`（SSE）  
  或 `WS /workspaces/:id/jobs/:jobId/ws`

### Models & Quota（admin+）
- `GET /workspaces/:id/model-config`
- `PUT /workspaces/:id/model-config`
- `GET /workspaces/:id/quota`

### Audit（admin+）
- `GET /workspaces/:id/audit?from=&to=&actor=`

错误码约定：`401` 未登录 · `403` RBAC · `404` · `409` 配额用尽 / 最后 owner · `422` 校验。

## 8. Worker ↔ Codex 约定

MVP 推荐路径：**`codex exec --json`**（实现简单、好 ingest）。

```bash
# 容器内示意
source /secrets/load-provider.sh
cd /work/repo
codex exec --json --skip-git-repo-check "$PROMPT"
```

- stdout：JSONL → 解析为 `job_events`  
- stderr：采集为日志 artifact  
- 退出码：映射 job 终态  
- 配置：挂载只读 `config.toml` 模板；密钥仅环境变量注入，**不进镜像层、不进前端**

二期再上长连接 `app-server`（IDE 级交互、中途审批）。

## 9. 仓库拆分

| 仓库 | 用途 |
|------|------|
| `nanjingwuyanzu/codex` | 底座学习 / 必要 patch（少改） |
| `nanjingwuyanzu/WeKnora` | 对照 RBAC/沙箱文档（只读参考） |
| `nanjingwuyanzu/penguin-harness` | 对照工场体验（二期） |
| **`nanjingwuyanzu/codex-platform`（待建）** | Web + API + Worker —— **真正交付网址的仓库** |

## 10. 里程碑（设计已定后的实现顺序）

1. 空仓库 + 登录 + 空间 + 成员 RBAC  
2. Job 表 + 假 Worker（不跑 Codex，只推模拟事件）—— 打通 UI  
3. 真 Docker Worker + `codex exec --json`  
4. 配额 + 审计  
5. （可选）GitHub 仓库绑定、产物 diff 展示  
6. （更后）知识面 / Penguin 工场 / OIDC / E2B  

## 11. 开放问题（开工前可再定）

1. ~~登录优先~~ → **已定：邮箱 + 密码**  
2. Job 输入：MVP 允许任意 `repo_url`（可后加「仅模板仓库」）  
3. 审批：MVP **`approval_policy = never` + workspace-write 沙箱**；网页只做观看与取消  

已拍板默认：邮箱登录；`repo_url`；自动审批 + 强沙箱。
