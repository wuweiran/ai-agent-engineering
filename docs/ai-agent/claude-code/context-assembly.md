---
layout: default
title: Claude Code 怎样组装 Context
parent: 从 Claude Code 深入 Agent 系统
grand_parent: AI Agent 工程
nav_order: 1
permalink: /docs/ai-agent/claude-code/context-assembly/
---

# Claude Code 怎样组装 Context

用户刚启动 Claude Code，还没有输入任务，模型已经拥有一部分信息：它知道自己有哪些内置工具，当前工作目录和操作系统是什么，仓库处于哪个分支，也可能看到了用户规则、项目规则和以前保存的经验。

用户发送“修复登录测试失败”后，提示又会加入会话历史。Claude 搜索并读取文件，代码和测试输出继续进入历史。**Context 会随任务动态变化，由 Runtime 在每次模型调用时呈现的系统指令、工具定义和消息序列共同组成。**

## 启动时先装入什么

官方 Context Window 文档给出的顺序可以概括为：

| 部分 | 内容 | 加载方式 |
| --- | --- | --- |
| System Prompt | Claude Code 的核心行为、工具使用和输出规则 | 每次会话自动加载，用户通常看不到 |
| Auto Memory | Claude 以前为这个仓库保存的经验 | `MEMORY.md` 前 200 行或 25 KB，取先到者 |
| 环境信息 | 工作目录、平台、Shell、操作系统、Git 仓库与分支状态 | Runtime 在启动时生成 |
| MCP 工具索引 | 已连接服务器的工具名称和服务器说明 | 默认先不加载完整 Schema |
| Skill 索引 | 可由模型调用的 Skill 名称与描述 | 只加载描述，正文按需进入 |
| 持久指令 | 组织、用户、项目和本地 `CLAUDE.md`，以及无路径条件的 Rules | 按作用域和目录顺序拼接 |
| 用户消息 | 当前目标、附件、IDE 选区等 | 用户提交后进入消息历史 |

表格描述的是信息类别。它们在 Claude API 请求中并不都位于同一个字段：核心指令和部分环境信息属于系统侧内容，官方文档明确说明 `CLAUDE.md` 作为 System Prompt 之后的用户消息交给模型；工具的名称、描述和输入 Schema 则通过工具定义提供。

这一区分很重要。`CLAUDE.md` 能影响模型行为，却不是强制策略。真正禁止命令或文件访问，需要权限规则、Hook 或沙箱执行。

## `CLAUDE.md` 怎样沿目录拼接

Claude Code 从当前工作目录向上遍历目录树，查找每一级的 `CLAUDE.md` 和 `CLAUDE.local.md`。找到的内容不会互相覆盖，而是全部拼进 Context。

加载顺序从广到窄：

1. 组织托管的 `CLAUDE.md`；
2. 用户目录中的 `~/.claude/CLAUDE.md`；
3. 从文件系统上层逐级到当前工作目录的项目指令；
4. 同一级中，`CLAUDE.local.md` 排在 `CLAUDE.md` 后面。

越接近当前目录的规则越晚出现，但“晚出现”不等于 Runtime 自动解决冲突。两条指令互相矛盾时，仍然由模型理解，结果可能不稳定。

当前工作目录以下的文件采用惰性加载。Claude 读取 `src/payments/handler.ts` 时，Runtime 才发现并加入该子目录下的 `CLAUDE.md`。`.claude/rules/*.md` 带有 `paths:` 时也采用类似机制：读取的文件匹配 Glob，规则才进入消息历史。

因此，仓库规则有两个层次：

- 每轮都需要的构建命令和全局约束放在根目录；
- 只影响某个模块或文件类型的规则随文件按需加载。

这正是上下文工程中的“稳定信息预加载，局部细节延迟加载”。

`CLAUDE.md` 还支持 `@path` 导入。导入内容在会话启动时展开，递归深度最多四层。拆成多个文件可以改善维护，却不会节省 Context，因为被导入的正文仍会全部进入。

## Memory 为什么只加载一个索引

Auto Memory 保存在 `~/.claude/projects/<project>/memory/`。入口文件是 `MEMORY.md`，可以再链接到 `debugging.md`、`api-conventions.md` 等主题文件。

启动时只读取 `MEMORY.md` 的前 200 行或 25 KB。主题文件不会自动进入 Context，Claude 需要时再用文件工具读取。这使长期经验形成两层结构：

- 小型索引随每次会话进入；
- 大量细节保存在外部文件中，按需取回。

`CLAUDE.md` 和 Auto Memory 都跨会话，但写入者与可信度不同。前者由团队维护，表达项目要求；后者由 Claude 根据工作过程写入，适合保存构建命令、调试经验和用户偏好。两者都只是模型输入，不能替代程序约束。

## Skills 怎样避免一次装入所有方法

如果每个工作流都写进 `CLAUDE.md`，会话一开始就要为大量暂时无关的步骤付出 Context。Skills 使用渐进披露解决这个问题。

普通 Skill 在启动时只把 `name`、`description` 和适用场景加入索引。模型判断当前任务匹配某个 Skill，或用户输入 `/skill-name` 后，Runtime 才把渲染后的 `SKILL.md` 正文作为一条消息加入当前会话。此后正文在多轮中持续存在。

一个 Skill 因此有两个 Context 成本：

1. 每次会话都支付很小的发现成本；
2. 真正使用后才支付完整方法的成本。

设置 `disable-model-invocation: true` 后，连描述也不会预先放入模型，只能由用户显式调用。Skill 引用的详细文档和示例仍可继续放在独立文件中，让 Claude 在需要时读取。

## MCP 工具为什么只先暴露名称

工具定义包含名称、说明和输入 Schema。连接多个 MCP Server 后，完整工具定义可能占据大量 Context，而且工具集合一变化，稳定 Prompt 前缀也会变化。

Claude Code 默认启用 Tool Search：会话开始时只加载 MCP 工具名称和服务器说明，完整 Schema 延迟。模型需要某类能力时先搜索工具，匹配到的定义再进入 Context。只有真正使用到的工具支付完整 Schema 成本。

加载策略可以调整：

- 默认状态下，所有 MCP 工具都延迟发现；
- `ENABLE_TOOL_SEARCH=auto` 在全部 Schema 不超过 Context Window 10% 时直接预加载，否则延迟；
- `ENABLE_TOOL_SEARCH=false` 关闭延迟，启动时加载全部工具；
- 单个服务器可以设置 `alwaysLoad: true`，让它的工具始终可见。

内置文件、搜索和终端工具属于 Claude Code 的基础能力，不通过这套 MCP 延迟流程。Tool Search 主要解决可扩展工具生态的 Context 膨胀。

## 文件和工具结果怎样进入后续调用

第一次模型调用可以抽象为：

```text
System 与环境
+ 工具定义或工具索引
+ CLAUDE.md、Memory 与 Skill 索引
+ 用户任务
```

模型若请求读取 `src/auth.ts`，Runtime 执行 Read，然后把文件内容作为对应 `tool_use` 的 `tool_result` 加入消息历史。第二次调用看到的内容变为：

```text
原有启动 Context
+ 用户任务
+ assistant 的 Read tool_use
+ user 侧的 tool_result（文件内容）
```

模型再请求运行测试，测试输出继续追加。Claude API 使用 `tool_use_id` 将每个结果与请求关联；一次返回多个工具请求时，Runtime 需要执行全部调用，并在同一条结果消息中带回对应结果。

这说明“Claude 看过一个文件”的实际含义是：文件内容已经成为某轮工具结果，并占用后续 Context。终端只显示一行 `Read auth.ts`，模型看到的可能是几千 Token 的正文。

IDE 选区、用户以 `@file` 引用的内容、`!command` 的命令和输出、Hook 返回的 `additionalContext` 也会进入消息历史。PostToolUse Hook 的普通成功输出不会自动全部送给模型；只有文档规定的反馈字段或错误信息进入 Context，这避免了每个自动化脚本都无意污染会话。

## 每轮 Context 都在增长

Claude Code 调用的是无状态的 Messages API。为了保持连续，Runtime 在下一轮重新发送必要的 System、工具定义和消息历史。Prompt Cache 可以复用稳定前缀的计算，但不会把 API 变成有状态服务。

每次 Read、Grep 和测试都会增加历史。过早读取整个仓库，后续每轮都要携带这些内容，直到旧工具输出被清理或对话被压缩。上下文选择因此直接影响质量、延迟与成本。

从 Claude Code 可以提炼出三个可迁移判断：

- 固定规则、能力索引和动态事实应分层加载；
- 大型知识和工具 Schema 应按任务逐步披露；
- 工具返回既是环境反馈，也是后续每轮都要承担的 Context 成本。

业务 Agent 连接订单、日志或知识库时面对同一个问题。系统拥有数据，只说明它可以查询；哪些内容何时进入模型，仍然需要 Runtime 明确设计。
