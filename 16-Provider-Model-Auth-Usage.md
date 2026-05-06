# OpenClaw Provider、Model、Auth 与 Usage

## 1. 为什么这一层要单独分析

前面的文档已经覆盖了 Gateway、agent loop、context、tools 和 config。  
但模型调用本身还没有被完整展开。

OpenClaw 的模型层不只是“选一个 provider 调 API”。它至少包括：

- provider/model ref 解析
- model allowlist 与 alias
- per-agent model defaults
- session-level model override
- auth profiles
- provider 内部 auth failover
- fallback model chain
- usage/cost snapshot
- provider plugin runtime hooks

这一层直接影响 agent 的稳定性、成本、上下文能力、图片/PDF/媒体能力和失败恢复。

说明：
- 当前本地没有完整 OpenClaw 源码检出
- 本篇基于官方 Models、Model Providers、Model Failover、Authentication、CLI models 文档整理
- 具体实现仍建议后续用源码校验

## 2. 核心心智模型

OpenClaw 的模型调用链可以理解为：

```text
agent/session request
  -> resolve model ref
  -> check allowlist/catalog
  -> resolve provider
  -> resolve auth profile
  -> prepare provider-specific runtime config
  -> stream model response
  -> classify errors
  -> rotate auth profile or fallback model
  -> record usage/cost/cooldown state
```

这说明模型层同时是：
- 能力选择层
- 认证路由层
- 失败恢复层
- 成本观测层
- provider plugin 扩展层

## 3. Model selection 顺序

官方 Models 文档给出的选择顺序是：

```text
1. Primary model
   agents.defaults.model.primary
   或 agents.defaults.model

2. Fallback models
   agents.defaults.model.fallbacks

3. Provider auth failover
   在当前 provider 内部先轮换可用 auth
```

更准确地说，普通运行会先解析当前 session/agent 的 model override，再落到默认 primary/fallback。  
如果当前 provider 的一个 auth profile 失败，OpenClaw 会先尝试 provider 内部的 auth 轮换；如果该 provider 全部失败，再进入下一个 fallback model。

## 4. Model ref 与 alias

官方文档说明 model ref 采用：

```text
provider/model
```

例如：

```text
openai/gpt-5.4
anthropic/claude-sonnet-4-6
openrouter/moonshotai/kimi-k2
```

解析规则里有几个细节：

- model ref 按第一个 `/` 分割
- 如果 model id 自己包含 `/`，必须显式写 provider 前缀
- 省略 provider 时，优先匹配 alias
- 然后匹配唯一 configured-provider model id
- 最后才走 default provider fallback

这能避免 OpenRouter 这类 provider 的模型 ID 被错误拆分。

## 5. Allowlist 与 catalog

`agents.defaults.models` 是模型 allowlist/catalog。

如果设置了 allowlist，用户在 `/model` 或 session override 中选择不在列表里的模型，OpenClaw 会直接返回不允许的错误，而不是继续生成普通回复。

这说明模型 allowlist 是硬边界：

```text
模型不可用
  不是 prompt 拒绝
  而是运行前策略拒绝
```

对多 agent 部署来说，建议：
- coding agent 允许强工具模型
- public channel agent 限制为经过验证的模型
- image/pdf/media 模型单独配置
- fallback 不要指向能力差异过大的模型

## 6. Per-agent model defaults

官方 Models 文档提到 per-agent defaults 可以覆盖全局 defaults。  
这和 [12-Multi-Agent-路由与隔离.md](./12-Multi-Agent-路由与隔离.md) 的结论一致：

```text
agentId 是模型策略边界。
```

同一个 Gateway 下，不同 agent 可以拥有：
- 不同 primary model
- 不同 fallback chain
- 不同 auth profiles
- 不同 model allowlist
- 不同 image/pdf/media model

这让 OpenClaw 可以在同一控制面里同时运行：
- 高能力 coding agent
- 低成本日常聊天 agent
- 只读客服 agent
- 多媒体 agent

## 7. Session-level model switching

官方文档说明 `/model` 可以在聊天中切换当前 session 的模型。

几个关键行为：
- `/model` 和 `/model list` 打开 picker
- `/model <ref>` 可以直接指定
- 切换会立即持久化 session selection
- 如果 agent idle，下一轮直接使用
- 如果 run 已经 active，OpenClaw 会把 live switch 标成 pending
- 如果 tool 活动或回复输出已经开始，pending switch 可能要等到干净 retry 点或下一轮用户消息

这说明模型切换不是简单改变量。  
它必须和当前 run 的流式输出、工具调用、retry、transcript 保持一致。

## 8. 多媒体模型配置

官方 Models 文档列出几类专用模型：

```text
agents.defaults.imageModel
agents.defaults.pdfModel
agents.defaults.imageGenerationModel
agents.defaults.musicGenerationModel
agents.defaults.videoGenerationModel
```

这说明 OpenClaw 不把所有媒体任务都压到 text model 上。

例如：
- 主模型不能接受图片时，使用 image model
- PDF tool 可以使用 pdf model
- image/music/video generation 有独立 generation model

架构含义：

```text
model policy 不只是 chat model policy，
而是多能力 provider policy。
```

这也解释了为什么 provider plugin 不只是 text inference provider。

## 9. Provider plugins 的职责

官方 Model Providers 文档说明 provider plugins 可以注入 model catalog，并通过大量 runtime hooks 拥有 provider-specific 行为。

典型职责包括：
- model catalog
- auth env metadata
- model id normalization
- transport normalization
- config normalization
- API key resolution
- synthetic auth
- dynamic model resolution
- tool schema normalization
- reasoning output mode
- extra params
- stream function creation/wrapping
- OAuth refresh
- usage snapshot
- failover reason classification
- context overflow error matching
- missing auth message

这说明 provider plugin 是模型能力的 ownership boundary。  
核心不应该硬编码每个厂商的细节，而应该通过 provider capability contract 调用。

## 10. Auth profiles

官方 Authentication 与 Model Failover 文档说明，OpenClaw 支持：

- API key
- OAuth
- Claude CLI reuse
- setup-token / paste-token
- SecretRef-backed static credentials

auth profiles 存在 agent 目录下，例如：

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

而 config 里的 `auth.profiles` / `auth.order` 更偏 metadata 与 routing，不直接保存 secrets。

这点很重要：

```text
auth config 决定用谁，
auth store 保存真实凭据。
```

## 11. SecretRef 与 auth

官方文档说明 API key / token 类 credentials 可以使用 SecretRef。  
OAuth-mode profiles 不支持 SecretRef-backed `keyRef` / `tokenRef` 输入。

这说明 auth 的 secret 解析有明确边界：
- API key 和 token 可以来自 env/file/exec 等 secret source
- OAuth profile 是完整 OAuth 状态，不只是一个 key 引用
- `models status` 可以展示 marker，而不是泄露明文 secret

这和 [14-Config-Secrets-配置激活.md](./14-Config-Secrets-配置激活.md) 里的结论一致。

## 12. Auth failover

官方 Model Failover 文档说明 OpenClaw 失败恢复分两级：

```text
1. 当前 provider 内部 auth profile rotation
2. 切到 agents.defaults.model.fallbacks 中的下一个 model
```

这避免了一个 provider 的某个账号限流时立刻切 provider。  
更合理的顺序是：

```text
same provider, another auth profile
  -> same model or sibling model if allowed
  -> next fallback model
```

## 13. Cooldown 与 billing disable

官方文档区分：

```text
rate limit / overload
  短 cooldown，指数退避

billing / credit failure
  更长 disable window
```

cooldown 状态存储在 `auth-state.json` 的 `usageStats` 下。  
常见字段包括：
- `lastUsed`
- `cooldownUntil`
- `errorCount`
- `disabledUntil`
- `disabledReason`

这说明 model failover 不是 stateless retry。  
它会持久化 provider/profile 的健康状态，避免每次 run 都重复撞同一个失败 profile。

## 14. 哪些错误会触发 fallback

官方文档说明，fallback 主要针对：
- auth failures
- rate limits
- timeouts after profile rotation exhausted
- overloaded / provider busy signals
- billing/credit failures

不是所有错误都应该切模型。  
例如 schema 错误、prompt 构造错误、工具调用错误，通常不应该被当作 provider failover。

这类分类最好由 provider plugin 参与，因为不同 provider 的错误形态不同。

## 15. Usage 与 cost

`openclaw models status` 会展示 resolved default/fallbacks 和 auth overview。  
当 provider usage snapshots 可用时，OAuth/API-key 状态里还会包含 provider usage windows 和 quota snapshots。

官方 CLI 文档提到当前支持 usage-window 的 provider 包括 Anthropic、GitHub Copilot、Gemini CLI、OpenAI Codex、MiniMax、Xiaomi、z.ai 等。

usage auth 来源包括：
- provider-specific hooks
- auth profiles
- env credentials
- config

这说明 usage 不是简单统计本地 token。  
它还要对接 provider 自己的 quota/usage API 或状态。

## 16. `models status` 的诊断价值

`models status` 至少能回答：
- 当前 primary model 是什么
- fallback chain 是什么
- image/pdf/media model 是什么
- 每个 provider 的 auth 是否可用
- OAuth 是否快过期
- env/config/store 哪个来源生效
- 是否有 missing auth
- live probe 是否通过
- usage/quota snapshot 是否可用

`--check` 可以用于自动化，缺失或过期时用退出码表达。  
`--probe` 会发起真实请求，因此可能消耗 token 或触发 rate limit。

这适合放进运维诊断，而不是每次普通状态查看都跑 probe。

## 17. Models registry

官方 Models 文档说明 custom providers 会写入 agent 目录下的 `models.json`：

```text
~/.openclaw/agents/<agentId>/agent/models.json
```

并且 `models.providers` 与 agent `models.json` 有 merge 逻辑。

几个关键原则：
- agent-local `baseUrl` 可优先
- SecretRef-managed `apiKey` 不应把 resolved secret 持久化进去
- source marker 要保持 source-authoritative
- config 更新可以刷新 normalized catalog data

这说明 `models.json` 不是单纯 cache，也不是唯一真相。  
它是 agent-local provider/model registry，与 active config 和 secret source 共同决定运行时模型可用性。

## 18. 与 Context / Tools 的关系

模型层会影响 context 与 tools：
- context window 决定 compaction 触发与 token budget
- provider 支持的 tool schema 决定工具适配
- reasoning 输出模式影响 transcript hygiene
- image support 决定是否需要 imageModel
- provider cache eligibility 影响 system prompt cache 策略

因此 model selection 不应只按“最强模型”选择。  
还要看：
- tool calling 稳定性
- image/media support
- context window
- reasoning visibility
- cache support
- rate limit
- cost
- auth 可用性

## 19. 常见风险

### 19.1 Fallback 能力不等价

主模型支持 tool calling / image，fallback 不支持，会导致运行中能力退化。

### 19.2 Provider auth 看起来存在但不可执行

env marker、auth profile、models.json 都可能显示“有配置”，但真实 probe 才能证明能调用。

### 19.3 Session override 绕过团队策略

如果 allowlist 太宽，用户在 channel 里切到高成本或弱安全模型。

### 19.4 OAuth profile 过期

长驻 Gateway 更适合 API key 或稳定 token。OAuth 需要过期检查和刷新策略。

### 19.5 Usage snapshot 被当作本地 token 统计

provider usage window 是 provider 侧 quota 信息，不等同于 session token usage。

## 20. 对源码阅读的建议

拿到完整源码后建议读：

1. `src/agents/model-selection.ts`
2. `src/agents/model-auth.ts`
3. `src/agents/auth-profiles.ts`
4. provider plugin registry 相关实现
5. `models status` CLI 实现
6. `models.json` merge/refresh 逻辑
7. `auth-state.json` cooldown/disable 状态读写
8. provider runtime hooks
9. tool schema normalization 入口
10. usage snapshot fetch 入口

重点问题：
- session override、agent default、global default 的优先级
- model allowlist 在哪里拒绝
- live model switch 的 pending 状态如何落盘
- auth profile rotation 是否按 session 粘性处理
- provider error 如何分类为 rate_limit / overloaded / billing / auth
- usage snapshot 是否会阻塞正常 agent run

## 21. 结论

OpenClaw 的模型层可以概括成：

```text
Model runtime =
  model policy
  + provider plugin capability
  + auth profile routing
  + failover/cooldown state
  + usage/cost observability
```

真正要记住的是：

```text
模型选择不是一个字符串，
而是 agent/session/provider/auth/tool/context 共同解析出的运行时决策。
```

## 22. 参考

- Models：<https://docs.openclaw.ai/concepts/models>
- Model Providers：<https://docs.openclaw.ai/concepts/model-providers>
- Model Failover：<https://docs.openclaw.ai/concepts/model-failover>
- Model provider authentication：<https://docs.openclaw.ai/gateway/authentication>
- Models CLI：<https://docs.openclaw.ai/cli/models>

