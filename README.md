# OpenClaw 分析文档集

本文档集聚焦公开仓库 `openclaw/openclaw` 的源码结构、运行时架构、扩展体系与安全边界，目标不是复述 README，而是给出一套便于继续深挖源码的中文分析底稿。

快照说明：
- 文档整理时间：2026-04-26
- 公开仓库：<https://github.com/openclaw/openclaw>
- 本次分析主要基于官方公开仓库 tree、`package.json`、`pnpm-workspace.yaml` 以及官方文档站 `docs.openclaw.ai`
- 若某一条结论来自架构推断而不是文档直接表述，文中会标记为“分析推断”

阅读顺序建议：
1. [00-总览与结论.md](/Users/seaya/huquan.95/project-4/open_claw/00-总览与结论.md)
2. [01-仓库结构与模块地图.md](/Users/seaya/huquan.95/project-4/open_claw/01-仓库结构与模块地图.md)
3. [02-启动流程与控制面.md](/Users/seaya/huquan.95/project-4/open_claw/02-启动流程与控制面.md)
4. [03-Agent-Loop-会话与队列.md](/Users/seaya/huquan.95/project-4/open_claw/03-Agent-Loop-会话与队列.md)
5. [04-渠道-插件与扩展体系.md](/Users/seaya/huquan.95/project-4/open_claw/04-渠道-插件与扩展体系.md)
6. [05-安全模型与隔离设计.md](/Users/seaya/huquan.95/project-4/open_claw/05-安全模型与隔离设计.md)
7. [06-源码阅读路线与关键文件索引.md](/Users/seaya/huquan.95/project-4/open_claw/06-源码阅读路线与关键文件索引.md)
8. [07-架构判断-复杂点与演进建议.md](/Users/seaya/huquan.95/project-4/open_claw/07-架构判断-复杂点与演进建议.md)
9. [10-Gateway-协议与API.md](/Users/seaya/huquan.95/project-4/open_claw/10-Gateway-协议与API.md)
10. [11-Workspace-与-Bootstrap.md](/Users/seaya/huquan.95/project-4/open_claw/11-Workspace-与-Bootstrap.md)
11. [12-Multi-Agent-路由与隔离.md](/Users/seaya/huquan.95/project-4/open_claw/12-Multi-Agent-路由与隔离.md)
12. [13-Tools-Sandbox-Exec审批.md](/Users/seaya/huquan.95/project-4/open_claw/13-Tools-Sandbox-Exec审批.md)
13. [14-Config-Secrets-配置激活.md](/Users/seaya/huquan.95/project-4/open_claw/14-Config-Secrets-配置激活.md)
14. [15-Cron-Hooks-自动化.md](/Users/seaya/huquan.95/project-4/open_claw/15-Cron-Hooks-自动化.md)
15. [16-Provider-Model-Auth-Usage.md](/Users/seaya/huquan.95/project-4/open_claw/16-Provider-Model-Auth-Usage.md)
16. [17-Channel-消息生命周期.md](/Users/seaya/huquan.95/project-4/open_claw/17-Channel-消息生命周期.md)
17. [18-Plugin-Runtime-实现细节.md](/Users/seaya/huquan.95/project-4/open_claw/18-Plugin-Runtime-实现细节.md)
18. [参考资料.md](/Users/seaya/huquan.95/project-4/open_claw/参考资料.md)

专题补充：
- [08-agent-loop-实现详解/README.md](/Users/seaya/huquan.95/project-4/open_claw/08-agent-loop-实现详解/README.md)
- [08-agent-loop-实现详解/01-主执行链路.md](/Users/seaya/huquan.95/project-4/open_claw/08-agent-loop-实现详解/01-主执行链路.md)
- [08-agent-loop-实现详解/02-状态-事件-队列.md](/Users/seaya/huquan.95/project-4/open_claw/08-agent-loop-实现详解/02-状态-事件-队列.md)
- [08-agent-loop-实现详解/03-模拟数据流实例.md](/Users/seaya/huquan.95/project-4/open_claw/08-agent-loop-实现详解/03-模拟数据流实例.md)
- [08-agent-loop-实现详解/04-源码路径与参考.md](/Users/seaya/huquan.95/project-4/open_claw/08-agent-loop-实现详解/04-源码路径与参考.md)
- [09-context-上下文管理/README.md](/Users/seaya/huquan.95/project-4/open_claw/09-context-上下文管理/README.md)
- [09-context-上下文管理/01-context-维护机制.md](/Users/seaya/huquan.95/project-4/open_claw/09-context-上下文管理/01-context-维护机制.md)
- [09-context-上下文管理/02-system-prompt-设计.md](/Users/seaya/huquan.95/project-4/open_claw/09-context-上下文管理/02-system-prompt-设计.md)
- [19-system-prompt-实现详解/README.md](/Users/seaya/huquan.95/project-4/open_claw/19-system-prompt-实现详解/README.md)
- [20-memory-记忆系统/README.md](/Users/seaya/huquan.95/project-4/open_claw/20-memory-记忆系统/README.md) —— 记忆分层、加载 vs 检索、compaction flush、dreaming 固化、理论依据

最短摘要：
- OpenClaw 不是单一 CLI，而是一个把 `gateway`、`agent loop`、`session/group state`、`channel/provider/plugin`、`control UI` 组合到一起的 agent 平台。
- 它的核心思路不是“每次消息直接调模型”，而是“所有状态修改走命令队列，所有端接入统一走 gateway，所有 agent 运行附着在 session/group 抽象之上”。
- 这让它非常适合做多入口、多会话、多角色、多插件的 agent 系统，但代价是状态机复杂、边界多、阅读门槛高。
- 后续补充的 Gateway Protocol、Workspace/Bootstrap、Multi-agent Routing 三篇，用来补齐“控制面 API、agent home、跨 agent 隔离”这三个平台运行面。
- Tools/Sandbox/Exec 审批、Config/Secrets、Cron/Hooks 三篇，用来补齐“运行时治理”：能力暴露、配置激活、后台自动化。
- Provider/Model/Auth/Usage、Channel 消息生命周期、Plugin Runtime 三篇，用来补齐“外部能力接入”：模型供应商、消息渠道、插件加载与能力注册。

如果只想快速进入源码，先看：
- `openclaw.mjs`
- `package.json`
- `pnpm-workspace.yaml`
- `src/gateway/*`
- `src/gateway/protocol/*`
- `src/gateway/server-methods/*`
- `src/agents/system-prompt.ts`
- `src/agents/skills.ts`
- `src/agents/pi-tools.ts`
- `src/agents/sandbox.ts`
- `src/agents/model-selection.ts`
- `src/agents/model-auth.ts`
- `src/channels/*`
- `src/plugins/*`
- `src/plugin-sdk/*`
- `docs/architecture.md`
- `docs/gateway-architecture.md`
- Gateway Protocol 文档：<https://docs.openclaw.ai/gateway/protocol>
- Tool configuration：<https://docs.openclaw.ai/gateway/config-tools>
- Exec Approvals：<https://docs.openclaw.ai/tools/exec-approvals>
- Configuration：<https://docs.openclaw.ai/gateway/configuration>
- Cron jobs：<https://docs.openclaw.ai/cron/>
- Automation hooks：<https://docs.openclaw.ai/automation/hooks>
- Models：<https://docs.openclaw.ai/concepts/models>
- Model Providers：<https://docs.openclaw.ai/concepts/model-providers>
- Messages：<https://docs.openclaw.ai/concepts/messages>
- Plugin Internals：<https://docs.openclaw.ai/plugins/architecture>
- `docs/pi.md`
- `docs/sessions-and-groups.md`
- `docs/session-compaction.md`
