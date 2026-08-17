---
name: create-dsh-plugin
description: 创建、开发、调试、打包 DeepSeek Harness（DSH）插件。当用户想要给 DSH 添加新能力——写插件、开发工具（tool）、注册事件监听、提供或消费服务（service）、写 LLM 适配器（adapter）、让插件接受配置、写 cordis.yml / cordis.patch.yml、运行 `dsh web --patch`、`dsh plugin add`、打包发布 bundle——都使用本技能。用户提到 "DSH 插件"、"harness 插件"、"给模型加个工具"、"插件要支持配置"、"监听 xxx 事件"、"扩展 DSH 能力"、"接一个新的模型提供方" 时一定要用；即使没有明说 "插件"，只要目标是扩展 DSH 的能力（新工具、钩子、命令、后台任务、模型适配器），也用本技能。
---

# Create DSH Plugin

指导创建 DeepSeek Harness（DSH）插件：从最小可加载插件，到带配置、工具、事件、服务的完整插件，再到打包发布。全部内容基于官方文档 https://deepseek-harness.github.io/deepseek-harness/develop/basic/ （中文版在同一路径加 .zh，如 .../develop/basic/index.zh.md）。与在线文档冲突时以在线文档为准。

## 插件是什么

DSH 插件是一个导出 `apply(ctx)` 函数的 TypeScript 模块。框架加载插件时调用 `apply`，传入 Cordis 的 `Context`（即 `ctx`），插件通过 `ctx` 注册工具、事件监听、定时器、服务等能力。底层框架是 Cordis（`@deepseek-ai/cordis`）。

最小插件（这些就是完整配置）：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-plugin'

export function apply(ctx: Context) {
  // Register capabilities here.
}
```

## 工作流：把一个插件跑起来

1. 建项目目录：`mkdir -p scratch-plugin/src`（也可以放在现有项目里）。
2. 写插件文件 `src/my-plugin.ts`（函数 / 对象 / 类三种形态，见下）。
3. 写覆盖层 `cordis.yml`，把本地插件插入配置。**插件路径必须是绝对路径**（patch 文件只贡献配置，不改变 loader 解析模块路径的 profile 目录，相对路径会解析失败）：

```yaml
- insert:
    - id: hello
      name: '/absolute/path/to/scratch-plugin/src/my-plugin.ts'
```

4. 启动 Web UI：

```sh
pnpm dsh web --patch ./cordis.yml
```

打开 http://127.0.0.1:3080，启动日志出现 `[hello-plugin] plugin loaded!` 即加载成功（全局安装的 CLI 直接写 `dsh web --patch ...`）。

5. 验证功能：在 UI 里直接向模型提问触发插件能力（工具插件就输入 "Use the greet tool to greet Ada."）。

注意：修改插件代码后要重启 `dsh web`；仅改配置（cordis.yml 里的 `config`）时框架会 HMR 热替换插件实例，无需重启。

## 插件的三种形态

- **函数形式**：`export function apply(ctx)` + 可选 `export const name`。大多数情况够用。
- **对象形式**：

```ts
import type { Context } from '@deepseek-ai/cordis'

export default {
  name: 'my-plugin',
  inject: ['tools'],
  apply(ctx: Context) {
    // ...
  },
}
```

- **类形式**：继承 `Service`，当插件要向其他插件提供服务时使用（见"提供服务"一节）。

## 生命周期与清理

每个已加载的插件实例拥有一个 **Fiber** 作用域，状态机如下：

```
PENDING → LOADING → ACTIVE
                 ↘ FAILED
ACTIVE → UNLOADING → DISPOSED
```

- PENDING：已声明，但所需依赖未就绪（`inject` 的服务缺席时不报错，一直等）。
- LOADING / ACTIVE：`apply` 正在执行 / 已完成。
- FAILED：`apply` 或配置校验抛错。
- UNLOADING / DISPOSED：正在释放资源 / 已完全卸载。

要点：

- 通过 `ctx` 注册的任何东西（事件监听、工具、定时器）在插件卸载时**自动清理**，不需要手动 removeListener / clearInterval。
- 需要显式清理的资源（如网络连接）用 `ctx.effect()` 提供 disposer：

```ts
import type { Context } from '@deepseek-ai/cordis'

export function apply(ctx: Context) {
  ctx.effect(() => {
    const timer = setInterval(() => {
      console.log('heartbeat')
    }, 5000)

    // The returned function runs when the plugin unloads.
    return () => clearInterval(timer)
  })
}
```

- 卸载时 disposer 按注册顺序的**逆序**调用，但多个**异步** disposer 并发执行、无串行保证；有顺序依赖的清理必须放进同一个 `ctx.effect()` 的 disposer 里自己 await。
- 因为注册都是 effect，HMR 替换旧实例时不会残留注册。
- `ctx.plugin(childPlugin)` 创建子 Fiber，继承父上下文但有独立生命周期，随父卸载。
- `fiber.dispose()` 手动提前终止：移除全部注册、递归卸载子插件、等异步清理完成后再 resolve。

## 诊断插件为什么不加载

插件 `inject` 的服务没人提供时，它停在 PENDING 且**不打印任何内容**——这不是报错。遍历注册表检查 fiber 状态：

```ts
import { FiberState, type Context } from '@deepseek-ai/cordis'

export function apply(ctx: Context) {
  setTimeout(() => {
    for (const runtime of ctx.registry.values()) {
      for (const fiber of runtime.fibers) {
        if (fiber.state === FiberState.PENDING) {
          console.log(`${fiber.name} is PENDING — a required service is missing`)
        }
      }
    }
  }, 500)
}
```

## 依赖注入：inject

插件消费 `tools`、`llm`、`agents` 等服务时声明 `inject`，框架保证服务就绪后才调用 `apply`：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-tool-plugin'
export const inject = ['tools']

export function apply(ctx: Context) {
  // ctx.tools is ready here.
  ctx.tools.register(/* ... */)
}
```

依赖语义：

- 必选依赖（`inject` 声明）：服务缺席时插件保持 PENDING 不加载，等待就绪。
- 可选依赖（不 inject）：`const svc = ctx.get('metrics')` 用时查询，可能为 undefined。
- 运行中必选服务消失（如提供者卸载或被 HMR 替换）：依赖插件自动 dispose，服务回来时重新加载。
- `cordis.yml` 可用 `@deepseek-ai/cordis-plugin-group` + `isolate` 让不同插件组看到各自的服务实例（服务隔离）。

## 开发一个工具（最常见需求）

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet someone by name.',
    parameters: {
      name: { type: 'string', required: true, description: 'The name to greet' },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `Hello, ${args.name}!`
    },
  }))
}
```

要点（理解这些"为什么"再写工具）：

- `defineTool` 从 `parameters` 推导 args 类型并在执行前校验（类型、必填、字面量约束、oneOf 等），所以 `execute` 里的 `args` 是强类型且可信的；DSL 表达不了的约束（非空字符串、正数、跨字段规则）仍需自己检查。显式对象节点必须声明 `additionalProperties: true | false`。
- `execute` 只返回 `output.schema` 声明的**规范值**（canonical value，会被序列化为 lossless JSON）；`output.render(args, value)` 负责把它转成模型可见的内容。不要把内容块或给人读的 prose 当返回值。
- 抛错或返回非法值 = 工具失败（isError）。基础设施故障用 throw；业务上的"非理想但成功"结果（如非零退出码）用规范值表达。
- 尊重 `exec.signal`：触发时取消进行中的工作。注册表会给每次调用分配不透明的 `exec.token`，`args` 在策略开始前被冻结，视为只读。
- 长任务用 `ctx.jobs.start({ kind, label, owner: exec.agent, run })` 走后台任务（需先用配置开关 `run_in_background`），不要在前台阻塞。后台分支返回 `{ kind: 'background', jobId }` 类型的规范句柄；发布任务后用任务自己的取消信号，不再用 `exec.signal`。
- 异步通知用 `exec.agent.inject({ content, source: { kind: 'plugin', plugin: '<name>' } })`：追加持久化上下文，下一次模型请求能看到；不是唤醒（对 dispose 的 agent 要 try/catch）。
- 工具名、description 是模型看到的全部说明，描述要写清楚参数语义。
- 部署策略（允许/拒绝/询问、超时、指标）不要内建到工具里：用 `tools/pre-execute`（可重排的 allow/deny/ask waterfall，返回类型化决策）、`ctx.tools.guard()`（最终单调拒绝，后续监听器无法撤销）、`tools/execute`（包裹分发加截止时间/重试）、`tools/post-execute`（替换内容或值）、`tools/result`（只读观察不可变结果）。
- UI 卡片与模型内容分离：可选 `presentCall(args)` / `presentResult(args, result)` 返回 `card` 标签的渲染意图（generic / terminal / diff / search / web），必须纯函数（重放时也会运行）；`output.presentationMeta(args, value)` 从规范值派生可回放的卡片数据。没有展示方法时回退通用卡片。
- Code Mode 下每个可见工具自动可用为 `await tools.<name>(args)`，成功解析为策略处理后的规范 JSON 值，失败以 `ToolCallError` reject。

完整契约（嵌套 schema、后台任务、策略钩子、UI 卡片、Code Mode、`finalizeContent`/`timeoutMs`/`isConcurrencySafe`）见 references/tool-authoring.md 与 references/subsystems-tools.md。

## 插件配置

导出 `Config` 类型 + 同名的 Schemastery schema（默认值写在 schema 字段上），`apply(ctx, config)` 第二参即配置：

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'my-plugin'

export interface Config {
  greeting: string
  maxRetries: number
  verbose?: boolean
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  maxRetries: Schema.number().default(3),
  verbose: Schema.boolean().default(false),
})

export function apply(ctx: Context, config: Config) {
  console.log(config.greeting)  // User value or schema default.
}
```

```yaml
- insert:
    - id: hello
      name: '/abs/path/scratch-plugin/src/my-plugin.ts'
      config:
        greeting: 'Hi there'
        maxRetries: 5
```

设计原则：

- **凡不同部署可能需要不同值的参数，必须做成配置字段**。检验标准：能否在 cordis.yml 里改值而不改代码？硬编码可调参数（如 `const TIMEOUT = 30000`）是错误写法。
- **配置错误要响亮**：用 Schema 表达自身完备的约束（`Schema.string().required()`、`Schema.union([...]).default(...)`），无效配置在插件加载时就失败（fiber 进 FAILED）并给出明确错误。
- **不要导出普通对象作为 `Config`**，它不满足 Cordis 要求的 Standard Schema 接口。
- 对服务或已注册资源的引用不走 schema，走依赖注入。
- 配置项元数据：`id` 给条目稳定身份（loader 靠它区分"修改"与"删除+新增"，没有 id 的条目每次读配置都会重新挂载）；`disabled: true` 保留条目但跳过挂载，改回后连同 PENDING 依赖一起加载。
- 计算型配置值用 `!!js` 标签，只在 `config` 内和条目的 `disabled` 字段生效（如 `apiKey: !!js process.env.MY_API_KEY`、`disabled: !!js process.platform !== 'win32'`）；其余元数据保持字面值。

## 事件系统

- 监听 `ctx.on('event-name', handler)`，广播 `ctx.emit('event-name', payload)`。
- **五种模式**（模式是事件公开契约的一部分，每个 harness 事件在子系统页面标注 `@mode`）：
  - `emit`：同步广播，所有监听者执行，返回值被忽略（不 await）。
  - `parallel`：`await ctx.parallel(name, ...args)` 所有监听器并发运行，一起等待。
  - `serial`：`await ctx.serial(...)` 按注册顺序 await，第一个非 null/false/undefined 的结果停止后续。
  - `bail`：serial 的同步版，`ctx.bail(...)` 遇第一个非 null/false/undefined 同步返回值即停止。
  - `waterfall`：管道，监听器**必须调用 `next()`** 委托下游；不调用即短路（这被设计为拦截/网关手段）。
- 事件名用 `namespace/action` 形式：`agent/step`、`agent/request`、`agent/request-error`、`tools/result`、`session/event` 等。
- `turn/*`、`step/*`、`tool/call`、`tool/result`、`compaction/*` 是**持久化 session-event 类型，不是同名 Cordis 事件**；要观察它们需监听 `session/event` 并检查 `event.type`。
- 类型安全事件用声明合并：

```ts
import '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Events {
    'my-plugin/ready': (payload: { id: string }) => void
  }
}
```

- 监听器也是 effect：插件卸载时自动移除。`ctx.on(name, listener, { prepend: true })` 可在普通注册之前运行；`global: true` 绕过上下文过滤。

## 提供服务

类形式插件继承 `Service` 即向其他插件暴露服务；消费者 `inject: ['metrics']` 后 `ctx.metrics.record(...)`：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    metrics: MetricsService
  }
}

export default class MetricsService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'metrics')  // 'metrics' is the service name.
  }

  record(event: string, value: number) { /* ... */ }
}
```

依赖语义：

- 必选依赖（`inject` 声明）：服务缺席时插件不加载，等待就绪。
- 可选依赖（不 inject）：`const svc = ctx.get('metrics')` 用时查询，可能为 undefined。
- 运行中必选服务消失（如提供者卸载）：依赖插件自动 dispose，服务回来时重新加载。
- `cordis.yml` 可用 `@deepseek-ai/cordis-plugin-group` + `isolate` 让不同插件组看到各自的服务实例（服务隔离）。

### 能力分层（可替换能力的设计模式）

当一项能力需要支持可替换的提供方时（如 Bash 执行），拆成三个角色、放不同包：

- **Service Definition**（如 `dsh-shell`）：定义 Cordis 服务 + 请求/结果类型，抽象类继承 `Service`。
- **Service Provider**（如 `dsh-bash-local`）：继承 Definition 的抽象类实现具体行为。
- **Consumer**（如 `dsh-tool-bash`）：把能力公开为模型可调用的工具，`inject` Definition 定义的服务。

提供方可替换：换 provider 时 Definition 和 Consumer 都不变。不要预防性拆分——只有角色需要独立演进时才拆包，简单工具插件无需拆分。完整的 Bash 三包示例见 references/practice.md。

## LLM 适配器（接入新模型提供方）

继承 `LlmAdapter` 并实现 `stream()`，把 Harness 提供方无关的请求转成具体 API 调用，再把响应转回 Harness 分片：

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'
import { LlmAdapter, type GenerateOptions, type StreamChunk } from '@deepseek-ai/dsh-llm'

class MyAdapter extends LlmAdapter {
  private apiKey: string

  constructor(apiKey: string) {
    super()
    this.apiKey = apiKey
  }

  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    // 1. Convert options.messages to the provider format.
    // 2. Call the streaming API.
    // 3. Convert the response into StreamChunk values.
  }
}

export interface Config {
  apiKey: string
  providers: string[]
}

export const Config: Schema<Config> = Schema.object({
  apiKey: Schema.string().required(),
  providers: Schema.array(Schema.string()).required(),
})

export const name = 'my-llm-adapter'
export const inject = ['llm']

export function apply(ctx: Context, config: Config) {
  const adapter = new MyAdapter(config.apiKey)
  ctx.llm.registerAdapter(config.providers, adapter)
}
```

要点：

- `stream()` 的 `StreamChunk` 协议：每个内容块 `block-start` → `text-delta` / `tool-call-delta` → `block-end`；`usage` 必须在 `finish` 前；`finish` 必须是最后一个分片；`index` 从 0 递增标识块顺序。
- 传输/协议故障抛带稳定 code 的 `LlmError`，不要依赖普通 `Error` 被自动转换；每个 HTTP 请求合并 `attributionHeaders()` 并转发 `options.signal`。
- `registerAdapter(providers, adapter)` 第一个参数是适配器处理的提供方路由列表；`GenerateOptions.provider` 选择适配器，`model` 由适配器拥有。
- 覆写 `resolveModel(provider, model, signal?)` 返回确切的提供方/模型身份与可选元数据；适配器能公布模型选项时覆写 `listModels()`。
- 完整协议与实战参考（`llm-deepseek`、`llm-pi-ai` 两个实现）见 references/llm-adapter.md。

## 打包发布（bundle）

- **bundle**：发布"配置层"的 npm 包，`package.json` 里 `dsh.bundle.patch` 指向 patch 文件。
- **profile**：`$DSH_HOME/profiles/<name>` 目录，描述一次可运行的组合，`dsh.profile.bundles` 为有序列表（`dsh plugin` 自动创建和维护，不要手写）。
- bundle 三件套：

```json
{
  "name": "dsh-hello-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

```yaml
# cordis.patch.yml —— 插件行用包名引用（Node 解析已安装代码），而非相对源码路径
- insert:
    - id: hello
      name: dsh-hello-plugin
```

- 安装：`dsh plugin --profile demo add ./hello-plugin`（向 profile 转发 pnpm；首次使用会初始化 profile 并带上 `@deepseek-ai/dsh-base`）。验证：`dsh --profile demo --dump-config`。移除：`dsh plugin --profile demo remove dsh-hello-plugin`（同时移除依赖和层）。
- 没有 `dsh.bundle` 声明的包也能安装，但只作为普通依赖（`dsh plugin` 打印警告且不激活层）——库包用这种格式。
- **层序**（后层按行覆盖）：① profile 的 bundles 列表顺序 → ② profile 自身 `cordis.patch.yml` → ③ `$DSH_HOME/cordis.patch.yml` → ④ 每个 `--patch` 参数（按 argv 顺序）。
- **patch 覆盖是整行替换**：按 `id` 覆盖时要把该行需要的所有键重写，`config` 整体替换而非深合并。
- 表层 bundle 可以持有自己的命令行：挂载普通提供方插件（`inject = ['cmdlineArgs']`，用 `@deepseek-ai/dsh-cmdline` 的 `parseCmdline` 解析），受参数配置的行 `inject` 该服务并在 `!!js` 配置里读取（`port: !!js ctx.myAppStartup.port ?? 8080`）；`--help` 时提供方不发布服务，这些行不激活。
- Git 安装（`github:you/hello-plugin`）只拿源码不跑 build：作者需提供自包含的 `prepare` 脚本（不依赖 monorepo 上下文），用户需在 profile 的 `pnpm-workspace.yaml` 里 `allowBuilds` 放行——这是执行该包代码的许可，只放行可信源码并 pin commit（`#<sha>`）。
- 不想要构建许可：发布 npm（带 `lib/`）或 `pnpm pack` 出 tarball 分发（`dsh plugin add ./hello-plugin-0.1.0.tgz`）。

## 常见坑速查

1. cordis.yml 的插件路径必须是绝对路径（patch 不改变 loader 的 profile 目录）。
2. `Config` 必须是 Schemastery schema（Standard Schema），不能是普通对象。
3. patch 覆盖行是整体替换 `config`，不是合并——重写全部所需键。
4. 可调参数硬编码是错的；任何跨部署可能不同的值都进配置。
5. 改插件代码要重启 `dsh web`；改 config 可 HMR 热替换。
6. 工具 `execute` 抛错 = isError；成功业务结果用规范值返回，别让调用方从 prose 里解析 id/字段。
7. waterfall 监听器忘调 `next()` 会静默短路管道。
8. 插件不加载也不报错：先查 fiber 状态——多半是 `inject` 的服务没有提供方（PENDING 是合法状态）。
9. `!!js` 只用于 `config` 值和条目的 `disabled` 字段，其他地方是普通字面数据。
10. 后台任务发布后用任务自己的取消信号，不要继续用 `exec.signal`（外层取消只会停止等待，不会杀已发布的任务）。

## 参考文件（官方文档原文，按需读取）

| 文件 | 内容 | 何时读 |
|---|---|---|
| references/basic.md | 第一个插件完整教程 | 首次开发、确认插件形态/注册方式 |
| references/framework.md | Fiber 状态机、生命周期、HMR、嵌套上下文 | 理解插件加载/卸载/热替换时 |
| references/tool.md | 工具 DSL 基础教程 | 写第一个工具 |
| references/tool-authoring.md | 工具完整契约：schema 校验、规范值、后台任务、策略钩子、UI 卡片、Code Mode | 工具需要嵌套参数、后台运行、UI 卡片或策略钩子时 |
| references/subsystems-tools.md | 工具注册表 API：`register`/`restrict`/`guard`/`execute`、`tools/*` 事件签名与 `@mode` | 写策略钩子、守卫、事件监听时 |
| references/config.md | 插件配置与 schema 校验、`!!js` 计算值 | 需要配置时 |
| references/events.md | 事件系统五种模式与类型化事件 | 做事件钩子时 |
| references/service.md | 服务与依赖、隔离 | 提供/消费服务时 |
| references/practice.md | 能力分层：Service Definition / Provider / Consumer 三包模式 | 设计可替换能力时 |
| references/llm-adapter.md | LLM 适配器完整协议与参考实现 | 接入新模型提供方时 |
| references/publish.md | 打包 bundle、profile 安装、层序、cmdline 服务、Git 安装陷阱 | 发布插件时 |
| references/cordis-primer.md | Cordis 核心概念、五种分发模式、Loader 配置 | 想深入理解底层框架时 |
| references/extension-cookbook.md | 扩展插件形态：钩子插件、UI 插件、协议驱动、功能→机制映射 | 做钩子/UI/集成或找扩展点时 |
