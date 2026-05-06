# OpenClaw Config、Secrets 与配置激活

## 1. 为什么配置面值得单独分析

OpenClaw 的配置不是普通应用里的“读一个 json”。  
它会影响：

- Gateway 监听与远端接入
- agent 列表与 multi-agent 路由
- model/provider/auth
- channels
- tools 与 sandbox
- plugins 与 hooks
- secrets
- reload 与升级策略

因此配置系统本身就是控制面的一部分。  
如果配置激活、校验、密钥引用和 reload 机制不稳定，Gateway、agent loop、plugin runtime 都会受影响。

说明：
- 当前本地没有完整源码检出
- 本篇基于官方 configuration、secrets、tools、agents 配置文档整理
- 涉及实现路径时作为后续源码阅读指引

## 2. 配置文件定位

官方配置文档说明，Gateway 会从：

```text
~/.openclaw/openclaw.json
```

读取全局配置。

这个文件是可选的。  
如果不存在，OpenClaw 会使用安全默认值启动。

这说明配置系统的首要设计目标不是“必须声明所有东西”，而是：

```text
默认可运行，
需要扩展时再显式覆盖。
```

## 3. JSON5 与 include

官方文档提到配置文件支持 JSON5。  
这意味着可以有注释、尾逗号等更适合人工维护的写法。

配置还支持 `$include`。  
这对复杂部署很有用，例如：

```text
openclaw.json
  -> includes/channels.json5
  -> includes/agents.json5
  -> includes/tools.json5
```

但 include 也带来治理问题：
- 哪个文件覆盖哪个文件
- 循环 include 如何处理
- include 后的最终配置如何审计
- 配置错误如何回滚

所以更适合把 include 用在边界清楚的配置块，而不是过度拆碎。

## 4. 配置编辑入口

官方文档列出多种配置方式：

- `openclaw onboard`
- `openclaw configure`
- `config get`
- `config set`
- `config unset`
- Control UI
- 直接编辑 `openclaw.json`

这说明配置不是纯文件系统问题。  
Gateway 运行时也会暴露配置读写能力。

结合 Gateway Protocol，可推断配置相关 RPC 属于高权限控制面方法。  
这也解释了为什么 Gateway Protocol 把 `config.*` 归到 admin 级别。

## 5. Schema 校验

官方配置文档强调配置写入会做严格 schema 校验。  
Control UI 也可以基于 schema 渲染配置界面。

这对平台很关键：

```text
配置 schema =
  人类可编辑边界
  + UI 表单边界
  + Gateway RPC 校验边界
  + agent system prompt 可查询边界
```

在 [09-context-上下文管理/02-system-prompt-设计.md](./09-context-上下文管理/02-system-prompt-设计.md) 中已经提到，OpenClaw 会提醒模型用 `config.schema.lookup` 查看配置 schema，再考虑 `config.patch` 或 `config.apply`。

这说明 config schema 不只是开发者文档，也是 agent 自我修改配置时的安全辅助。

## 6. Hot Reload 与 Restart

官方配置文档把配置变更分成几类：

```text
hot reload
  可在运行中生效

restart required
  需要重启 Gateway

hybrid
  部分字段热生效，部分字段需要重启

off
  不自动 reload
```

这对控制面设计有两个含义：

1. 配置字段必须带生效语义  
   不能所有字段都假装能热更新。

2. Gateway 需要管理 active config 与 pending config  
   否则 UI 显示“已保存”和运行时真实生效状态可能不一致。

## 7. Active Config 与 Last-known-good

官方文档提到配置会校验并激活，失败时可保留上一份可用配置。

这类机制通常可以理解为：

```text
candidate config
  -> parse
  -> include resolution
  -> env / secret resolution where allowed
  -> schema validation
  -> activation plan
  -> apply hot fields
  -> persist or keep last-known-good
```

分析判断：
- OpenClaw 不应该让一份坏配置把 Gateway 直接带崩
- 对远端 Gateway 来说，配置回滚能力尤其重要
- UI/CLI 应该能告诉用户哪些字段还需要 restart

## 8. 环境变量加载

官方文档说明 Gateway 会加载：

```text
父进程环境变量
当前工作目录的 .env
~/.openclaw/.env
```

并且 `.env` 不会覆盖已经存在的环境变量。

这体现了一个常见安全原则：

```text
启动环境优先于本地 .env 文件。
```

否则远程或部署系统传入的环境可能被工作目录里的 `.env` 意外覆盖。

## 9. Secrets 与 SecretRef

OpenClaw 支持 secrets 管理，并在配置中使用 SecretRef。

高层模型：

```text
secret store
  保存真实密钥值

openclaw.json
  保存 SecretRef 引用

runtime resolver
  在需要时解析 SecretRef
```

这比把 API key 直接写进 `openclaw.json` 更安全，也更利于备份配置文件。

SecretRef 的关键价值是：
- 配置可版本化
- 密钥不随配置泄漏
- UI 可以展示“已配置/未配置”，不展示明文
- agent 修改配置时不需要看到真实 secret

## 10. Secrets 的控制面风险

Secrets 一旦进入 Gateway API，就必须非常克制。

需要区分：

```text
secret presence
  是否存在、名称、用途

secret value
  真实密钥内容
```

大多数 UI 和 agent 工具只应该看到 presence，而不是 value。

Gateway Protocol 中存在 `operator.talk.secrets` 这类 scope，说明官方也把 secret 访问看成更高风险的控制面能力。

## 11. Config 与 Multi-agent

配置里会定义 agents。  
这会影响：

- `agentId`
- workspace
- agentDir
- default agent
- channel bindings
- per-agent model/auth/tools/sandbox

这和 [12-Multi-Agent-路由与隔离.md](./12-Multi-Agent-路由与隔离.md) 强相关。

错误配置的典型风险：
- 两个 agent 复用 `agentDir`
- 默认 agent 不明确
- channel binding 太宽
- per-agent tools 继承了全局高权限工具
- workspace 路径指向同一个目录

所以 agent config 应该被当成隔离边界配置，而不只是展示列表。

## 12. Config 与 Tools

tools 配置是配置系统里风险最高的一块之一。

例如：

```text
tools.profile
tools.allow
tools.deny
tools.byProvider
tools.exec
```

这些字段会直接改变模型可见工具集合。  
一旦热更新，就可能让正在运行或下一轮 agent run 看到不同能力。

需要重点核查：
- tool config 变更是否影响 active runs
- 变更后是否需要重新生成 system prompt 和 tool schemas
- provider-specific 工具策略是否有确定性合并顺序
- UI 是否明确展示当前 agent 实际 tool set

## 13. Config 与 Plugins

plugins 配置会影响：
- 哪些插件启用
- 插件是否 unsafe
- 插件能注册哪些 hooks
- 插件能暴露哪些 Gateway RPC / tools / providers / channels
- 插件 approval 行为

插件配置如果支持热加载，就必须考虑：
- 插件启动失败是否回滚
- hook 注册是否重复
- 插件禁用后是否清理 runtime state
- 插件 RPC method 是否从 feature list 中移除

这也是后续值得补充 Plugin Runtime 细节的原因。

## 14. Config 与 Agent 自我修改

OpenClaw 的 system prompt 里提到模型可以在一定条件下使用 `config.patch` / `config.apply` 或 update 相关工具。

这意味着配置系统面向的不只是人类操作者，也包括 agent。

风险和价值同时存在：
- 价值：agent 可以帮助用户调整模型、工具、插件、渠道
- 风险：agent 如果误改配置，可能改变自己的权限或执行边界

因此 agent 修改配置至少应满足：
- 先查询 schema
- 生成 patch，而不是直接覆盖整文件
- 高风险字段需要 approval
- secrets 不暴露明文
- 修改后有 validation 和 rollback

## 15. 推荐分析维度

如果继续深挖配置系统，建议按这张表做笔记：

| 维度 | 要问的问题 |
| --- | --- |
| 来源 | 文件、env、SecretRef、CLI、UI、RPC 谁能改 |
| 校验 | schema 在哪里定义，运行时如何执行 |
| 合并 | include、profile、agent override 的优先级 |
| 激活 | hot、restart、hybrid、off 如何区分 |
| 回滚 | 配置失败是否保留 last-known-good |
| 审计 | 谁改了什么，是否有日志 |
| 权限 | 哪些字段需要 admin / approval |
| 可见性 | agent 和 UI 能看到多少 |

## 16. 对源码阅读的建议

拿到完整源码后建议读：

1. config schema 定义
2. `src/config/*`
3. Gateway `config.*` server methods
4. secrets store 实现
5. SecretRef resolver
6. config reload watcher / activator
7. Control UI config editor
8. `system-prompt.ts` 中 config guidance
9. plugin config activation 路径
10. tools config merge 路径

重点问题：
- schema 是否和文档保持一致
- include resolution 是否防循环
- SecretRef 解析是否只在需要时发生
- config patch 是否保留注释或只写 canonical JSON
- hot reload 如何通知 running sessions
- bad config 是否会影响已建立 Gateway 连接

## 17. 结论

OpenClaw 的 config 系统是控制面核心，不是辅助文件。

更准确的模型是：

```text
Config =
  platform topology
  + agent isolation
  + model/provider/auth
  + tools/sandbox policy
  + channel/plugin wiring
  + reload/activation contract

Secrets =
  sensitive values hidden behind references

Activation =
  validation + hot/restart planning + rollback
```

如果后续做二开，配置系统应当优先保持保守：  
先保证 schema、权限、回滚和审计，再追求动态热更新能力。

## 18. 参考

- Gateway configuration：<https://docs.openclaw.ai/gateway/configuration>
- Secrets：<https://docs.openclaw.ai/gateway/secrets>
- Tool configuration：<https://docs.openclaw.ai/gateway/config-tools>
- Configuration agents：<https://docs.openclaw.ai/gateway/config-agents>
- Gateway Protocol：<https://docs.openclaw.ai/gateway/protocol>

