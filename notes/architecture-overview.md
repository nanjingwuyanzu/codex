# Codex 架构总览：壳、Harness、模型

## 一张图

```text
┌──────────────────────────────────────────────────────────┐
│  壳（可替换）                                              │
│  TUI · VS Code/IDE · Desktop · Web · 你的平台 UI          │
│  ───────────────── 或 ─────────────────                   │
│  codex exec（脚本/CI/队列 worker）                         │
└───────────────────────────┬──────────────────────────────┘
                            │ JSON-RPC / 进程内 API / CLI
                            ▼
┌──────────────────────────────────────────────────────────┐
│  App Server / CLI 适配层                                   │
│  thread/*  turn/*  审批回传  事件流                         │
└───────────────────────────┬──────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────┐
│  codex-core harness（真正编排）                            │
│  ThreadManager → Session → run_turn                        │
│       │              │                                     │
│       │              ├─ compact / hooks / skills / MCP     │
│       │              ├─ model sampling (Responses API)     │
│       │              └─ ToolCallRuntime → handlers         │
│       │                       │                            │
│       │                       └─ ToolOrchestrator          │
│       │                            审批 + 沙箱 + 重试       │
└───────────────────────────┬──────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────┐
│  model_providers（可插拔）                                 │
│  openai / 自定义网关 / ollama / bedrock …                  │
│  wire_api = "responses"                                   │
└──────────────────────────────────────────────────────────┘
```

## 三层分别解决什么

1. **壳**：给人看、给平台接。不负责「何时调工具、如何沙箱」。  
2. **Harness（core）**：agent loop。OpenAI 自己的 IDE/桌面也复用它（见官方 *Unlocking the Codex harness*）。  
3. **Provider**：只负责模型协议与鉴权。多租户 key 通常放在这一层外的网关。

## 关键概念：Thread / Turn / Step / ToolCall

| 概念 | 含义 |
|------|------|
| **Thread** | 一条会话线（可 fork、存历史、挂附件） |
| **Turn** | 用户一次提交触发的完整回合（可能多次采样 + 多次工具） |
| **Step / StepContext** | 某次采样用到的「模型可见工具表 + 配置快照」 |
| **ToolCall** | 模型发出的一次工具调用（带 `call_id` + payload） |

平台套壳时：UI 管理 Thread；每次用户消息对应 `turn/start`（或 `exec`）；中间的 tool/approval 事件要回传给 UI。

## 和「聊天 Bot」的本质差别

普通 Bot：`用户 → LLM → 文本`。

Codex：`用户 → LLM →（工具｜审批｜沙箱）→ 观察 → LLM → … → 最终消息`，并且：

- 工具可 **并行**（见 `tools/parallel.rs`）
- 危险操作走 **策略化审批**
- 执行落在 **可升级沙箱**
- 全程有 **可订阅事件流**（供 IDE/平台渲染 diff、命令、计划）

这就是它适合做 agent / 编排底座的原因；展开论述见 `why-orchestration-matters.md`。

## 本仓库模块对应

| 能力 | 主要 crate / 路径 |
|------|-------------------|
| Harness | `codex-rs/core` |
| 协议事件 | `codex-rs/protocol` |
| App Server | `codex-rs/app-server` (+ `app-server-protocol`) |
| 模型提供方 | `codex-rs/model-provider*` |
| 沙箱 | `codex-rs/sandboxing`、`linux-sandbox` 等 |
| TUI | `codex-rs/tui` |
| 对外 SDK | `sdk/typescript`、`sdk/python` |

## 二次开发选型建议

| 你想做 | 接哪里 |
|--------|--------|
| Web/桌面产品壳 | `app-server` JSON-RPC |
| CI / 工作流一步 | `codex exec --json` |
| 换模型 / 多租户路由 | `model_providers` + 外置网关 |
| 加能力但不改 loop | MCP / skills / hooks |
| 改 agent 行为本身 | `core` 的 turn / tools（维护成本最高） |
