# OpenClaw 记忆系统实现详解

这个目录单独维护 OpenClaw **记忆（memory）** 的设计分析。

前面 `09-context-上下文管理` 讲的是「一次 run 内模型看到的上下文如何维护」，`19-system-prompt-实现详解` 讲的是「run 开始前交给模型的系统协议如何拼出来」。本目录补上第三条轴：**跨 run、跨 session、跨 compaction 的长期记忆如何持久化、如何被检索、如何被固化**。

适用范围：
- 基于 2026-06-07 可见的官方文档（`docs.openclaw.ai/concepts/memory`、`/concepts/system-prompt`、`/concepts/dreaming`、`/reference/AGENTS.default`）与社区拆解（memsearch 抽取、masterclass）整理
- 当前工作区不是 OpenClaw 源码本体，源码细节以公开 GitHub 路径与官方文档为准
- 事实与分析推断会分开标注：**【事实】** 来自文档明确说明，**【推断】** 是根据命名与机制做出的判断

## 一句话结论

```text
OpenClaw 的记忆不是一个向量数据库，而是一套「以 Markdown 文件为唯一真相、
向量索引只是加速层、按冷热分层注入、靠 compaction flush 和 dreaming 固化」
的分层记忆系统。它把认知科学里的工作记忆/长期记忆/记忆固化三件事，
分别映射到 daily notes / MEMORY.md / dreaming sweep 上。
```

## 核心心智模型

```text
                  写入路径                          读取路径
                  --------                          --------
对话中产生信息  ->  agent 主动/被动写文件        会话启动:  bootstrap 注入
                       |                                      (MEMORY.md + 今天/昨天 daily)
                       v                                         |
            memory/YYYY-MM-DD.md  (热/working)                   v
                       |                          运行中:  memory_search / memory_get
            compaction flush (临界前抢救)                   (按需检索, 不占常驻预算)
                       |
            dreaming sweep (夜间固化)
                       |
                       v
              MEMORY.md  (冷/long-term, 高信号)
```

三层对应关系（贯穿全篇的主线）：

| 认知科学概念 | OpenClaw 载体 | 注入方式 | 篇章 |
|---|---|---|---|
| 工作记忆 / 情景记忆 | `memory/YYYY-MM-DD.md` 日记 | 仅今天+昨天自动注入，其余按需检索 | 01 / 02 |
| 长期记忆 / 语义记忆 | `MEMORY.md` | 每次会话启动注入（受预算截断） | 01 / 02 |
| 记忆固化（睡眠/海马体回放） | dreaming sweep | 不注入，定时后台运行 | 05 |
| 身份/人格（自我图式） | `SOUL.md` / `USER.md` / `AGENTS.md` | bootstrap 每次重载 | 01 |

## 建议阅读顺序

1. [01-记忆分层与文件地图.md](./01-记忆分层与文件地图.md) —— 有哪些文件，各自存什么，为什么这样分
2. [02-加载-vs-检索.md](./02-加载-vs-检索.md) —— 冷热分层、常驻注入 vs 按需检索、context 预算
3. [03-记忆工具与混合检索.md](./03-记忆工具与混合检索.md) —— `memory_search`/`memory_get`、向量+BM25 混合、分块与去重
4. [04-compaction-与记忆持久化.md](./04-compaction-与记忆持久化.md) —— memory flush、什么活下来、headroom 配置
5. [05-dreaming-记忆固化.md](./05-dreaming-记忆固化.md) —— 睡眠隐喻、三阶段、打分门槛、DREAMS.md
6. [06-理论依据与设计取舍.md](./06-理论依据与设计取舍.md) —— 为什么这样设计：认知科学、RAG、可审计性、分层防御
7. [07-源码路径与参考.md](./07-源码路径与参考.md) —— 继续深挖的源码路径与公开资料

## 三个最值得先记住的判断

- **文件是唯一真相，索引不是。** “If it's not written to a file, it doesn't exist.” 没有隐藏状态——模型只“记得”落盘的东西。这是整套设计的第一原理，决定了它的可审计性和所有取舍。
- **记忆按冷热分层注入，不是全量塞进 prompt。** 长期记忆常驻、当日记忆常驻、历史记忆按需检索。这是在「连续性」和「context 预算」之间的工程折中。
- **固化是异步的、有门槛的。** daily notes 不会自动晋升进 `MEMORY.md`；dreaming 用打分+门槛挑出高信号项再晋升，模仿睡眠期的记忆巩固，目的是保持长期记忆“高信噪比”。

## 快照时间

- 本目录整理时间：2026-06-07
- 后续若拿到完整本地源码（`src/agents/system-prompt.ts` 的 `buildMemorySection`、`memory-core` 插件实现），建议对 02/03 两篇做二次校正。
