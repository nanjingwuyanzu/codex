# Codex 核心代码地图（从哪读起）

> 学习用索引。仓库根：`codex-rs/`。本 fork：`nanjingwuyanzu/codex`。

## 30 秒结论

真正干活的不是 TUI，而是 **`codex-core`**：一轮用户输入会走进

`SessionTask(RegularTask)` → `session::turn::run_turn` → 模型采样 → `ToolCallRuntime`（并行调度）→ `ToolRouter` / `ToolOrchestrator`（审批 + 沙箱）→ 结果回灌再采样。

TUI / IDE / Desktop / `codex exec` 只是不同「壳」，共用这套 harness。

## 必读优先级

| 优先级 | 路径 | 为什么 |
|--------|------|--------|
| P0 | `core/src/session/turn.rs` | **编排心脏**：`run_turn`、采样、工具回合闭环 |
| P0 | `core/src/tasks/regular.rs` | 普通会话任务如何启动 `run_turn` |
| P0 | `core/src/tools/parallel.rs` | 工具并行：`ToolCallRuntime`、读写锁门控 |
| P0 | `core/src/tools/orchestrator.rs` | 单工具：审批 → 选沙箱 → 执行 → 升级重试 |
| P0 | `core/src/tools/router.rs` | `ToolCall` 路由到具体 handler |
| P1 | `core/src/tools/registry.rs` | 工具注册 / Pre-Post hooks / 执行封装 |
| P1 | `core/src/thread_manager.rs` | 多线程（thread）生命周期、fork、历史 |
| P1 | `core/src/session/session.rs` | Session 服务总线（事件、MCP、client…） |
| P1 | `app-server/` + `app-server-protocol/` | 套壳 RPC：`thread/start`、`turn/start` |
| P2 | `core/src/client.rs` | Responses API 客户端会话 |
| P2 | `core/src/tools/handlers/` | shell / apply_patch / MCP / multi_agents… |
| P2 | `core/src/agent/` | agent roles / control（多 agent） |
| P2 | `core/src/guardian/` | Guardian 审核相关 |
| P3 | `tui/` | 交互展示层，平台壳不必抄 |
| P3 | `cli/` | 二进制入口、`exec` / `app-server` 子命令 |

## 目录职责速查（`core/src`）

| 目录/文件 | 职责 |
|-----------|------|
| `session/` | Turn 上下文、输入队列、MCP 预热、多 agent 会话设置、`turn.rs` |
| `tasks/` | SessionTask：`regular` / `review` / `compact` / `user_shell` |
| `tools/` | 工具编排全套：router、registry、orchestrator、parallel、handlers、sandboxing |
| `tools/handlers/` | 具体工具实现（`apply_patch`、`mcp`、`multi_agents*`、`plan`…） |
| `thread_manager.rs` / `codex_thread.rs` | Thread 抽象与管理 |
| `agent/` | Agent 角色解析与控制面 |
| `mcp*.rs` / `mcp_tool_call*` | MCP 连接与调用 |
| `exec*.rs` / `sandboxing/` / `safety.rs` | 命令执行与沙箱策略 |
| `compact*.rs` | 上下文压缩（本地/远程） |
| `hook_runtime.rs` | lifecycle hooks |
| `skills.rs` / `plugins/` | Skills 与插件 |
| `guardian/` | 自动审核 / Guardian 流程 |

## 入口对照

| 你从哪启动 | 大致落到 |
|------------|----------|
| `./target/debug/codex`（TUI） | `tui` → core Session / turn |
| `codex exec "..."` | CLI exec → 同一 `run_turn` harness，事件可 JSONL |
| `codex app-server` | JSON-RPC → `ThreadManager` / `submit(Op::…)` → core |
| IDE / Desktop | 官方也是 app-server 客户端 |

## 建议阅读顺序（两晚能摸完）

1. 读本文件 + `notes/architecture-overview.md`  
2. 打开 `tasks/regular.rs` 看如何调用 `run_turn`  
3. 在 `session/turn.rs` 搜 `run_turn` / `run_sampling_request` / `ToolCallRuntime`  
4. 读 `tools/parallel.rs` 的 `supports_parallel` 分支（读锁并行 vs 写锁串行）  
5. 读 `tools/orchestrator.rs` 文件头注释（审批→沙箱→重试）  
6. 挑一个 handler：`tools/handlers/apply_patch.rs` 或 `mcp.rs`  
7. 扫 `app-server/src/message_processor.rs` 里 `thread/start` | `turn/start`

更细的回合时序见 `notes/turn-loop-deep-dive.md`；「为什么编排牛」见 `notes/why-orchestration-matters.md`。
