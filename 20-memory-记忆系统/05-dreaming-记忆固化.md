# dreaming：记忆固化子系统

01 篇说过 daily notes 不会自动晋升进 `MEMORY.md`，需要被「蒸馏」。**dreaming（做梦）就是 OpenClaw 把这个蒸馏自动化的后台引擎**：它定时把短期信号里反复出现、高质量的那部分，固化成长期记忆。

这是整套设计里理论隐喻最重的一块，直接借用了**睡眠期记忆巩固（memory consolidation）**。

## 1. 它解决什么问题

没有 dreaming 时，「daily → MEMORY.md」的蒸馏要靠 agent 在对话中自觉做，或靠用户手动让它整理。问题：
- 不可靠（依赖每次自觉）
- 占用对话 turn（在用户面前花算力整理旧账）
- 容易把一次性噪声也写进长期记忆，污染信噪比

dreaming 把蒸馏变成**异步、有门槛、可审计**的后台过程。**【事实】**：

```text
Dreaming 自动把强短期信号(strong short-term signals)晋升进
持久长期记忆(MEMORY.md), 用睡眠相位隐喻让这个过程可解释、可复核。
```

## 2. 三个相位（睡眠隐喻）

**【事实】** 三个协作相位，对应睡眠科学里的不同阶段：

| 相位 | 类比 | 做什么 |
|---|---|---|
| **Light（浅睡）** | 摄入、去重 | 摄入近期 daily 信号与 recall 痕迹，去重，暂存候选，**不写持久存储** |
| **Deep（深睡）** | 巩固 | 用加权打分 + 门槛闸门给候选排名，把合格条目**追加进 `MEMORY.md`** |
| **REM** | 抽象/反思 | 抽取主题模式与反思，生成「强化信号」反哺后续 deep 排名 |

```text
daily notes + recall traces
      │
   Light: 去重, 暂存候选(不落盘)
      │
   REM:  抽象主题, 产出强化信号 ───┐
      │                            │ 反哺
   Deep: 加权打分 + 门槛 ──────────┘
      │
      v
   只有 Deep 写 MEMORY.md
```

> **只有 deep 相位写 `MEMORY.md`** 是关键约束：长期记忆只有一个写入口，且这个入口前面挂着打分门槛。这保证了 `MEMORY.md` 的「高信号」属性不被随意破坏。

## 3. 打分：六个加权信号

**【事实】** deep 排名用六个基础信号加权（权重之和 = 1.00）：

| 信号 | 权重 | 含义 |
|---|---|---|
| Relevance（相关性）| 0.30 | 该条目的平均检索质量 |
| Frequency（频率）| 0.24 | 累积的短期信号量 |
| Query diversity（查询多样性）| 0.15 | 有多少**不同上下文**召回过它 |
| Recency（新近性）| 0.15 | 时间衰减的新鲜度 |
| Consolidation（巩固度）| 0.10 | 跨多天反复出现的强度 |
| Conceptual richness（概念丰富度）| 0.06 | 概念标签密度 |

候选必须通过三道门槛闸门：`minScore`、`minRecallCount`、`minUniqueQueries`。

**为什么是这六个信号？** 它们共同回答「这条记忆值不值得长期留」：
- **Relevance + Frequency**：被用得多、用得准 → 有用
- **Query diversity + Consolidation**：在不同场景、不同天反复浮现 → 是稳定模式而非一次性噪声
- **Recency**：太旧的低权，避免固化过时信息
- **Conceptual richness**：信息密度

特别是 **query diversity 和 consolidation** 这两个——它们专门过滤「单次高频但本质是噪声」的情况：一件事只在一个上下文里被反复查（diversity 低）、或只在一天内出现（consolidation 低），就算 frequency 高也难过关。这正是认知科学里「分布式练习 / 间隔重复」比「集中重复」更利于长期记忆的工程映射。

## 4. 输出文件

**【事实】**

| 输出 | 性质 |
|---|---|
| `memory/.dreams/` | 机器状态（非人读）|
| `MEMORY.md` | 唯一持久晋升目标（仅 deep 写）|
| `DREAMS.md` | 人读的 Dream Diary 叙事条目 |
| `memory/dreaming/<phase>/YYYY-MM-DD.md` | 可选的各相位报告 |

**【事实】** Dream Diary：每个相位积累够材料后，一个后台 subagent 往 `DREAMS.md` 追加叙事条目——**仅供人读，不具晋升资格**。另有「grounded historical backfill」命令，允许做恢复/回放而不污染实时短期状态。

> 把「机器状态、晋升结果、人读叙事」三种输出分开，是为了**可审计**：人能读 `DREAMS.md` 理解「agent 这几晚想固化什么、为什么」，而不必去看机器状态；同时叙事不能反过来影响晋升，避免「自我强化的幻觉」污染长期记忆。

## 5. 调度与配置

**【事实】**

```json
{ "dreaming": {
  "enabled": true,
  "frequency": "0 3 * * *"
}}
```

- 默认调度 `"0 3 * * *"`——凌晨 3 点跑，字面意义的「夜里做梦」
- 可配 timezone 和 cadence
- 可用 `dreaming.model` 单独覆盖模型（和 flush 一样，把这种后台批处理放到便宜/本地模型上很合理）

> 选「夜间低峰定时」而非「实时」，既是成本考虑（错峰、可用小模型），也呼应隐喻：固化发生在「不工作的时候」，不和前台对话抢资源。

## 6. dreaming 在整个记忆生命周期里的位置

把前四篇串起来，一条记忆的完整生命：

```text
对话中产生
  │ agent 写
  v
memory/<today>.md  ── 当天/次日常驻注入, 之后仅检索可见
  │ 被 memory_search 反复召回(累积 relevance/frequency/diversity 信号)
  v
dreaming deep 打分 + 过门槛
  │ 合格才晋升
  v
MEMORY.md  ── 长期常驻注入, 高信号
  │ 过期项被(agent 或人)删除
  v
退役
```

这条链路把「短期 → 长期」的转换，从「靠模型自觉」升级成「靠使用统计驱动的、有门槛的、异步的固化」。**记忆是否值得长期留，由它实际被用的方式决定，而不是由产生时的一次性判断决定**——这是 dreaming 最核心的设计立场。

下一篇 [06-理论依据与设计取舍.md](./06-理论依据与设计取舍.md)：把散落各篇的「为什么」收拢成一套理论框架。
