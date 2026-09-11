# Codex 套壳与平台化调研笔记

> 面向二次开发学习：为什么社区拿 Codex CLI 做 agent / 编排 / 多租户平台，以及同类项目怎么分层。  
> 整理日期：2026-09-11 · 对照上游：[openai/codex](https://github.com/openai/codex) · 本 fork：[nanjingwuyanzu/codex](https://github.com/nanjingwuyanzu/codex)

## 先分清两条路线

网上说「拿 Codex 套壳做平台」，其实常把两件事混在一起：

| 路线 | 套在哪 | 解决什么 | 典型形态 |
|------|--------|----------|----------|
| **A. 模型网关** | Codex ↔ 上游模型之间 | 多账号、多租户 key、路由、限流、计费、协议转换 | OpenAI 兼容 HTTP 代理 |
| **B. Agent 产品壳** | Codex 进程之外 | 会话 UI、IM、队列、多 agent 编排、审批看板 | 调 `app-server` / `codex exec` |

成熟「AI 平台」通常是 **A + B**：网关管模型与租户配额，外壳管会话与编排；Codex 负责真正的 coding agent loop。

## 社区在做什么（抽样）

### A. 模型入口 / 多租户网关

| 项目 | 大致定位 |
|------|----------|
| [kittors/CliRelay](https://github.com/kittors/CliRelay) | 自建网关：把多种 Coding CLI / 订阅收成统一 OpenAI/Claude/Gemini/Codex 兼容端点；Web 控制台、日志、配额（星数较高，偏「平台感」） |
| [christiandoxa/prodex](https://github.com/christiandoxa/prodex) | 多账号 Codex wrapper + `prodex gateway`：轮询、配额、虚拟 key、仪表盘 |
| [NodeNestor/CodeGate](https://github.com/NodeNestor/CodeGate) | Drop-in gateway：多租户 API key、路由、failover、PII guardrails |
| [ZiryaNoov/codex-proxy](https://github.com/ZiryaNoov/codex-proxy) | 轻量网关：多厂商协议转换（含 Responses ↔ Chat）、JWT/多用户预算 |
| [Portkey ↔ Codex](https://docs.portkey.ai/docs/virtual_key_old/integrations/libraries/codex) | 商业企业网关：预算、RBAC、观测；通过 Codex `model_providers` 接入 |

接入方式通常是改 Codex 的 `~/.codex/config.toml`：

```toml
model_provider = "my_gateway"
model = "some/model-id"

[model_providers.my_gateway]
name = "My Gateway"
base_url = "https://gateway.example.com/v1"
env_key = "MY_GATEWAY_API_KEY"
wire_api = "responses"   # 注意：Codex 已移除 wire_api = "chat"
```

### B. Agent 进程壳 / 编排

| 项目 | 大致定位 |
|------|----------|
| [Grail-Computer/FastClaw](https://github.com/Grail-Computer/FastClaw) | Slack/Telegram「员工」：Rust 服务 + Codex CLI + 队列 / 记忆 / cron / 管理台 |
| [nessielabs/boring-orchestrator](https://github.com/nessielabs/boring-orchestrator) | 调度与监控 Claude / Codex CLI 的本地看板 |
| [agentrq/codex-gateway](https://github.com/agentrq/codex-gateway) | 把 Codex CLI 接到远程任务管理（AgentRQ） |
| [swaylq/hermit-agent](https://github.com/swaylq/hermit-agent) | 一键起 Telegram 连接的 Coding Agent（Claude/Codex） |
| [YuanpingSong/embassy](https://github.com/YuanpingSong/embassy) | 多 CLI agent 互相发消息（本机或 SSH） |
| [dingpeng328/feishu-claude-bridge](https://github.com/dingpeng328/feishu-claude-bridge) | 飞书话题桥接本地 Claude / Codex |
| [codex-yolo/codex-yolo](https://github.com/codex-yolo/codex-yolo) | tmux 并行多 Codex + 自动批准权限类提示 |

模式共性：外层管 **触发 / 队列 / 通道 / 多会话**；内层仍跑官方（或 fork）Codex 二进制。

## 为什么 Codex 适合当底座

OpenAI 官方说明：CLI、IDE、Desktop、Web **共用同一套 agent harness**；对外产品接口是 **App Server**（JSON-RPC）。也就是说「可套壳」是一等设计，不是民间硬拗。参考：[Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/)。

### 1. 完整 Coding Agent，不是聊天框

`codex-rs/core` 里已有：读改代码、跑 shell、apply patch、工具调用、MCP 等。套壳不必从零写 agent loop。

### 2. `codex app-server`：产品壳正规接口

- 位置：`codex-rs/app-server`（协议见 `codex-rs/app-server-protocol`）
- 传输：stdio（默认）、unix socket、实验性 websocket
- 能力：`initialize` / `thread/*` / `turn/*`、审批、diff、历史等双向 JSON-RPC
- VS Code 扩展、Desktop 都走这条路

做 Web / IDE / 桌面壳，应优先接 **app-server**，不要去刮 TUI。

### 3. `codex exec`：编排原语

非交互模式，适合 CI、队列 worker、流水线一步：

- 进度走 stderr，最终答复走 stdout
- `--json`：JSONL 事件流（`thread.started` / `turn.*` / `item.*`）
- `--output-schema`：结构化最终输出
- `codex exec resume`：续跑会话
- 显式 sandbox / approval，便于自动化权限最小化

文档：[Non-interactive mode](https://developers.openai.com/codex/noninteractive)

### 4. 可插拔模型层

`[model_providers.*]` 支持自定义 `base_url` + `env_key`；内置 openai / ollama / lmstudio / amazon-bedrock 等。平台可把「租户 → 上游」放在网关，Codex 只认一个 provider。

**注意：** `wire_api = "chat"` 已删除，自定义 provider 需 `wire_api = "responses"`（或网关做协议转换）。

### 5. 安全策略是一等公民

- Sandbox：`read-only` / `workspace-write` / `danger-full-access`
- Approval policy、execpolicy、hooks、`requirements.toml`（托管约束）
- 多租户隔离仍要平台自己做（容器 / 独立 `CODEX_HOME` / 密钥隔离），但「工具能否写盘联网」有现成旋钮

### 6. 扩展面

MCP、skills、slash commands、hooks、`AGENTS.md`；仓库内还有 TypeScript / Python SDK（`sdk/`）。很多能力用配置加，不必 fork 核心。

### 7. 许可证

Apache-2.0，二次开发与商用套壳友好。上游**不接受外部代码 PR**（社区贡献以 issue / 分析为主），平台能力需 fork 或外围包。

## 推荐对照架构

```text
┌─────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│ Web/IM/IDE  │────▶│ 编排层（租户/队列/计费）│────▶│ app-server/exec │
└─────────────┘     └──────────────────────┘     └────────┬────────┘
                                                          │
                                                          ▼
                                                 ┌─────────────────┐
                                                 │  codex-rs core  │
                                                 └────────┬────────┘
                                                          │
                                                          ▼
                                                 ┌─────────────────┐
                                                 │ 模型网关（多租户）│──▶ 上游 LLM
                                                 └─────────────────┘
```

## 和「类似项目」怎么比

对比时建议固定几列，避免只比星数：

1. **套的是模型还是 agent？**（A / B / A+B）
2. **Agent loop 自研还是复用 Codex/Claude Code？**
3. **产品接口**：刮 CLI、调 `exec`、还是接 `app-server`？
4. **多租户落点**：API key 虚拟化、容器隔离、还是仅本机多 profile？
5. **协议**：是否要求 `/v1/responses`、能否转 Chat Completions
6. **安全模型**：sandbox / 审批 / 是否把订阅 token 暴露给租户代码
7. **扩展**：MCP / skills / hooks 是否可配

## 本 fork 二次开发建议阅读路径

1. `codex-rs/app-server` + `app-server-protocol` — 套壳 RPC  
2. 官方 noninteractive 文档 + `codex exec --help` — 编排入口  
3. `codex-rs/model-provider` / `model-provider-info` — 自定义上游  
4. `codex-rs/core` — turn / tools / 会话状态（先浏览，再深入）  
5. `codex-rs/tui` — 仅作交互参考，平台壳不必复制  
6. `docs/` + [Config advanced](https://developers.openai.com/codex/config-advanced) — provider / sandbox / hooks  

## 本机已验证过的配置（仅笔记，密钥勿入库）

本 fork 在学习机上已 `cargo build --bin codex` 通过；自定义 provider 示例：

- Provider id：`shengsuanyun`
- Model：`deepseek/deepseek-v4-pro`
- `wire_api = "responses"`
- 密钥放环境变量（如 `SHENGSUANYUN_API_KEY`），**不要**写进本仓库

## 局限（别神话）

- 多租户不是开箱功能：默认是本机单用户 agent  
- App Server WebSocket 仍标 experimental；公网暴露需自建鉴权  
- 上游不收外部 PR：长期维护靠自己的 fork  
- Debug 二进制很大、编译重；平台交付需考虑 release 构建与按租户资源配额  

---

后续可在本目录继续追加：`notes/architecture-walkthrough.md`（模块地图）、`notes/compare-matrix.md`（与 CliRelay / Prodex 等逐项对比表）。
