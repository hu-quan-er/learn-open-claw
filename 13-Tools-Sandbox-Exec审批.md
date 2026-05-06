# OpenClaw Tools、Sandbox 与 Exec 审批

## 1. 为什么这一层要单独分析

前面的安全文档讲的是结构性边界：Gateway、pair、wizard、sandbox、queue。  
这一篇补的是安全真正落地的执行面：

- agent 到底能看到哪些 tools
- `exec`、`write`、`edit`、`apply_patch` 这类高风险工具如何被限制
- sandbox 与 approvals 分别管什么
- host 与 node 执行能力如何收敛
- 工具策略怎样按 agent、provider、channel 分层

对 agent 平台来说，这层是最容易出事故的地方。  
因为“模型能不能做某件事”最终不是由一句 prompt 决定，而是由工具暴露、执行环境、审批策略和上下文共同决定。

说明：
- 当前本地没有完整 OpenClaw 源码检出
- 本篇基于官方 configuration tools、exec approvals、sandbox、gateway protocol 文档整理
- 具体实现仍建议后续用源码校验

## 2. 工具能力不是单一开关

OpenClaw 的工具策略更像多层叠加：

```text
tools.profile
  -> tools.allow / tools.deny
  -> tools.byProvider
  -> tools.byAgent
  -> tools.byChannel
  -> tools.exec.*
  -> sandbox / approvals
  -> runtime actual tool catalog
```

这意味着：
- profile 给出默认能力集合
- allow/deny 做显式收口或放开
- provider/agent/channel 可以进一步覆盖
- `tools.exec` 决定 shell 执行行为
- sandbox 和 approvals 决定最终是否能执行

不要把 `tools.profile = "full"` 理解成“完全无边界”。  
它只是工具目录上的宽 profile，后面仍可能被 sandbox、approval、host policy、node permission 拦住。

## 3. Tool profiles

官方配置文档给出几种 profile：

```text
minimal
  只保留消息、会话、运行时状态这类低风险工具

messaging
  适合聊天渠道，包含消息发送、会话控制和有限媒体能力

coding
  适合代码工作，包含 read/search/edit/apply_patch/exec 等开发能力

full
  开放完整 catalog，通常只适合可信本地开发环境
```

分析判断：
- `minimal` 更像默认安全基线
- `messaging` 面向远端聊天入口
- `coding` 面向本地代码 agent
- `full` 应该只在非常明确的信任域里使用

## 4. Tool groups

官方配置支持 `group:<name>` 形式的 tool group。

常见 group 包括：

```text
group:messaging
group:session
group:runtime
group:filesystem
group:coding
group:node
group:memory
group:network
group:browser
group:knowledge
group:automation
group:plugins
group:admin
group:voice
```

group 的价值是避免策略文件写成超长工具名清单。  
例如：

```text
allow:
  - group:messaging
  - group:session

deny:
  - exec
  - apply_patch
```

这比逐个列出所有消息工具更稳定。  
后续平台新增某个 messaging tool 时，只要归组正确，策略就能自然继承。

## 5. Allow 与 deny

工具策略通常遵循一个重要原则：

```text
deny 优先。
```

也就是说，某个工具即使命中了 allow，只要也命中了 deny，就应该被隐藏或禁用。

官方文档也提示匹配是大小写不敏感，支持通配符。  
这对策略维护有用，但也带来风险：

```text
deny: ["*write*"]
```

这类宽通配符可能会误杀正常工具。  
更稳妥的方式是按 group 和明确工具名组合。

## 6. Provider-specific 工具策略

`tools.byProvider` 可以按 provider 或 provider/model 覆盖工具集合。

这说明 OpenClaw 没有假设所有模型 provider 能力相同。  
不同 provider 在 tool calling、schema、streaming、reasoning 能力上差异很大，因此 tool catalog 需要按 provider 调整。

典型用途：
- 对不稳定 provider 禁用复杂工具
- 对不支持某种 schema 的模型隐藏特定工具
- 对低成本模型保留只读工具
- 对高能力模型开放 coding profile

这和 [09-context-上下文管理](./09-context-上下文管理/README.md) 里提到的 provider hygiene 是同一类问题：  
模型差异不能只靠 prompt 兜底，工具层也要适配。

## 7. Agent-specific 与 Channel-specific 策略

多 agent 模式下，工具策略还可以按 agent 覆盖。  
这和 [12-Multi-Agent-路由与隔离.md](./12-Multi-Agent-路由与隔离.md) 的结论一致：

```text
agentId 不只是身份标签，
它也是工具能力、sandbox、auth、workspace 的隔离键。
```

channel-specific 策略也很重要。  
因为 Telegram、WhatsApp、Discord、Slack、Web UI、CLI 的信任等级不同：
- 本地 CLI 可以更宽
- 公共群聊应该更窄
- 远端移动设备应限制高风险工具
- 业务账号可能只允许 messaging 和 session 工具

这类策略如果只靠用户“别让 agent 做危险事”是不够的，必须写进工具暴露面。

## 8. `tools.exec` 是高风险中心

`exec` 不是普通工具，它是直接连到宿主或节点执行环境的能力。

官方配置里 `tools.exec` 涉及：
- 是否启用
- 执行位置
- 超时
- 后台执行
- cleanup
- 是否允许 `apply_patch`
- 是否通知

高层可以理解为：

```text
exec tool
  -> policy check
  -> approval check
  -> sandbox / host / node resolution
  -> process execution
  -> result capture
  -> transcript / stream
```

真正的安全问题通常不在“有没有 exec”，而在：
- exec 在哪里运行
- cwd 是不是 workspace
- 是否受 sandbox 约束
- 是否需要 approval
- 是否允许后台长任务
- 是否允许访问网络和宿主文件系统

## 9. Sandbox 与 Approval 的分工

这两个概念容易混：

```text
sandbox
  限制执行环境和文件系统边界

approval
  决定某个命令是否需要人类或策略批准
```

一个命令可以：
- 在 sandbox 里执行，但仍需要 approval
- 不在 sandbox 里执行，但由 allowlist 自动放行
- 被 sandbox 拦住，无论是否审批通过
- 被 approval policy 拒绝，即使 sandbox 允许

所以更准确的模型是：

```text
工具是否可见
  -> 命令是否符合 exec policy
  -> 是否需要 approval
  -> approval 是否通过
  -> sandbox / host / node 是否允许执行
```

## 10. Exec Approvals

官方 exec approvals 文档说明，审批策略主要用于托管 `exec` 工具。

典型配置项包括：
- `mode`
- `ask`
- `askFallback`
- `allowlist`
- `denylist`
- `safeBins`
- `strictInlineEval`
- `policy`

可以把它们分成三类：

```text
安全等级
  mode / policy

命中策略
  allowlist / denylist / safeBins

交互策略
  ask / askFallback
```

## 11. `mode` 与 `policy`

官方文档里的 `mode` 包括：

```text
off
  关闭审批

suggest
  提示或建议式审批

enforce
  强制执行审批策略
```

`policy` 则更像安全等级：

```text
full
  最宽松

allowlist
  只允许明确命中的命令

safe
  更偏保守的安全基线

deny
  拒绝高风险命令
```

分析判断：
- `mode` 决定审批系统是否真正拦截
- `policy` 决定拦截时采用哪种规则集

生产或远端接入场景下，应避免把 `mode` 关掉。

## 12. `ask` 与 `askFallback`

官方文档提到 `ask` 可控制什么时候询问：

```text
off
on-miss
always
```

高层含义：
- `off`：不问人
- `on-miss`：规则没命中时问
- `always`：每次都问

`askFallback` 则处理无法询问时的行为。  
这对 cron、webhook、headless node、后台任务很关键。

如果自动化任务里需要 exec，但没有可用 operator 回答 approval，系统必须知道：
- 默认拒绝
- 默认允许
- 还是进入 pending 状态

这就是为什么 approval 不能只理解成 UI 弹窗。

## 13. Allowlist 与 Denylist

执行审批里 allowlist/denylist 的作用不是隐藏 tool，而是判断某条 shell 命令能否执行。

例子：

```text
allowlist:
  - git status
  - rg *
  - npm test

denylist:
  - rm -rf *
  - curl * | sh
  - chmod *
```

实际策略应比例子更严格，尤其要注意：
- shell 管道
- 重定向
- command substitution
- inline script
- package manager install
- network download
- credential path 访问

官方文档还提到 `strictInlineEval`。  
这类选项的价值就是避免命令表面看起来安全，实际通过 `sh -c`、`node -e`、`python -c` 绕开。

## 14. Node Host 执行边界

OpenClaw 支持 `role: "node"` 的能力端，这意味着 exec 不一定永远在 Gateway host 上执行。

需要区分：

```text
gateway host
  Gateway 进程所在机器

agent workspace
  agent 默认 cwd

sandbox workspace
  受限执行目录

node host
  远端或移动/桌面节点设备
```

如果执行位置可以在 gateway 和 node 之间切换，安全策略就必须明确：
- 谁能触发 node 命令
- node 声明了哪些 command
- Gateway 是否做服务端 allowlist
- node approval 谁来批准
- 执行结果如何回流 transcript

这也是 Gateway Protocol 把 `operator` 和 `node` 分成两个角色的原因之一。

## 15. Tool visibility 与 Prompt 的关系

工具是否出现在模型可见 tool schema 里，会直接影响模型行为。

如果某个 agent 不应该执行命令，最稳妥的是：

```text
不要给它 exec tool schema。
```

而不是只在 system prompt 里说“不要执行命令”。  
prompt 是软约束，tool catalog 是硬约束。

这和 context 文档中的结论一致：
- tool schemas 本身是 context 开销
- tool schemas 会影响模型可行动作
- 工具排序和可见性应该尽量 deterministic

## 16. 常见风险

### 16.1 远端 channel 继承本地 coding profile

如果 Telegram/Discord 入口拿到了本地 coding profile，就可能把群聊消息转成宿主机命令执行。

### 16.2 `full` profile 配合 approval off

这基本等价于把 agent 变成高权限自动执行器。

### 16.3 只限制 exec，忘了 write/edit/apply_patch

很多破坏不需要 shell。  
文件写入、patch、browser action、node command 也需要纳入策略。

### 16.4 宽泛 allowlist

例如允许 `npm *`、`python *`、`node *`，通常等价于允许任意代码执行。

### 16.5 sandbox 被误解为 workspace

workspace 是默认目录，不是强隔离。  
只有真实 sandbox policy 才能提供执行边界。

## 17. 对源码阅读的建议

拿到完整源码后建议读：

1. `src/agents/pi-tools.ts`
2. `src/agents/pi-tools.policy.ts`
3. `src/agents/pi-tools.schema.ts`
4. `src/agents/openclaw-tools.ts`
5. `src/agents/tools/*`
6. `src/agents/sandbox.ts`
7. `src/gateway/server-methods/*approval*`
8. `src/node-host/*`
9. 配置 schema 中 `tools` / `sandbox` / `approvals` 相关部分

重点问题：
- tool profile 如何展开成具体工具集合
- allow/deny 是在 schema 生成前还是工具调用时生效
- provider-specific 工具策略如何和全局策略合并
- exec approval 是同步阻塞还是 pending/resume
- node command 是否存在 Gateway-side allowlist
- sandbox 对 symlink、绝对路径、后台进程如何处理

## 18. 结论

OpenClaw 的工具安全不是单点机制，而是一组叠加边界：

```text
tool catalog
  -> profile
  -> allow/deny
  -> provider/agent/channel overrides
  -> exec policy
  -> approvals
  -> sandbox / host / node boundary
```

真正要守住的是：

```text
模型可见能力必须小于等于当前入口、agent、workspace、设备和审批策略允许的能力。
```

这层如果分析透，OpenClaw 的安全模型就不再停留在概念层，而能落到每一次 tool call。

## 19. 参考

- Tool configuration：<https://docs.openclaw.ai/gateway/config-tools>
- Exec Approvals：<https://docs.openclaw.ai/tools/exec-approvals>
- Sandbox CLI：<https://docs.openclaw.ai/cli/sandbox>
- Gateway Protocol：<https://docs.openclaw.ai/gateway/protocol>

