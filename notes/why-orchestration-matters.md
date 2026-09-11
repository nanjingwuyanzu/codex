# 为什么说 Codex 的编排「很牛」

不是营销形容词，而是相对「自己用 LLM API 拼个 agent」时，它已经把最难的几层做厚了。下面每一点都尽量挂到源码/机制。

## 1. 官方就按「可套壳 harness」造

OpenAI 的 CLI / IDE / Desktop / Web 共用同一 agent loop，对外暴露 **App Server（JSON-RPC）**。  
你做平台时，是在接一条 **被 IDE 验证过的编排总线**，不是逆工程 TUI。

证据：官方文章 *Unlocking the Codex harness*；代码 `app-server/`、`app-server-protocol/`。

## 2. Turn 是完整控制论闭环，不是单次 completion

`run_turn`（`core/src/session/turn.rs`）内含：

- 采样前 compact  
- hooks / skills / MCP 就绪  
- 采样（允许 parallel tool calls）  
- 工具执行与结果回灌  
- 再采样直到回合结束  

编排对象是 **闭环**，所以适合当工作流引擎的「智能一步」。

## 3. 工具并行有策略，不是 naive 串行

`ToolCallRuntime`（`tools/parallel.rs`）：

- 工具声明是否 `supports_parallel`
- 并行工具共享 **读锁**；非并行工具抢 **写锁**

效果：能并行的（只读搜索、部分查询）一起跑；危险/互斥的（写盘、某些 shell）自动串行。  
这是工业级 agent runtime 的味道。

## 4. 审批 × 沙箱 × 重试 做成统一编排器

`ToolOrchestrator`（`tools/orchestrator.rs`）把三件事焊在一条管道：

`approval → select sandbox → attempt → escalate sandbox & retry`

还带网络策略审批、审批缓存（避免重复弹窗）。  
平台多租户时，你仍要做租户隔离，但 **单租户内的工具安全语义已经齐**。

## 5. 事件流是一等公民（UI/编排可观察）

壳不是轮询「聊到哪了」，而是订阅：

- turn started/completed/failed  
- item：消息、reasoning、命令、文件变更、MCP、plan…  
- 审批 / elicitation 请求  

`codex exec --json` 把同一思想导出给脚本。  
**可观察的 agent** 才能嵌进平台看板与自动化。

## 6. 多 Agent 长在工具层，而不是外挂脚本

`tools/handlers/multi_agents*`、`agent/`、`thread_manager`：

- 可 spawn / 发消息 / followup  
- Thread 可 fork、存历史  
- Guardian 等审核角色可介入  

也就是说「编排多个 coding worker」是 **产品内核能力**，不是 tmux 开俩窗口那么糙（社区也有 tmux 方案，但那是外层）。

## 7. 扩展点让你少改 core

| 扩展 | 用途 |
|------|------|
| MCP | 接外部系统工具 |
| Skills | 可复用流程知识 |
| Hooks | Pre/Post tool、策略拦截 |
| model_providers | 换模型网关（多租户常接这里） |
| execpolicy / permissions | 命令与权限策略 |

平台多数需求应先走扩展点；fork core 是最后手段（上游还不收外部 PR）。

## 8. 同一 harness，多种部署形态

| 形态 | 适合 |
|------|------|
| TUI | 人在终端里用 |
| App Server | 做 IDE/Web/桌面壳 |
| `codex exec` | CI、队列、cron、autofix |
| SDK | 用 TypeScript/Python 嵌进服务 |

「编排牛」很大程度是因为：**一套语义，多种投影**。

## 9. 和常见「套壳项目」怎么对照理解

结合 `notes/platform-survey.md`：

- **网关型（CliRelay / Prodex…）** 强化的是模型层多租户 —— 补的是 Codex 有意留给外部的 provider 边界。  
- **Agent 壳型（FastClaw / orchestrator…）** 强化的是触发与通道 —— 吃的是 exec/app-server 的编排红利。  
- 你要做「AI 平台」，通常 **网关 + harness 壳** 都要，但 **别重写 run_turn**。

## 10. 它并不神秘的边界（保持清醒）

- 多租户隔离、计费、公网鉴权：要你自己做  
- App Server WebSocket 仍偏实验：生产更稳妥用 stdio/unix + 你的网关  
- `wire_api` 只剩 `responses`：网关要兼容  
- `core` 体量巨大：二次开发要克制，优先扩展点  

## 一句话

Codex 牛的不是「会聊天」，而是把 **采样、并行工具、策略化执行、可观察事件、多线程/多 agent、可插拔模型** 收成一个被官方多端复用的 harness——所以社区才会在外面加壳做平台，而不是从零写 agent loop。
