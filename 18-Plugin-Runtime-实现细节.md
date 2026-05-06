# OpenClaw Plugin Runtime 实现细节

## 1. 为什么要补 Plugin Runtime

[04-渠道-插件与扩展体系.md](./04-渠道-插件与扩展体系.md) 已经讲了插件体系的架构价值。  
[15-Cron-Hooks-自动化.md](./15-Cron-Hooks-自动化.md) 也讲了 hooks。

这一篇补实现层问题：

- plugin manifest 做什么
- discovery 与 config validation 如何不加载 runtime
- enable/deny/slots 如何决定生效插件
- native plugin 为什么是高信任代码
- plugin registry 记录哪些 surface
- capability registration 为什么比 hook-only 更长期
- channel/provider/runtime helpers 的边界在哪里

说明：
- 当前本地没有完整源码检出
- 本篇基于官方 Plugin Internals、Plugin Manifest、Channel Plugins、Provider Plugins 文档整理
- 具体实现仍建议后续用源码校验

## 2. 插件系统的四层

官方 Plugin Internals 文档把插件系统拆成四层：

```text
1. Manifest + discovery
   找到候选插件，读取 manifest 和 package metadata

2. Enablement + validation
   判断启用、禁用、阻止、exclusive slot

3. Runtime loading
   native plugin in-process 加载并注册能力

4. Surface consumption
   core 从 registry 读取 tools/channels/providers/hooks/routes/CLI/services
```

这条链路的关键是：

```text
先用静态 metadata 建立可验证边界，
再加载 runtime code。
```

## 3. Manifest-first 行为

官方 Manifest 文档说明 `openclaw.plugin.json` 是 OpenClaw 在加载插件代码前读取的 metadata。

它适合放：
- plugin identity
- config validation
- config UI hints
- auth/onboarding/setup metadata
- provider/channel env vars
- static capability ownership snapshots
- QA metadata
- channel-specific config metadata

它不适合放：
- runtime behavior registration
- code entrypoints
- npm install metadata

这些应该在插件代码和 `package.json` 里。

## 4. Manifest 的安全价值

manifest-first 的最大价值不是方便展示插件名称，而是：

```text
不执行插件代码，也能完成 discovery、配置校验和 UI hints。
```

这对安全和稳定性很关键：
- broken manifest 直接让 config validation 失败
- unknown plugin id 可以提前报错
- unknown channel config 可以提前报错
- disabled plugin 的 config 可以保留但给 warning
- setup/onboarding 可以少加载 runtime

如果每次展示插件状态都要执行插件代码，插件系统会很快变得不可控。

## 5. JSON Schema 要求

官方 Manifest 文档要求每个 plugin 都必须提供 JSON Schema，即使没有配置。

空 schema 也可以：

```json
{ "type": "object", "additionalProperties": false }
```

schema 在 config read/write time 校验，而不是 runtime 才校验。

这说明插件配置错误应该在控制面配置阶段暴露，而不是等 agent run 时才炸。

## 6. Validation 行为

官方文档列出几类 validation：

- 未知 `channels.*` key 是错误，除非 channel id 由插件 manifest 声明
- `plugins.entries.<id>`、`plugins.allow`、`plugins.deny`、`plugins.slots.*` 必须引用可发现 plugin id
- broken/missing manifest/schema 会导致 validation 失败
- disabled plugin 的 config 会保留，但 Doctor/logs 给 warning

这让插件系统和 config 系统强绑定：

```text
插件 discovery 结果
  会影响 openclaw.json 是否有效。
```

## 7. Load pipeline

官方 Plugin Internals 文档给出的启动加载管线大致是：

```text
discover candidate plugin roots
  -> read native or compatible bundle manifests and package metadata
  -> reject unsafe candidates
  -> normalize plugin config
  -> decide enablement
  -> load enabled native modules via jiti
  -> call register(api) or legacy activate(api)
  -> collect registrations into plugin registry
  -> expose registry to commands/runtime surfaces
```

这条管线说明：
- unsafe rejection 在 runtime execution 前发生
- enabled native modules 才会 in-process load
- `activate` 只是 `register` 的 legacy alias
- core 后续消费 registry，而不是到处 import plugin module

## 8. Unsafe candidate gates

官方文档提到，候选插件在 runtime execution 前会被安全 gate 拦截：
- entry escaping plugin root
- path world-writable
- non-bundled plugin path ownership suspicious

这说明 OpenClaw 至少在加载前做基本供应链防护。  
但这不是 sandbox。

native plugin 一旦加载，仍然是 Gateway 进程内代码。

## 9. Native plugin trust boundary

官方 Plugin Internals 文档明确指出 native plugin 在 Gateway 进程内运行，不是 sandboxed。

含义非常直接：
- native plugin 可以注册 tools、network handlers、hooks、services
- plugin bug 可以 crash 或 destabilize Gateway
- malicious plugin 等价于 Gateway 进程内任意代码执行

因此：

```text
native plugin 是高信任扩展，
不是低信任脚本。
```

生产环境应该使用 allowlist 和明确 install/load path。  
workspace plugins 更适合开发期、补丁测试和 hotfix，而不是默认生产插件来源。

## 10. Allow、deny、entries、slots

插件 enablement 通常由几类配置共同决定：

```text
plugins.allow
plugins.deny
plugins.entries
plugins.slots.*
plugins.load.paths
```

其中 slots 用于 exclusive plugin kind，例如：
- memory
- context-engine

这说明有些能力不是“多个插件叠加”，而是“选择一个 owner”。  
例如 memory backend 或 context engine 如果多个插件同时生效，状态边界会很难定义。

## 11. Registry model

官方文档说明 loaded plugins 不直接修改任意 core globals，而是注册到 central plugin registry。

registry 跟踪：
- plugin records
- tools
- legacy hooks
- typed hooks
- channels
- providers
- gateway RPC handlers
- HTTP routes
- CLI registrars
- background services
- plugin-owned commands

核心运行时再从 registry 读取这些 surface。

这条边界很重要：

```text
plugin module -> registry registration
core runtime -> registry consumption
```

它避免 core 到处 special-case 某个插件模块。

## 12. Capability model

官方文档把 capabilities 定义为 native plugin 的公共模型。  
常见 capability registration 包括：

- text inference provider
- CLI inference backend
- speech
- realtime transcription
- realtime voice
- media understanding
- image generation
- music generation
- video generation
- web fetch
- web search
- channel / messaging

hook-only plugin 仍然支持，但方向是新 bundled/native plugins 优先使用 explicit capability registration。

这说明 OpenClaw 的插件系统正在从“hooks 扩展”演进到“能力注册”。

## 13. Plugin shape

官方文档提到插件会按实际注册行为分类：

```text
plain-capability
  只注册一种 capability

hybrid-capability
  注册多种 capability

hook-only
  只注册 hooks

non-capability
  注册 tools/commands/services/routes 但没有 capabilities
```

`openclaw plugins inspect <id>` 可用于查看 shape 和 capability breakdown。

这对维护很有帮助。  
因为插件“看起来是 provider”，实际可能还注册了 HTTP route、CLI 和 hooks。

## 14. Capability ownership

官方文档强调 plugin 是 ownership boundary，capability 是 core contract。

例如：
- `openai` plugin 可以同时拥有 text inference、speech、media understanding、image generation
- `google` plugin 可以拥有 model provider、media understanding、web search 等
- `voice-call` 这类 feature plugin 可以消费 shared speech / realtime capability

这套模型避免：
- channel 直接 import vendor implementation
- core 为某个 vendor 写特殊逻辑
- provider-specific helper 变成隐性公共 API

更好的方向是：

```text
core 定义 capability contract
vendor plugin 注册实现
channel/feature plugin 消费 api.runtime helper
```

## 15. Contract enforcement

官方文档提到两层 enforcement：

```text
runtime registration enforcement
  plugin registry 校验 duplicate provider id 等问题

contract tests
  bundled plugins 的 ownership 被测试捕获
```

这说明插件能力不是“谁先注册谁赢”的随意机制。  
至少对于 bundled/native plugin，OpenClaw 试图让 ownership 可声明、可诊断、可测试。

## 16. Channel plugins 与 shared message tool

官方 Plugin Internals 文档明确：channel plugins 不需要注册单独的 send/edit/react tool。  
OpenClaw core 保留一个共享 `message` tool。

边界是：

```text
core
  message tool host
  prompt wiring
  session/thread bookkeeping
  execution dispatch

channel plugin
  scoped action discovery
  capability discovery
  schema fragments
  channel-specific conversation grammar
  final action adapter
```

这能防止每个 channel 都向模型暴露一套不同消息工具。

## 17. Provider plugins 与 runtime hooks

provider plugin 不只是静态 catalog。  
官方 Model Providers 文档列出大量 provider runtime hooks，例如：
- normalize model/config/transport
- resolve API key
- dynamic model resolution
- tool schema normalization
- reasoning output mode
- stream function creation/wrapping
- OAuth refresh
- usage snapshot
- failover reason classification

这说明 provider plugin 是运行时热路径的一部分。  
它会影响每次 model call，而不只是配置页面。

## 18. CLI loading 分两阶段

官方 Plugin Internals 文档提到 plugin CLI root command discovery 分为：

```text
parse-time metadata
  来自 registerCli descriptors

real CLI module
  首次调用时 lazy register
```

这样做的目的：
- root help 可以展示插件命令
- parser 可以提前保留 root command name
- 不必为了显示 help 加载完整 plugin runtime

这和 manifest-first 是同一设计方向：  
能用轻 metadata 解决的问题，不要执行完整 runtime。

## 19. Gateway RPC 与 HTTP routes

插件可以注册 Gateway RPC handlers 和 HTTP routes。  
这使插件不仅能扩展 agent 能力，也能扩展控制面。

风险也更高：
- RPC method scope 是否明确
- HTTP route 鉴权由谁负责
- 是否使用 plugin-specific prefix
- 是否碰到 core admin namespace
- route handler 是否可能阻塞 Gateway

官方 Channel Plugin 文档提醒，如果 `registerFull` 注册 Gateway RPC methods，应使用 plugin-specific prefix。  
核心 admin namespace 如 `config.*`、`exec.approvals.*`、`wizard.*`、`update.*` 保持 reserved。

## 20. Loader caches

官方文档说明 OpenClaw 保留短期 in-process caches：
- discovery results
- manifest registry data
- loaded plugin registries

这些是性能缓存，不是持久化。  
可以通过环境变量禁用或调整窗口，例如：
- `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE`
- `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE`
- `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS`
- `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`

这对调试插件加载很重要。  
否则改了 manifest 但看到旧结果，容易误判。

## 21. 与 Config / Doctor 的关系

插件系统和 config/doctor 强相关：
- broken manifest 会让 config validation 失败
- disabled plugin config 会被保留但 warning
- `openclaw doctor` 报 plugin error
- `openclaw plugins inspect` 展示 shape/capabilities
- `openclaw status --all` 也会展示 compatibility signals

这说明 plugin runtime 不是隐藏在运行时的黑盒。  
OpenClaw 把插件健康状态暴露到运维控制面。

## 22. 常见风险

### 22.1 把 native plugin 当低风险脚本

native plugin 是 Gateway 进程内代码，恶意插件等价于进程内任意代码执行。

### 22.2 Manifest 与 runtime 漂移

manifest 声明的能力、env vars、config schema 与 runtime 注册不一致，会导致 UI、Doctor、实际运行三者不一致。

### 22.3 插件 helper export 过宽

channel 或 core 直接 import vendor-specific helper，会破坏 capability boundary。

### 22.4 Duplicate ownership

多个插件注册同一个 provider/channel/capability id，会造成不可预测行为。  
registry 应该诊断并拒绝。

### 22.5 Hook-only 无限扩张

所有扩展都靠 hooks，会让主链路变得不可解释。  
长期应尽量沉淀成 typed capability。

## 23. 对源码阅读的建议

拿到完整源码后建议读：

1. `src/plugins/*`
2. `src/plugin-sdk/*`
3. plugin discovery loader
4. manifest parser / schema validator
5. plugin registry 实现
6. capability registration API
7. Gateway RPC / HTTP route registration
8. plugin CLI descriptor loading
9. bundled provider plugin
10. bundled channel plugin
11. Doctor plugin diagnostics
12. plugin config schema

重点问题：
- unsafe candidate rejection 的具体规则
- allow/deny/entries/slots 的优先级
- runtime registry 是否 immutable after startup
- duplicate provider/channel id 如何报错
- hook order 与 capability registration 是否分开管理
- plugin disable 后 registry 是否会清理
- manifest cache 失效策略
- setup-time runtime hook 与 full runtime hook 是否隔离

## 24. 结论

OpenClaw 的 Plugin Runtime 可以概括为：

```text
manifest-first discovery
  -> config validation
  -> enablement decision
  -> native in-process loading
  -> register(api)
  -> central registry
  -> core surfaces consume registry
```

真正要记住的是：

```text
plugin 是 ownership boundary，
capability 是 core contract，
registry 是运行时集成点。
```

这层守住以后，OpenClaw 才能在不把 core 写成一堆 vendor/channel special-case 的前提下继续扩展。

## 25. 参考

- Plugin Internals：<https://docs.openclaw.ai/plugins/architecture>
- Plugin Manifest：<https://docs.openclaw.ai/plugins/manifest>
- Building Channel Plugins：<https://docs.openclaw.ai/plugins/sdk-channel-plugins>
- Model Providers：<https://docs.openclaw.ai/concepts/model-providers>
- Plugin hooks：<https://docs.openclaw.ai/plugins/hooks>

