# 对照笔记：PenguinHarness × WeKnora × Codex（vs Dify）

> 参考仓库（已 fork）：
> - https://github.com/nanjingwuyanzu/penguin-harness ← Prism-Shadow/penguin-harness  
> - https://github.com/nanjingwuyanzu/WeKnora ← Tencent/WeKnora  
> - https://github.com/nanjingwuyanzu/codex ← 你的 coding agent 底座  

目标产品：多租户 **Web 平台**，底座 Codex harness，交付网址。

## 一句话定位

| 项目 | 它到底是什么 | 星数量级 | 和你目标的贴合 |
|------|--------------|----------|----------------|
| **Codex** | 工业级 **Coding Agent harness**（turn/tool/sandbox/app-server） | 上游十万级 | **执行底座**（写代码、改仓库、跑命令） |
| **PenguinHarness** | **用 Agent 造 Agent 应用** 的本地/服务器 harness（TS monorepo：core/cli/server/web/desktop） | ~2k | **产品形态与「一句话生成应用」体验**；偏开发者工具与自进化 |
| **WeKnora** | **知识平台**：RAG + Agent + Wiki + **原生多租户/RBAC/沙箱集群** | ~22k | **多租户 Web、权限、知识、沙箱运维** 最可抄 |
| **Dify** | 通用 LLM 应用编排（工作流/Chatflow） | 很大 | 通用能力强，但对「coding agent 平台」往往过重且不够「写代码原生」 |

你觉得 Dify 不如这两家好用，在「做 coding / agent 平台」语境下合理：Dify 强在通用编排；Penguin 强在 agent 开发闭环；WeKnora 强在知识+多租户 Web 产品完整度。

## PenguinHarness 可借鉴什么

架构信号（`packages/`）：`core` · `cli` · `server` · `web` · `desktop` · `docs` · `landing` —— 已经是「有 Web」的平台形，不只是 CLI。

值得抄的思路：

1. **精简工具集 + 低层接口**：少 tool call、少 token；对 DeepSeek 等开放模型友好（和你胜算云路线一致）。
2. **Skills / hooks 驱动**：办公、软件开发、AI 应用开发插件；agent 可自评自进化（benchmark → 改 skill → 下一版）。
3. **Trace 可观测**：每个请求可回放 —— 多租户平台必备。
4. **「一句话生成可运行 agent 应用」**：这是差异化产品叙事，可做你平台的「应用工场」层。
5. **协议开放**：任意 OpenAI 兼容端点。

要注意：

- 默认叙事偏 **本地优先 / 桌面安装**；你要交付 **网址**，应主要吃它的 `server`+`web` 思路，而不是桌面发行版。
- 它是 **另一套 agent runtime（TS）**，不是 Codex。不要幻想「合并成一个 monorepo 就完事」——应 **借鉴产品层**，执行层仍用 Codex（或双 runtime 可选）。
- 它宣传里拿 Codex 当编程对标，说明社区已把 Codex 当 coding 上限参考。

## WeKnora 可借鉴什么

这是腾讯开源的 **LLM 知识平台**：文档 → RAG → 推理 Agent → 自维护 Wiki；topics 里直接有 `multi-tenant`。

值得抄的（对你「多租户网址」几乎是满分参考）：

1. **Workspace RBAC**：`viewer / contributor / admin / owner` + 资源 `creator_id`（见上游 `docs/RBAC说明.md`）。
2. **共享空间（Organization）与空间 RBAC 正交**：跨租户共享知识而不搞乱归属。
3. **Web UI 完整度**：知识库、Wiki 图、会话、审计、Langfuse 追踪、嵌入站点 widget。
4. **沙箱集群**：Docker / CubeSandbox / E2B；标准 sandbox 镜像；Skills 在沙箱里跑（见 `docs/sandbox-*.md`）。
5. **Agent 工具设计克制**：读写改执行原语精简、并行有屏障（读并行、写串行）——和 Codex `ToolCallRuntime` 哲学相近，可互证。
6. **运维面**：worker pool、任务队列、平台级 API Key、OIDC、IM 通道等。

要注意：

- 许可证显示为 **Other（非标准 OSI 一句话）**：商用前 **必须读 LICENSE / THIRD_PARTY**，别假设 Apache。
- 强项是 **知识+问答+Wiki**；**硬核改大型代码仓库**不如 Codex。它该当你的「知识与租户壳」，不是替代 coding harness。
- Agent 工具偏知识/沙箱文档任务；接 Codex 应用「写生产代码」更强。

## 三者怎么「结合」（推荐组合，而不是硬合并代码）

```text
                    ┌─────────────────────────────┐
   浏览器用户 ──────▶│  Web 控制面（借鉴 WeKnora）   │
                    │  租户 / RBAC / 计费 / 审计    │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
     ┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐
     │ 知识面 WeKnora  │  │ 应用工场 Penguin │  │ Coding 执行面     │
     │ 思路：RAG/Wiki │  │ 思路：生成/评测  │  │ Codex app-server │
     │ /检索给 Agent  │  │ /Skill 自进化    │  │ / exec + 沙箱    │
     └────────────────┘  └─────────────────┘  └──────────────────┘
                                   │
                                   ▼
                        模型网关（胜算云等）
```

### 结合原则

| 层 | 采用谁的思路 | 实现建议 |
|----|--------------|----------|
| 多租户 + Web 产品 | **WeKnora** | 新平台仓库自建；对照其 RBAC/空间/审计，不整库 fork 进 Codex |
| Agent 应用生成 / 自进化体验 | **Penguin** | 产品功能参考；runtime 可后置，先做「模板+评测」也行 |
| 真正改代码 / 跑仓库任务 | **Codex** | `app-server` 或 `codex exec` 作为 worker；每任务独立容器 + `CODEX_HOME` |
| 模型与配额 | 自建网关 | 对齐 Penguin「任意 OpenAI 兼容」+ WeKnora「平台/空间 API Key」 |

### 不建议的做法

- 把 WeKnora、Penguin、Codex **揉进一个巨型 monorepo** 一次重写。  
- 用 Dify 工作流硬模拟 coding agent（能跑 demo，难做成「仓库级 agent 平台」）。  
- 放弃 Codex 只用 Penguin core 写代码——除非你愿意长期维护第二套 coding harness。

## 相对 Dify：你为何会觉得「这两家更好用」

| 维度 | Dify | Penguin | WeKnora |
|------|------|---------|---------|
| 心智模型 | 通用工作流画布 | Agent 开发与自进化 | 知识+Agent 产品 |
| Coding 原生 | 弱（偏工具节点） | 中（可调 Claude Code 等） | 中（沙箱技能，非 Codex 级） |
| 多租户 Web | 有，但偏应用编排 SaaS | 弱于 WeKnora（更偏本机/单部署） | **强（RBAC/空间/嵌入）** |
| 和 Codex 互补 | 重叠在「编排 UI」 | 重叠在「harness 叙事」 | 互补在「租户与知识」 |

对你定的目标（Codex 底座 + 多租户网址）：  
**WeKnora = 壳与租户教科书；Penguin = 体验与 agent 工厂灵感；Codex = 引擎。Dify 可降级为「工作流可选插件」，不必当主干。**

## 建议的下一步（仍可不写业务代码）

1. 读 WeKnora：`docs/RBAC说明.md`、`共享空间说明.md`、`sandbox-cluster.md`  
2. 读 Penguin：`packages/server`、`packages/web`、官方 docs（penguin.ooo/docs）  
3. 画一版你自己的「最小平台」：只要 **登录 + 空间 + 发起一次 Codex 任务 + 看事件流**  
4. 单独开 `codex-platform` 仓库，三份 fork 只做对照，不往 Codex fork 里塞 Go/TS 整站  

许可证再确认一次 WeKnora 条款后再定商用路径。
