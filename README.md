# create-dsh-plugin

为 [DeepSeek Harness（DSH）](https://deepseek-harness.github.io/deepseek-harness/) 编写插件的 Agent 技能（skill）。

本技能指导 AI 代理创建、开发、调试、打包 DSH 插件：从最小可加载插件，到带配置、工具、事件、服务的完整插件，再到打包发布为可安装的 bundle。所有内容基于 DSH 官方文档整理，并内置了官方文档原文作为参考材料。

## 这是什么

DSH 插件是一个导出 `apply(ctx)` 函数的 TypeScript 模块，通过 Cordis 的 `Context` 注册工具、事件监听、定时器、服务等能力。本技能让代理无需联网查文档即可按官方约定开发插件，覆盖：

- **插件形态**：函数 / 对象 / 类三种写法，`inject` 依赖注入，`ctx.effect()` 生命周期清理
- **工具开发**：`defineTool` DSL（参数 schema、规范值、render、后台任务、UI 卡片契约）
- **插件配置**：`Config` 类型 + Schemastery schema（默认值、严格校验、设计原则）
- **事件系统**：`emit` / `bail` / `serial` / `waterfall` 四种模式、类型化事件
- **服务与依赖**：`Service` 类、声明合并、服务隔离
- **打包发布**：bundle / profile 概念、`dsh plugin add`、配置层序、Git 安装陷阱

## 目录结构

```
create-dsh-plugin/
├── SKILL.md                    # 技能主指令（工作流 + 代码模板 + 常见坑）
├── evals/
│   └── evals.json              # 评估用例（带断言）
└── references/                 # 官方文档原文（按需加载）
    ├── basic.md                # 第一个插件教程
    ├── tool.md                 # 工具 DSL 基础
    ├── tool-authoring.md       # 工具编写完整契约
    ├── config.md               # 插件配置与 schema 校验
    ├── events.md               # 事件系统
    ├── service.md              # 服务与依赖
    └── publish.md              # 打包与安装
```

## 安装

把本目录复制（或软链）到你的 skills 目录，例如：

```sh
# macOS / Linux
git clone git@github.com:kaijia323/create-dsh-plugin.git ~/.agents/skills/create-dsh-plugin

# Windows
git clone git@github.com:kaijia323/create-dsh-plugin.git %USERPROFILE%\.agents\skills\create-dsh-plugin
```

重启会话后，代理在遇到"创建 DSH 插件 / 给模型加工具 / 写 cordis.yml / 打包 bundle"等请求时会自动加载本技能。

## 使用方式

直接向代理描述需求即可，例如：

- "帮我在项目里创建一个 DSH 插件，给模型加一个 `current_time` 工具"
- "写一个监听 `tools/result` 事件、可配置截断长度的日志插件"
- "把这个插件打包成可安装的 DSH bundle"

代理会按 SKILL.md 中的工作流产出：插件源码 + `cordis.yml` 覆盖层 + 启动/验证命令。

## 文档来源

- 官方文档：<https://deepseek-harness.github.io/deepseek-harness/develop/basic/>
- 源码仓库：<https://github.com/deepseek-ai/deepseek-harness>（`docs/user/develop/**` 与 `docs/cookbook/adding-a-tool.md`）

`references/` 中的文件是上述官方文档对应页面的原文副本（英文版；中文版在同一路径加 `.zh`）。如与在线文档冲突，以在线文档为准。

## 测试与评估

`evals/evals.json` 包含 3 个评估用例（工具插件 / 事件日志插件 / bundle 打包），每个用例都有可程序化检查的断言。

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

技能指令（SKILL.md、evals/）由本仓库作者编写；`references/` 内容版权归 DeepSeek AI 官方文档所有，仅作参考用途转载。
