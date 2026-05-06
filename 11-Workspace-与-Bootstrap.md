# OpenClaw Workspace 与 Bootstrap

## 1. 为什么要单独看 Workspace

前面的 context 文档已经分析了 system prompt 如何拼装，但 Workspace 本身还没有被单独展开。

这很容易造成一个误解：

```text
Workspace 只是 agent 的 cwd。
```

官方 Agent Workspace 文档实际表达得更重：workspace 是 agent 的 home，是文件工具和 workspace context 使用的默认工作目录，同时也承载人格、记忆、规则和启动文件。

它既不是普通项目目录，也不是完整安全沙箱。

## 2. Workspace 与 `~/.openclaw/` 的分工

官方文档明确区分：

```text
workspace
  agent 的工作目录、上下文文件、记忆文件、技能目录

~/.openclaw/
  config、credentials、agent state、sessions、managed skills
```

这意味着：
- workspace 更像“agent 能读写和记住的家”
- `~/.openclaw/` 更像“平台自己的状态目录”

不要把两者混在一起理解。  
workspace 可以被备份为私有 Git 仓库，但 `~/.openclaw/credentials`、auth profiles、sessions 等不应该直接提交。

## 3. Workspace 不是硬 sandbox

这是官方文档里非常重要的一点：

```text
workspace 是默认 cwd，不是 hard sandbox。
```

工具的相对路径会以 workspace 为基准解析，但如果没有启用 sandbox，绝对路径仍可能访问宿主机其他位置。

只有启用 sandbox，并且 `workspaceAccess` 不是 `"rw"` 时，工具才会在 `~/.openclaw/sandboxes` 下的 sandbox workspace 中运行。

架构含义：
- workspace 解决“默认上下文在哪里”
- sandbox 解决“能力边界在哪里”
- 二者不是同一个抽象

这也能解释为什么安全文档里要把 agent workspace 和 sandbox 分开讨论。

## 4. 默认路径与 profile

默认 workspace：

```text
~/.openclaw/workspace
```

如果设置了 `OPENCLAW_PROFILE` 且不是 `default`，默认 workspace 会变成：

```text
~/.openclaw/workspace-<profile>
```

这说明 profile 不只是配置文件切换，也会影响 agent 的默认工作区。

## 5. 标准 Workspace 文件

官方 Agent Workspace 文档列出了标准文件。按作用可以整理为：

```text
AGENTS.md
  agent 操作说明、优先级、如何使用 memory

SOUL.md
  persona、语气、边界

USER.md
  用户身份、称呼、偏好

IDENTITY.md
  agent 名称、身份、展示信息

TOOLS.md
  本地工具和约定说明，不直接控制工具可用性

HEARTBEAT.md
  heartbeat run 的小型检查清单

BOOT.md
  Gateway 启动时可由 hook 执行的启动清单

BOOTSTRAP.md
  首次运行仪式文件，只应在新 workspace 中存在

MEMORY.md
  可选的长期 curated memory

memory/YYYY-MM-DD.md
  按日期组织的 memory log

skills/
  workspace-specific skills

canvas/
  节点显示或 Canvas UI 文件
```

这些文件不是简单“用户文档”。  
它们会影响 system prompt、bootstrap context、agent 行为和长期记忆。

## 6. Bootstrap 做什么

官方 Bootstrapping 文档说明，bootstrap 是 first-run ritual。

在首次 agent run 时，OpenClaw 会准备 workspace，并做几件事：
- seed `AGENTS.md`
- seed `BOOTSTRAP.md`
- seed `IDENTITY.md`
- seed `USER.md`
- 运行一轮简短 Q&A
- 将身份和偏好写入 `IDENTITY.md`、`USER.md`、`SOUL.md`
- 完成后移除 `BOOTSTRAP.md`

这说明 bootstrap 不是单纯复制模板，而是“把 agent 的初始人格和用户偏好落到 workspace 文件里”。

## 7. Bootstrap 与 Gateway host

官方文档还指出，bootstrapping 总是在 Gateway host 上运行。

这在远程 Gateway 场景里很关键：
- macOS app 连接远端 Gateway 时，workspace 不在本机 app 目录
- bootstrap 生成的文件在远端 Gateway 所在机器
- 编辑 workspace 文件时，要去 Gateway host 上编辑

因此在多端架构里，workspace 的归属权属于 Gateway host，而不是某个客户端。

## 8. Workspace 文件如何进入 context

从 [09-context-上下文管理](./09-context-上下文管理/README.md) 可知，OpenClaw 会把 workspace bootstrap files 注入 system prompt 的 `Project Context` 或相关 section。

高层链路可以理解为：

```text
workspace files
  -> bootstrap/context resolver
  -> system prompt builder
  -> injected project context
  -> model visible context
```

这解释了为什么 workspace 文件设计得很细：
- `AGENTS.md` 管行为规则
- `SOUL.md` 管人格
- `USER.md` 管用户偏好
- `TOOLS.md` 管工具约定
- `MEMORY.md` 管 curated memory

这些内容稳定、可缓存、可审阅，比每轮动态塞一堆 prompt 更可控。

## 9. 缺失文件与截断

官方文档说明，如果 bootstrap 文件缺失，OpenClaw 会在 session 中注入 missing file marker 并继续。  
大文件会被截断，限制可以通过配置调整。

这背后有两个设计选择：

1. 缺文件不中断 agent run  
   平台更倾向于继续运行，同时把缺失状态显式暴露给 agent。

2. 注入内容有上限  
   workspace 不是无限上下文，仍然要服从 token budget 和 prompt cache 设计。

## 10. Workspace-specific skills

workspace 下可以有 `skills/`。

官方文档提到 workspace-specific skills 拥有较高优先级，可以覆盖 project agent skills、personal agent skills、managed skills、bundled skills 和 extra dirs 中的同名 skill。

这说明 workspace 不只是“上下文文件目录”，还参与能力扩展解析。

分析判断：
- agent workspace 是 persona、memory、tools guidance、skills override 的组合边界
- multi-agent 下，每个 agent 的 workspace 差异会直接影响技能集合和行为风格

## 11. 与 Session Store 的区别

官方文档明确说 sessions 在：

```text
~/.openclaw/agents/<agentId>/sessions/
```

而不是 workspace 里。

这意味着：

```text
workspace
  可人工维护、可审阅、可备份的 agent home

session store
  平台持久化的会话 transcript 与 metadata
```

两者都会影响 context，但职责不同：
- workspace 更偏长期规则和 curated memory
- session store 更偏真实交互历史和运行状态

## 12. 对源码阅读的建议

后续如果拿到完整源码，建议读：

1. `src/agents/system-prompt.ts`
2. `src/agents/system-prompt-params.ts`
3. `src/agents/skills.ts`
4. `src/agents/sandbox.ts`
5. workspace/bootstrap 相关 resolver
6. `agents.files.*` Gateway server methods
7. `docs` 中 Agent Workspace / Bootstrapping / Standing Orders 相关页面

重点问题：
- 哪些文件每轮都会注入，哪些只在特定模式注入
- `BOOTSTRAP.md` 完成后由谁删除
- workspace 文件缺失 marker 的具体格式
- workspace skills 与 managed skills 的优先级在哪里实现
- sandbox workspace copy 如何处理 symlink / hardlink

## 13. 结论

OpenClaw 的 workspace 是 agent 的“可维护长期上下文边界”，不是普通 cwd。

最稳妥的心智模型是：

```text
Workspace =
  default cwd
  + bootstrap files
  + persona/user/tool/memory guidance
  + workspace skills
  + optional canvas assets

Sandbox =
  execution isolation boundary

Session Store =
  transcript and runtime state
```

把这三者分清，才能理解 OpenClaw 为什么同时需要 workspace、session、memory、compaction 和 sandbox。

## 14. 参考

- Agent Workspace：<https://docs.openclaw.ai/concepts/agent-workspace>
- Agent Bootstrapping：<https://docs.openclaw.ai/start/bootstrapping>
- System Prompt：<https://docs.openclaw.ai/concepts/system-prompt>

