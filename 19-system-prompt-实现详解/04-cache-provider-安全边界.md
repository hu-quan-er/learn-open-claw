# Cache、Provider 与安全边界

OpenClaw system prompt 的设计有三个容易被低估的重点：
- 它是 prompt cache 友好的
- 它允许 provider 做有限调优
- 它承认 prompt 只是软约束，硬边界必须靠工具和运行时策略

## 1. Prompt cache 是一等设计约束

官方 Prompt Caching 文档说明，OpenClaw 会把 system prompt 拆成：

```text
stable prefix
cache boundary
volatile suffix
```

稳定前缀中放相对静态的内容：
- identity
- tooling
- tool call style
- execution bias
- safety
- skills metadata
- memory guidance
- workspace/docs/sandbox
- stable project context files

动态后缀中放容易变化的内容：
- `HEARTBEAT.md` 等 dynamic project context
- channel/session-specific guidance
- group/subagent context
- reactions
- provider dynamic suffix
- heartbeats
- runtime metadata

这个拆分的目标不是语义优雅，而是让 provider 能复用 unchanged prompt prefixes，降低成本和延迟。

## 2. 稳定性的具体手段

### 2.1 固定排序

工具和 context files 都做稳定排序。

工具方面：
- core tools 走固定 `toolOrder`
- 额外工具排序后追加
- dedupe 时保留调用方 casing

文件方面：
- workspace bootstrap files 有固定权重
- 同权重再按 basename/path 排序
- dynamic files 单独分流

排序稳定的价值很直接：相同语义不应因为数组顺序不同导致 cache miss。

### 2.2 稳定 prefix 本地缓存

源码里有 `stablePromptPrefixCache`，用 hash 后的 prompt 输入作为 key，缓存 stable prefix 字符串。

这个缓存不是 provider prompt cache，而是 OpenClaw 进程内避免重复构造 stable prefix 的小缓存。它和 provider cache 目标一致：让稳定部分可复用。

### 2.3 prompt fingerprint 归一化

官方 Prompt Caching 文档提到 system prompt fingerprint 会对 whitespace、line ending、hook-added context、runtime capability ordering 做归一化。

这说明 OpenClaw 在 cache 上不是只依赖 provider，而是自己先把 prompt shape 变得 deterministic。

### 2.4 时间处理让位于 cache

System Prompt 文档说明，当前日期时间 section 现在主要保留 timezone，不再每轮注入动态 clock；需要当前时间时用 `session_status`。

这是一个非常典型的系统设计取舍：
- 动态时间放进 system prompt 会导致每轮 cache prefix 变化
- 改用工具查询会增加一次工具调用
- OpenClaw 选择保护 prompt cache 稳定性

## 3. Provider 贡献模型

`ProviderSystemPromptContribution` 的形状很克制：

```text
stablePrefix?: string
dynamicSuffix?: string
sectionOverrides?: {
  interaction_style?: string
  tool_call_style?: string
  execution_bias?: string
}
```

这说明 provider 不能接管整份 prompt，只能：
- 替换少数模型行为相关 section
- 在稳定边界上方加入静态模型族指导
- 在动态边界下方加入确实会变化的 provider 文本

官方 System Prompt 文档也提到，provider plugins 可以贡献 cache-aware prompt guidance，但不替换完整 OpenClaw-owned prompt。

## 4. 为什么不让 provider 替换整份 prompt

分析推断，有三个原因。

第一，OpenClaw 的 prompt 同时承载工具、安全、workspace、channel、sandbox、memory、skills 等宿主协议。如果 provider 替换整份 prompt，容易丢失宿主约束。

第二，provider 的最佳实践经常变，但 OpenClaw 的运行时 contract 需要稳定。有限 overlay 可以调优模型族行为，又不破坏平台 contract。

第三，cache boundary 需要跨 provider 统一。若 provider 完全接管 prompt，OpenClaw 就很难保证 stable prefix 与 volatile suffix 的一致拆分。

## 5. Prompt 是软约束

官方 System Prompt 文档明确说，system prompt 中的 safety guardrails 是 advisory。它们指导模型行为，但不执行 policy。硬约束来自：
- tool policy
- exec approvals
- sandboxing
- channel allowlists
- gateway/config 权限

因此不能把 prompt 里的 “do not” 当成安全边界。

## 6. Tool visibility 才是关键硬边界

OpenClaw 的工具设计隐含一个原则：

```text
如果不希望模型执行某类动作，最可靠的方法是不要暴露对应 tool schema。
```

prompt 可以告诉模型不要执行 shell 命令，但如果 exec tool 可见，模型仍可能尝试调用。真正的控制应该在：
- tool allow/deny
- per-agent tool profile
- per-channel tool policy
- provider-specific tool compatibility
- exec approval rule
- sandbox workspace access

Tooling section 更像是“当前可见能力的说明书”，不是权限系统本身。

## 7. Safety section 的真实作用

`Safety` section 虽然不是硬边界，但仍然重要。它的作用是降低模型主动越界的概率，尤其是以下倾向：
- 为完成任务而忽视 oversight
- 说服用户放宽限制
- 自行修改安全规则
- 把长期目标凌驾于当前用户请求
- 在权限不足时绕道执行

它是 defense-in-depth 中的语言层约束。即使后面还有工具 policy 和 sandbox，这层仍能减少很多无意义或高风险的 tool call 尝试。

## 8. Self-update 的风险控制

OpenClaw 允许 agent 使用 gateway tool 做 config/restart/update，但 prompt 明确收紧：
- 不要编造命令
- 配置字段先查 schema
- update 只有显式用户请求才做
- restart 用 gateway/restart，而不是 stop+start
- 受保护 exec 配置不能被随便改写

这说明 OpenClaw 没有简单禁止 agent 自维护，而是把自维护做成显式、可审计、工具化路径。

风险仍然存在：如果 gateway tool 暴露给不合适的 agent/channel，prompt 只能劝阻，不能代替权限边界。

## 9. Sandbox prompt 与 sandbox enforcement 的分工

Sandbox section 告诉模型当前环境怎么用，但不负责创建 sandbox。真正的隔离来自 runtime：
- Docker/container 执行
- workspace mount
- elevated exec policy
- host/browser access policy
- per-agent sandbox config

prompt 的职责是减少误用：
- 避免路径混淆
- 避免让 sandboxed sub-agent 请求 host access
- 避免建议用户切到不可用的 `/elevated full`

所以 sandbox prompt 是“可用性和边界解释层”，不是隔离实现层。

## 10. Channel 安全边界

OpenClaw 的 message channel 也有安全含义：
- source channel 的 sender 不一定是 owner
- inline approvals 可能由 channel native UI 承载
- message tool 可能发送到不同 channel
- group chat context 和 private session context 不应混淆

system prompt 会描述这些差异，但实际安全仍依赖 channel allowlist、owner identity、native approval runtime 和 message routing policy。

## 11. 和 agent loop 的关系

system prompt 在 agent loop 中处于模型调用前，但它不是一次性背景材料。它会影响后续完整链路：

```text
system prompt
  -> 模型选择 tool call
  -> OpenClaw tool policy / sandbox / approval 执行或拒绝
  -> tool result 回写 transcript
  -> reply shaping / duplicate suppression
  -> session persistence
```

因此 prompt 设计和 agent loop 是互相咬合的：
- prompt 告诉模型如何使用工具
- agent loop 真正执行工具、发事件、落 transcript
- reply shaping 处理 `NO_REPLY` 和 message tool 去重
- 下一轮 prompt 又基于新的 session/context 重建

## 12. 设计判断

OpenClaw 的 system prompt 设计有几个值得借鉴的点：

1. Prompt owned by runtime
   - 平台自己拥有完整 prompt contract，provider 只做局部贡献。

2. Section by runtime problem
   - 每个 section 解决一个运行问题，而不是泛泛堆规则。

3. Capability-shaped prompt
   - 有什么工具、channel、sandbox、provider 能力，就注入什么指导。

4. Cache-aware layout
   - 稳定内容在上，易变内容在下，中间有 cache boundary。

5. Soft prompt + hard policy
   - prompt 解释和引导，权限与安全由工具、审批、sandbox、channel policy 执行。

6. Index-first context
   - skills 和 memory 默认给索引，不全量塞正文。

如果要复刻这类设计，重点不是复制 OpenClaw 某句 wording，而是复制这些结构性原则。
