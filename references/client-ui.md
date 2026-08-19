# 客户端 UI 扩展：往 DSH Web Client 注入前端页面

> 本文基于 DSH **0.1.0-rc.7** 安装包里的 `@deepseek-ai/dsh-client-*` 类型声明与官方 cookbook 整理（本机路径 `$DSH_HOME/profiles/node_modules/@deepseek-ai/`）。slot 清单会随版本变化，升级 DSH 后先用第 3 节的 `ctx.slots.snapshot()` 重新核实，不要盲信旧清单。

## 目录

1. [先选路：两种前端介入方式](#1-先选路两种前端介入方式)
2. [客户端插件的加载与打包契约](#2-客户端插件的加载与打包契约)
3. [Slot 系统心智模型](#3-slot-系统心智模型)
4. [怎么查当前 DSH 有哪些 slot](#4-怎么查当前-dsh-有哪些-slot)
5. [rc.7 槽位清单](#5-rc7-槽位清单)
6. [常用配方](#6-常用配方)
7. [没有现成 slot 怎么办](#7-没有现成-slot-怎么办)
8. [状态管理与框架选择](#8-状态管理与框架选择)
9. [常见坑](#9-常见坑)

---

## 1. 先选路：两种前端介入方式

用户说"给 DSH 插件加前端页面"时，先确认他要的是哪一种，两者机制完全不同：

| | A. 内嵌 Web Client（client plugin） | B. 插件自带独立页面 / 外部 UI |
|---|---|---|
| 表现 | 侧边栏按钮、右侧面板、模态窗、设置卡片、会话里的业务行、新视图 tab | 插件自己起的 web 页面、桌面壳、CLI 前端 |
| 机制 | `dsh.client` 浏览器插件 + **slot 体系**，组件由宿主 React 渲染 | 监听 `session/event` 渲染内容，输入用 `agent.followup()` / `agent.steer()` 驱动（或协议桥） |
| 框架自由 | **不自由**：slot 渲染契约是 React 组件 | 自由，可以用任何前端栈 |
| 本文覆盖 | ✅ 全文 | 见 references/extension-cookbook.md 的 UI plugin / protocol driver 两节 |

本文全部内容针对 A。判断标准：用户提到"在 DSH 里加""设置卡片""会话节点""右侧栏""模态窗""改原生界面"，就是 A；提到"我自己的前端页面""单独的一个面板站点"，多半是 B。

## 2. 客户端插件的加载与打包契约

客户端插件是**同一个包里的 browser half**：宿主半（Host half）在 `src/`，浏览器半在 `src/client/`，导出为 `./client` 并声明 `dsh.client`。官方设置卡片包 `dsh-client-ui-settings-plugins` 就是模板。

```jsonc
{
  "exports": {
    ".": { "types": "./lib/types/index.d.ts", "default": "./lib/index.js" },
    "./client": { "types": "./lib/types/client/index.d.ts", "default": "./lib/client.js" }
  },
  "dsh": {
    "client": {
      "platform": "web",
      // 你的 client 插件 inject 了哪些服务，这里就列提供它们的包。
      // slots/sessions/workspaces 由 dsh-client-runtime 提供；
      // 需要多语言再加 dsh-client-locale，需要连接/远程调用再加 dsh-client-connection。
      "inject": [
        "@deepseek-ai/dsh-client-runtime",
        "@deepseek-ai/dsh-client-locale",
        "@deepseek-ai/dsh-client-connection"
      ]
    }
  },
  "peerDependencies": {
    "react": "^18.2.0",
    "@deepseek-ai/cordis": "^4.0.1",
    "@deepseek-ai/dsh-client-runtime": "^0.1.0-rc.7"
  }
}
```

加载模型（来自 `dsh-client-modules`）：

- Node 侧扫描启用的 Loader 条目，解析每个 `dsh.client` 包的 `exports["./client"]`，把**打包产物**哈希进 boot graph，以 `/plugins` 提供。
- 浏览器侧是 **lazy-CJS 工厂表**：脚本执行时只 `window.__ModuleLoader__.load({id, factory})` 注册工厂；真正 `import` 时才物化。所以 `./client` 必须产出这种 lazy-CJS 格式的 bundle（仓库内是 `tsdown` + 共享 preset；仓库外的包要自己复刻同款产物，见 references/client-settings-card.md 的 Packaging 一节）。
- **宿主提供 React**：`react` 是 peer dependency，bundle 不携带 React 全家桶，这是"插件前端很轻"的真正原因。
- **bundle-purity gate**：插件 bundle 之间禁止 value import（跨插件 import 只能用 type-only）。你的卡片/面板要自带 chrome 和表单逻辑，不能 import 其他 UI 插件的组件。
- value-import `@deepseek-ai/dsh-client-runtime` 必须用 **`/client` 子路径**；裸包名不在 loader externals 表里，会 inline 第二份模块实例，私有 scope-tag Symbol 对不上。

浏览器半的形态与宿主侧插件一致（`export const inject` + `export function apply`），区别是 context 用 `ClientContext`：

```ts
import type { ClientContext } from '@deepseek-ai/dsh-client-runtime/client'

export const name = 'my-ui'
export const inject = ['slots']   // 也可能再加 'locale' / 'connection' / 'settingsScope' 等

export function apply(ctx: ClientContext) {
  ctx.slots.register({ name: 'shell.overlay', id: 'my-panel' }, MyPanel)
}
```

## 3. Slot 系统心智模型

一次 `ctx.slots.register(options, Component)` 同时做四件事：往目标 slot **贡献组件**、可选**声明子 slot**（children）、可选挂 **store seat**、可选带 **inject 业务面**；还能声明 `locale` 拿 `t` 翻译函数。完整类型契约在 `@deepseek-ai/dsh-client-ui-slots/lib/types/index.d.ts`，注册的运行时服务在 `dsh-client-runtime/lib/types/client/slots.d.ts`。

### kind：四种容量模型

| kind | 语义 | 追加 or 替换 |
|---|---|---|
| `single` | 一个 slot 一个赢家。同优先级第二个注册会抛错；不同优先级共存，**数字最小者渲染**。动态注册的条目优先级低于内置条目，所以插件注册进来即赢家 → **整体替换原生区域** | 替换 |
| `list` | 每个条目带唯一 `id`（+`order`/`label`），全部渲染 | 追加 |
| `keyed` | 每个条目带 `key`，按 key 分发。新 key = 追加；已有 key = 替换该 key | 按 key |
| `chain` | 条目带纯函数 `select(owner)`，第一个返回非 null 者当选并收到 `matched` prop；全 null 走 owner 的 fallback | 选举式接管 |

记住这个判据：想"往原生结构里加东西"→ 找 list/keyed；想"把原生那块换成自己的"→ 用 single/chain；**没有任何 slot 都想要自己画一块** → `shell.overlay`（见第 7 节）。

### scope：数据作用域

- `root`：全局组件，标准 props 含 `useSessions` / `useWorkspaces`。
- `session`：严格会话组件，标准 props 含 `sessionId` + `useSession`（会话快照）+ `useProjection`；ui-conversation 再合并 `useInput` / `inputActions`。
- `session-maybe`：跨"无会话→有会话"保持挂载，相关 hooks 返回 `undefined` 直到有当前会话。

### 组件的四份 props（ComposedProps）

组件收到的 props 是四份的交集：运行时（owner 传的 + 标准 kit）、`renderSlot`（你声明的子 slot 的渲染函数）、store（`useStore` selector + 去 draft 的 `actions`）、inject（factory 返回值，其中 `hooks` 舱室会被绑定成 `useXxx` selector hook），外加声明的 `locale` 的 `t`。

### 生命周期与失败语义

- `register` 通过调用者的 fiber 生效：插件卸载自动撤销注册，并**递归塌掉它声明的所有子 slot**。
- 组件渲染崩溃会被边界捕获：shadowing 类 slot（single/keyed/list）里崩溃条目 `abdicate`（弃权），同 cell 下一个条目顶上；`ctx.slots.onEntryError` 可观察。
- slot 变化发 `slots/changed` 事件（`emit` 模式，key 为 payload）。

### 关键规则："声明 = 独占"

slot 由**占用某个父 slot 的条目**在 `children` 里声明；只有声明者有权 `renderSlot` 它。两个规则由此而来：

- 对**未声明**的 slot 直接 `register` 会抛错。
- 替换 single slot 的原生占用者时，它声明的子 slot 也随之失效（比如接管 `details`，内置的 `conversation.details.tool` 就没有渲染者了）。

## 4. 怎么查当前 DSH 有哪些 slot

三层方法，按可靠性排序：

**① 运行时（最权威，跨版本）**：`ctx.slots.snapshot()` 导出当前真实的 slot 树（name/kind/scope/declaredBy/occupants/children）：

```ts
import type { ClientContext } from '@deepseek-ai/dsh-client-runtime/client'
import type { LiveSlotNode } from '@deepseek-ai/dsh-client-ui-slots'

export const name = 'slot-inspector'
export const inject = ['slots']

function dump(nodes: LiveSlotNode[], depth = 0) {
  for (const node of nodes) {
    console.log(
      `${'  '.repeat(depth)}${node.name} [${node.kind}/${node.scope}] ` +
      `declaredBy=${node.declaredBy ?? '-'} ` +
      `occupants=${node.occupants.map(o => o.registrant ?? '?').join('|') || '—'}`,
    )
    dump(node.children, depth + 1)
  }
}

export function apply(ctx: ClientContext) {
  dump(ctx.slots.snapshot())
}
```

**② 类型层（编译期）**：所有槽位都在 `@deepseek-ai/dsh-client-ui-slots` 的 `SlotMap` 接口上声明合并。`keyof SlotMap` 就是全部 slot 名；每个 owner 包的 `lib/types/client/**/slots.d.ts`（或 `index.d.ts`）里有完整 owner props 契约。查某个 slot 的 props，直接读它声明所在包的 d.ts。

**③ 防御性探测/等待**：

- `ctx.slots.spec('some.slot')` 返回 `undefined` = 该版本没声明这个 slot；先探测再注册。
- slot 可能是**别的包运行时才声明**的（声明者没组装进 web bundle 时就不存在）。`ctx.slots.inject('settings.plugin.item', callback)` 声明存在时同步跑 callback、不存在时等待、声明塌掉时 dispose、重新声明时重跑——官方设置卡片就是用这个姿势。callback 返回一个 disposer 或 disposer 数组（generator 可把多个 `register` 做成一个事务）。

## 5. rc.7 槽位清单

### 布局骨架（顶层）

| slot | kind/scope | 用途 |
|---|---|---|
| `root` | single/root | 整个应用根。**永远不要注册**：你会 shadow 掉 AppFrame，整页只剩你的组件 |
| `sidebar` | single/root | 整个左列（替换 = 左列全归你，包括它声明的 workspace/settings 子槽） |
| `conversation` | single/session-maybe | 整个中间会话区 |
| `details` | single/session | **整个右列**（内置是工具详情面板 DetailsPanel，可接管） |
| `shell.overlay` | list/root | **全应用浮层兜底位**：模态窗、toast、浮出面板都放这。整层默认 click-through，条目自己开 pointer-events |

### 侧栏内部（由 sidebar 占用者声明）

| slot | kind/scope | 用途 |
|---|---|---|
| `sidebar.workspaces` | single/root | 工作区/会话浏览区（搜索、列表、对话框） |
| `sidebar.settings` | single/root | 侧栏底部设置入口 |
| `sidebar.footer.action` | list/root | 设置旁的附加动作按钮 ← 给插件加触发器最合适 |
| `sidebar.workspaces.directoryFlow` | single/root | "添加工作区"的选目录交互 |

### 会话区（由 conversation 占用者声明）

| slot | kind/scope | 用途 |
|---|---|---|
| `conversation.session` | single/session | 整个会话主体（接管者还要自管草稿镜像和视图环） |
| `conversation.session.header` | single/session | 会话标题栏整体 |
| `conversation.session.header.actions` | list/session | 标题栏按钮，`order` 排序、负值保留给静态会话上下文 |
| `conversation.session.header.utilities` | list/session | 标题栏右侧工具位 |
| `conversation.view` | list/session | **视图 tab 环**（chat / trajectory 都注册在这；加自己的 tab 最干净） |
| `conversation.chat.node` | keyed/session | 会话流里的一种**持久业务行**，按 `key` 分发（见 6.2） |
| `conversation.chat.commandview` | keyed/session | `/command` 自定义命令行 |
| `conversation.chat.turnTail` | chain/session | 完成轮次的尾部扩展 |
| `conversation.chat.assistant-actions` | list/session | 每条已定稿助手消息上的动作 |
| `conversation.details.tool` | single/session | 右列工具详情面板整体（被 `details` 接管后失效） |
| `conversation.composer` | chain/session | 输入区整体接管链 |
| `conversation.composer.bar` | single/session-maybe | 输入条本体 |
| `conversation.composer.dock` | list/session | 输入条下方环境信息带 |
| `conversation.input.dock` | list/session | 输入卡上方独占整行（队列行、todo 条、goal 条） |
| `conversation.input.left` / `conversation.input.right` | list/session | 输入卡工具行左右加小控件 |
| `conversation.input.overlay` | list/session | 输入条内浮层锚点（菜单/弹层） |
| `conversation.input.plan` / `conversation.input.model` | single/session | plan / model 专属位，未占用则完全不渲染 |
| `conversation.hero.workspace` | single/root | 空白态工作区选择器 |
| `conversation.hero.agentPreset` | single/root | 空白态 agent 预设 chip |
| `conversation.hero.workspace.directoryFlow` | single/root | 空白态选择器的选目录交互 |

### 设置（settings 面板内部）

| slot | kind/scope | 用途 |
|---|---|---|
| `settings.trigger` / `settings.header` / `settings.close` | single/root | 入口行内容 / 面板标题 / 关闭按钮文字 |
| `settings.action` | list/root | 设置面板头部操作按钮 |
| `settings.section` | list/root | **一个完整设置页**（id/order/label 驱动导航） |
| `settings.plugins.tab` | list/root | Plugins 设置区里的 tab 页 |
| `settings.plugin.item` | keyed/root | **插件自己的配置卡片**，key = 你的 settings namespace（见 6.1） |
| `settings.general.item` | list/root | General 区里的单行偏好 |
| `settings.onboarding` | list/root | 引导步骤 |

### 工具

| slot | kind/scope | 用途 |
|---|---|---|
| `tool.call.toolview` | keyed/session | 按工具名 `key: '<tool name>'` 自定义该工具在会话里的渲染；新工具名 = 追加，内置工具名 = 替换 |
| `tool.view.cordis` | keyed/session | cordis_run 卡片内的交互区，动态客户端代码用 `key: 'self'` |

## 6. 常用配方

### 6.1 设置卡片（最规范的入门路径）

一个包两半，用同一个 settings namespace 作 join key：

- **Host half**：`installSettingsSection(ctx, namespace, Config, config, {...})` 注册设置命名空间。
- **Browser half**：`ctx.slots.inject('settings.plugin.item', ...)` 里注册 keyed 条目（`key` = 同一 namespace），组件通过 `ctx.settingsScope.bind({ namespace })` 读写，写操作带 revision 栅栏。

完整代码、`role('secret')`、`applies: 'restart'`、bundle 输出格式见 **references/client-settings-card.md**（官方 cookbook 原文）。

### 6.2 会话业务行（conversation node）

适合"把插件的持久事件折叠成会话里的一行"。要设计可回放的事件族（稳定业务 id + start/progress/end），实现 `ConversationNodeDefinition`（`match`/`start`/`update`/`buildViewNode`），再注册 keyed 渲染器：

```ts
ctx.conversationEvents.register(reviewDefinition)
ctx.slots.inject('conversation.chat.node', () => ctx.slots.register({
  name: 'conversation.chat.node',
  key: 'review-job',
}, ReviewNodeView))
```

完整契约（分页、重放、prepend、publication 节奏）见 **references/client-conversation-node.md**（官方 cookbook 原文）。

### 6.3 模态窗 / 浮出面板：`shell.overlay`

官方为"插件自己的浮层表面"设计的兜底位，list 槽纯追加：

```tsx
import type { PropsRuntime } from '@deepseek-ai/dsh-client-ui-slots'

function GitOverlay(_props: PropsRuntime<'shell.overlay'>) {
  return (
    <div style={{ position: 'fixed', inset: 0, pointerEvents: 'none' }}>
      <div style={{ position: 'absolute', right: 0, top: 0, bottom: 0, width: 320, pointerEvents: 'auto' }}>
        {/* 文件树 / git log / 任何内容 */}
      </div>
    </div>
  )
}

// list 槽必须给唯一 id
ctx.slots.register({ name: 'shell.overlay', id: 'git-panel' }, GitOverlay)
```

要点：宿主层默认 click-through，所以外层 `pointerEvents: 'none'`、内容自己 `pointerEvents: 'auto'`；面板开合、定位、动画全是你自己的状态。

### 6.4 右侧栏（文件树 / git 记录这类）

`details` 是**唯一的右列槽位**，single。注册进去 = 替换内置工具详情面板：

```tsx
import type { PropsRuntime } from '@deepseek-ai/dsh-client-ui-slots'

function GitDetailsPanel({ useSession }: PropsRuntime<'details'>) {
  const session = useSession()
  return <aside>{/* 自绘整个右列 */}</aside>
}

ctx.slots.register({ name: 'details' }, GitDetailsPanel)
```

打开/关闭右列走 `ctx.layout`（`openDetails()` / `closeDetails()` / `toggleSidebar()`），所以通常配一个触发器：

```tsx
ctx.slots.register({
  name: 'conversation.session.header.actions',
  id: 'git-details',
  order: 100,
}, function OpenGitButton() {
  return <button onClick={() => ctx.layout.openDetails()}>Git</button>
})
```

（注入面通过 `inject` factory 传 `ctx.layout`，别在组件闭包直接抓 ctx。）

三种取舍：

| 方案 | 做法 | 代价 |
|---|---|---|
| 真右栏 | 接管 `details` | 内置工具详情面板不再渲染 |
| 不丢原生面板 | `shell.overlay` 自绘右滑面板 + 任意按钮触发 | 自己处理定位/动画 |
| 最 native | 注册 `conversation.view` 新 tab（trajectory 同款，list 加 `id`/`label`） | 是中心区 tab，不是侧栏 |

### 6.5 追加小控件（不碰原生结构）

- 侧栏底部按钮：`sidebar.footer.action`（list，owner 给 `wide`）。
- 标题栏按钮：`conversation.session.header.actions`（list）。
- 输入框工具行：`conversation.input.left` / `.right`（list，owner 给 `{ session, input }` 快照）。
- 自定义工具卡片：`tool.call.toolview`（keyed，`key` = 工具名；owner 给 `callId`/`toolName`/`block`/`cwd`/`openFile`）。

## 7. 没有现成 slot 怎么办

按这个阶梯走，不要一上来就 DOM hack：

1. **换语义，找 additive 位**。想"新增"就找 list/keyed：加按钮、加 tab、加工具卡片几乎都有现成位。
2. **接管上层 single 槽**。官方认可的"侵入原生结构"就是整体替换某个区域：注册进 `details` / `sidebar` / `conversation.session.header` 这类 single 槽，你的组件成为赢家；插件卸载后原生 UI 自动回来。代价是它声明的子槽随之失效，你要自绘整块区域。
3. **`shell.overlay` 兜底**。任何"漂在应用之上的自己一块 UI"都能放，不需要宿主配合，也不影响原生区域。
4. **自己声明子槽**。如果你接管了一个区域，可以在 `register` 的 `children` 里声明自己的子 slot 给其他插件用（声明 = 独占 = 渲染授权）；但不要重复声明已存在的 key（一个 key 一个声明者）。
5. **真的缺 seam → 给 DSH 提 issue/PR**。加一个 slot 只是"某占用者在 children 里多声明一行 + 渲染它"，是架构内低成本改动。
6. **不要 DOM hack**（MutationObserver 找节点、append 到 document.body、改宿主 DOM）。React reconciliation、HMR、版本升级都会弄碎它，且不在 fiber 生命周期里，插件卸载收不干净。

## 8. 状态管理与框架选择

- **渲染层必须跟宿主的 React 契约**：slot 组件返回 React 元素、key 是 React identity（`react` 为 peer，宿主提供）。
- Alpine.js 是处理自有 HTML 指令的运行时，没有组件注册 API，在 React 宿主里只能手动挂 DOM，重渲染即碎——**不适用**。
- Preact 组件要过 `preact/compat` 与宿主 React 互操作，属非官方路径，有风险。
- 推荐组合：**薄 React 视图 + `@preact/signals-core` 状态层**（~1.5 KB gzip）。把 `session/event` 增量推进 signals，React 组件用 `useSyncExternalStore` 订阅；状态逻辑纯 JS、可测试、细粒度更新。状态简单时 React `useState`/`useContext` 足够，不必引 signals。
- "不打包"在这个场景不成立：`./client` 产物本来就要求打包成 lazy-CJS bundle。所以别为了无构建去选 Alpine。

## 9. 常见坑

1. **向未声明的 slot `register` 直接抛错**。slot 由占用者运行时声明，组装缺包时就不存在。先 `spec()` 探测，或 `slots.inject()` 等待。
2. **`root` 永远不要注册**——你会把整页 shadow 成只有你的组件。
3. **single 槽是替换不是追加**。`sidebar`/`conversation`/`details`/`conversation.session.header` 注册进去就是整体接管，且被替换者的子槽声明随渲染一起失效。
4. **list 槽必须给唯一 `id`**（+可选 `order`/`label`）；**keyed 槽必须给 `key`**；**chain 槽必须给纯函数 `select`**。缺了在加载时抛错。
5. **同 cell 同优先级第二个注册会抛错**。默认优先级下，想"替换"某个 list/keyed 条目要给不同 priority；动态注册条目优先级低于内置、且低者赢——这是插件能接管 single 槽的机制。
6. **bundle-purity gate**：客户端插件 bundle 之间禁止 value import（只能 type-only）。自己的卡片自己画，不能 import 其他 UI 插件的组件。
7. **value-import `@deepseek-ai/dsh-client-runtime` 用 `/client` 子路径**，裸包名会 inline 第二份实例，scope 符号对不上。
8. **settings 卡片是两半配对**：Host 注册 namespace，Browser 注册同 namespace 的 keyed 卡片；只做一半，页面上要么没数据要么没卡片。
9. **session 级 slot 的数据走标准 hooks**（`useSession` / `useProjection` / `useInput`），不要在组件里扫描整个会话窗口或相邻节点——那是 O(n) 且不受约束的读取。
10. **slot 清单会随版本变化**。文档里的清单（含本文）只对 rc.7 负责；交付前用 `ctx.slots.snapshot()` 复验目标版本。
