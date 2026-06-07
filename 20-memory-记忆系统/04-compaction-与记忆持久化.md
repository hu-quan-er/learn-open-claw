# compaction 与记忆持久化

记忆系统的真正考验发生在 **context 写满、要做 compaction（压缩）的那一刻**。这一篇讲：压缩会丢什么、记忆怎么在压缩前抢救、什么东西能跨压缩存活。

这是整套记忆设计的「为什么存在」——如果 context 永不压缩，就不太需要落盘记忆。正是因为压缩有损，记忆才必须落盘。

## 1. 三种「记忆丢失」的根因

**【事实】** masterclass 把记忆丢失归为三类，分清它们才能理解 flush 在防什么：

| 根因 | 机制 | 是否有损 |
|---|---|---|
| **从未存储**（最常见）| 信息只在对话里，没写进文件 | 永久丢失 |
| **compaction 丢失** | 长会话触发摘要，丢细节/细微差别/具体约束 | 有损 |
| **session pruning** | 按请求在内存里裁剪旧 tool 输出 | 临时、对磁盘无损 |

**【事实】** 文档明确区分 compaction 与 pruning：

```text
Compaction  把整段对话历史摘要成一份紧凑 summary —— 有损
Pruning     只在内存里、按请求裁掉旧 tool 结果 —— 临时
```

pruning 的 cache-ttl 模式（默认 `"5m"`）：

```json
{ "contextPruning": { "mode": "cache-ttl", "ttl": "5m" } }
```

> pruning 是为了在「还没到必须 compaction」之前，先把 tool 输出这种体积大、价值低的东西裁掉，**推迟 compaction 的到来**。因为 compaction 会让 prompt cache 失效（见第 5 节），能晚点压缩就晚点失效。

## 2. compaction 前的 memory flush：临界抢救

**【事实】** 这是记忆持久化最关键的自动机制。在 compaction 把对话摘要掉之前，OpenClaw 跑一个**静默 turn**，提醒 agent 把重要上下文存进记忆文件。**默认开启。**

```text
Before compaction summarizes conversations, OpenClaw runs a silent turn
that reminds the agent to save important context to memory files.
```

相关配置参数（**【事实/社区拆解】**）：

| 参数 | 含义 | 值 |
|---|---|---|
| `memoryFlush.enabled` | 是否启用 flush | `true`（需确认）|
| `reserveTokensFloor` | 预留 headroom 缓冲 | `40000` |
| `softThresholdTokens` | 触发前的距离 | `4000` |
| `systemPrompt` | flush 的系统提示 | `"Session nearing compaction. Store durable memories now."` |
| `prompt` | flush 的指令 | `"Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."` |

触发逻辑（**【推断】** 由参数语义推断）：

```text
当  剩余 = context窗口 - reserveTokensFloor - softThresholdTokens
    快要见底时
  -> 先跑一个静默 agentic turn (flush)
  -> agent 把该记的写进 memory/<today>.md (没有就回 NO_REPLY)
  -> 然后才真正做 compaction
```

**这是为什么要留 headroom。** 文档原话：「The entire point of the headroom config… is to stay on the good path」——即区分两条路径：

```text
好路径: maintenance compaction —— 还有余量时主动、温和地压缩, flush 先抢救
坏路径: overflow recovery       —— 撑爆了被迫压缩, 最大损失, 来不及 flush
```

留 40K token 的地板，就是为了保证「到达临界前永远有空间先 flush 再压缩」，而不是被动溢出。

**【事实】** flush 用的模型可单独覆盖（对本地模型有用）：

```json
{ "agents": { "defaults": { "compaction": {
  "memoryFlush": { "model": "ollama/qwen3:8b" }
}}}}
```

## 3. 什么活下来、什么死掉

**【事实】** 一张清单，直接决定使用纪律：

**活下来（跨 compaction）：**
- 所有 workspace bootstrap 文件（`SOUL/USER/AGENTS/TOOLS/MEMORY.md`）—— 靠**从磁盘重载**，不是靠留在历史里
- daily memory logs —— 通过检索可见，不被重新注入
- 任何 agent 在压缩前**写进磁盘**的东西

**死掉（compaction 后不可逆）：**
- 嵌在对话里的指令
- 会话中途的偏好、纠正、决定
- 压缩前分享的图片
- tool 结果与上下文
- 细微差别与具体性（摘要是有损的）

核心原则再强调一遍：

```text
If it's not written to a file, it doesn't exist.
```

> 这张表是 01 篇「bootstrap 靠重载存活」说法的兑现：bootstrap 文件之所以免疫 compaction，正因为它们每次会话从磁盘重新读，压缩摧毁的是「对话历史」，而它们不在对话历史里。

## 4. 人工纪律：自动机制的补充

**【事实】** flush 是自动兜底，但不能全指望它。两条人工实践：

1. **任务切换前主动存**：在换任务前告诉 agent「Save this to MEMORY.md」
2. **策略性手动 `/compact`**：在给新指令**之前**手动压，而不是等溢出

```text
原理: 新指令落在「刚压缩完的、干净的」context 里, 拥有最长寿命。
```

> 这条很反直觉但重要：compaction 不是纯粹的坏事，**主动选时机压缩**反而能让接下来的重要指令有最大「续航」。被动溢出压缩才是最差情况。

## 5. 成本：compaction 与 prompt cache

**【事实】** prompt caching 能把重复 token 的成本降到约原价的 10%。但：

```text
compaction 会让 cache 失效 -> 下一次请求全价重新缓存
```

所以 pruning 的 cache-ttl 模式价值在于：在 compaction 变得必要之前，先裁掉 tool 膨胀，**避免触发 compaction 从而避免 cache 失效**。

这把第 1、2 节串起来了，形成一条成本驱动的链路：

```text
tool 输出膨胀 -> pruning 先裁(不失效 cache)
            -> 还是要满 -> 留 headroom, flush 抢救记忆
            -> maintenance compaction(可控失效)
            ── 全程避免 ──> overflow(最大损失 + 强制 cache 失效)
```

## 6. 小结

```text
compaction 有损, 这是记忆必须落盘的根本原因。
防线分三道:
  1. pruning(cache-ttl)  推迟压缩, 不失效 cache
  2. headroom + flush     压缩前静默抢救进 memory/<today>.md
  3. bootstrap 重载       身份/长期记忆永远从磁盘回来
活下来的只有「落了盘的」和「bootstrap」; 对话里的一切都不保证。
```

下一篇 [05-dreaming-记忆固化.md](./05-dreaming-记忆固化.md)：daily notes 不会自动晋升成长期记忆——dreaming 如何用睡眠隐喻做异步固化。
