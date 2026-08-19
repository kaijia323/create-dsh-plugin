# create-dsh-plugin

为 [DeepSeek Harness（DSH）](https://deepseek-harness.github.io/deepseek-harness/) 编写插件的 Agent 技能（skill）。

本技能指导 AI 代理创建、开发、调试、打包 DSH 插件：从最小可加载插件，到带配置、工具、事件、服务的完整插件，再到打包发布为可安装的 bundle。所有内容基于 DSH 官方文档整理，并内置了官方文档原文作为参考材料。

## 这是什么

DSH 插件是一个导出 `apply(ctx)` 函数的 TypeScript 模块，通过 Cordis 的 `Context` 注册工具、事件监听、定时器、服务等能力。本技能让代理无需联网查文档即可按官方约定开发插件，覆盖：

- **插件形态**：函数 / 对象 / 类三种写法，`inject` 依赖注入，`ctx.effect()` 生命周期清理，Fiber 状态机
- **工具开发**：`defineTool` DSL（参数 schema、规范值、render、后台任务、策略钩子、UI 卡片、Code Mode）
- **插件配置**：`Config` 类型 + Schemastery schema（默认值、严格校验、`!!js` 计算值、设计原则）
- **事件系统**：`emit` / `parallel` / `serial` / `bail` / `waterfall` 五种模式、类型化事件
- **服务与依赖**：`Service` 类、声明合并、服务隔离、能力分层（Definition / Provider / Consumer）
- **LLM 适配器**：`LlmAdapter`、`StreamChunk` 协议、`registerAdapter`
- **客户端 UI 扩展**：`dsh.client` 打包契约、slot 体系（single / list / keyed / chain 与 scope）、slot 查询方法、槽位清单、设置卡片 / 会话节点 / 模态浮层 / 右栏等配方
- **打包发布**：bundle / profile 概念、`dsh plugin add`、配置层序、cmdline 服务、Git 安装陷阱

## 目录结构

```
create-dsh-plugin/
├── SKILL.md                    # 技能主指令（工作流 + 代码模板 + 常见坑）
├── evals/
│   └── evals.json              # 评估用例（带断言）
└── references/                 # 官方文档原文（按需加载）
    ├── basic.md                # 第一个插件教程
    ├── framework.md            # 生命周期、Fiber 状态机、HMR
    ├── tool.md                 # 工具 DSL 基础
    ├── tool-authoring.md       # 工具编写完整契约
    ├── subsystems-tools.md     # 工具注册表 API 与 tools/* 事件签名
    ├── config.md               # 插件配置与 schema 校验
    ├── events.md               # 事件系统（五种模式）
    ├── service.md              # 服务与依赖
    ├── practice.md             # 能力分层三包模式
    ├── llm-adapter.md          # LLM 适配器
    ├── publish.md              # 打包与安装
    ├── cordis-primer.md        # Cordis 核心概念
    ├── extension-cookbook.md   # 钩子/UI/协议驱动扩展形态
    ├── client-ui.md            # 客户端 UI 扩展总纲（slot 体系与槽位清单）
    ├── client-settings-card.md # 官方 cookbook：插件设置卡片
    └── client-conversation-node.md # 官方 cookbook：会话业务行
```

## 安装

把本目录复制（或软链）到你的 skills 目录，例如：

```sh
# macOS / Linux
git clone git@github.com:kaijia323/create-dsh-plugin.git ~/.agents/skills/create-dsh-plugin

# Windows
git clone git@github.com:kaijia323/create-dsh-plugin.git %USERPROFILE%\.agents\skills\create-dsh-plugin
```

重启会话后，代理在遇到"创建 DSH 插件 / 给模型加工具 / 写 cordis.yml / 打包 bundle / 给 DSH 加设置卡片、侧边栏、模态窗或前端页面"等请求时会自动加载本技能。

## 使用方式

直接向代理描述需求即可，例如：

- "帮我在项目里创建一个 DSH 插件，给模型加一个 `current_time` 工具"
- "写一个监听 `tools/result` 事件、可配置截断长度的日志插件"
- "接一个新的模型提供方（LLM 适配器）"
- "在 DSH 设置页里给插件加一张配置卡片"
- "在 Web 客户端加一个右侧栏显示 git log 和文件树，该注册哪个 slot？"
- "把这个插件打包成可安装的 DSH bundle"

代理会按 SKILL.md 中的工作流产出：插件源码 + `cordis.yml` 覆盖层 + 启动/验证命令。

## 文档来源

- 官方文档：<https://deepseek-harness.github.io/deepseek-harness/develop/basic/>
- 源码仓库：<https://github.com/deepseek-ai/deepseek-harness>（`docs/user/develop/**`、`docs/cookbook/**`、`docs/subsystems/*` 等）

`references/` 中的文件是上述官方文档对应页面的原文副本（英文版；中文版在同一路径加 `.zh`）。如与在线文档冲突，以在线文档为准。

## 测试与评估

`evals/evals.json` 包含 9 个评估用例（工具插件 / 事件日志插件 / bundle 打包 / LLM 适配器 / Fiber 诊断 / 权限门禁钩子 / 客户端 slot 查询与右栏 / 设置卡片 / 模态窗兜底），每个用例都有可程序化检查的断言。

### iteration-3 结果（3 个新增 UI 扩展用例 × 24 条断言 × 2 配置 = 48 项检查，新技能 vs 旧版快照）

| 指标 | with_skill（新版） | old_skill（快照） | 差异 |
|---|---|---|---|
| 断言通过率 | 100% (24/24) | 79% (19/24) | **+21%** |

按用例：

| 用例 | with_skill | old_skill |
|---|---|---|
| eval-7 右侧栏 + slot 查询 | 9/9 | 4/9 |
| eval-8 设置卡片打包 | 10/10 | 10/10 |
| eval-9 模态窗兜底 | 5/5 | 5/5 |

要点：

- 核心差距在 eval-7：旧版技能缺少 `ctx.slots.snapshot()` / `spec()` / `inject()` 查询法、`details` 接管代码、`ctx.layout.openDetails()`、React 渲染契约与 DOM hack 理由；新版 `references/client-ui.md` 全部覆盖。
- eval-8 无区分度：基线代理可直接读本机安装的 `dsh-client-ui-settings-plugins` 包类型与 README。
- eval-9 首轮有一条偏窄断言（要求提及 `conversation.input.overlay`），按评审反馈删除后两配置均 5/5。
- 本轮环境未采集 token / 耗时指标（子代理完成通知不携带），故只报告断言通过率。
- 评审页面见同目录的 `create-dsh-plugin-workspace/iteration-1/review.html`（未包含在本仓库中）。

### iteration-2 结果（44 条断言 × 2 配置 = 88 项检查，新技能 vs 旧版快照）

| 指标 | with_skill（新版） | old_skill（快照） | 差异 |
|---|---|---|---|
| 断言通过率 | 100% (44/44) | 100% (44/44) | — |
| 平均 token | 11.9 万 | 13.5 万 | **-12%** |
| 平均耗时 | 125.1s | 179.1s | **-30%** |

要点：

- 断言层面无区分度，但**语义层面有实质差异**（grader 深度复核）：eval-5（Fiber 诊断）旧技能输出含 2 处平台事实错误——声称"DSH 没有 timer 服务"（实际 `@deepseek-ai/cordis-plugin-timer` 存在且 dsh-base 默认挂载）、FiberState 枚举值 UNLOADING/DISPOSED 写反；新技能两者均正确，正是新增 `references/framework.md` 的价值。
- 效率收益集中在新增内容：eval-6（权限门禁，旧技能无 guard 文档）-51% 耗时、-30% token；eval-2（事件日志）-55% 耗时。
- eval-4（LLM 适配器）是唯一持平用例（-11% 耗时、token 持平）：适配器复杂度本身主导成本，两配置都交付了完整实现。
- iteration-2 对应 DSH 0.1.0-rc.7：事件五种模式（含 `parallel`）、Fiber 状态机、LLM 适配器协议、`!!js` 计算配置、`guard`/`restrict`、Code Mode 与 UI 卡片契约。

### iteration-1 结果（24 条断言 × 2 配置 = 48 项检查）

| 指标 | with_skill | without_skill | 差异 |
|---|---|---|---|
| 断言通过率 | 100% (24/24) | 100% (24/24) | — |
| 平均 token | 165.2 万 | 237.1 万 | **-30%** |
| 平均耗时 | 319.6s | 529.3s | **-40%** |

要点：

- 质量上两者打平（基线代理可读取本机已安装的 `@deepseek-ai/*` 包真实类型定义，相当于拿到官方实现的 ground truth）。
- 技能的真实收益在**效率**：内置官方文档省去代理考古类型定义的过程。事件日志用例差距最大（-71% token、-63% 耗时）。
- 评估数据、评分脚本与评审页面见同目录的 `create-dsh-plugin-workspace/`（未包含在本仓库中）。

## 许可证

本仓库代码与技能指令（SKILL.md、evals/）采用 [MIT](LICENSE) 协议。

`references/` 中的文档原文版权归 DeepSeek AI 官方文档所有，仅作参考用途转载。
