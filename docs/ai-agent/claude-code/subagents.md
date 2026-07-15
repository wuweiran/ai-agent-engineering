---
layout: default
title: Claude Code 子 Agent
parent: Claude Code 的 Agent 实现
grand_parent: AI Agent 工程
nav_order: 4
permalink: /docs/ai-agent/claude-code/subagents/
---

# Claude Code 子 Agent

主 Agent 为了调查登录问题，需要搜索认证、会话和测试目录。若它亲自读取每个候选文件，这些中间内容都会进入主 Context，并在后续每轮重复占用空间。Claude Code 可以把“查清会话超时如何实现”交给子 Agent，让搜索过程留在另一个 Context Window 中。

子 Agent 首先隔离中间信息，也可以让边界清楚的任务并行运行。

## 子 Agent 启动时得到什么

普通子 Agent 每次启动都会创建新的 Context Window，不继承主会话已经累积的消息、读取过的文件和已调用的 Skills。Runtime 为它组装另一套输入：

1. 子 Agent 自己的 System Prompt 和基础环境信息；
2. 主 Agent 写出的任务委派消息；
3. 项目中的 `CLAUDE.md` 和 Rules；
4. 启动时的 Git 状态；
5. 为该 Agent 预加载的 Skills；
6. 它被允许使用的内置工具和 MCP Server。

普通子 Agent 不继承主会话的 Auto Memory。自定义子 Agent 只有配置了自己的持久 Memory，启动时才会加载那份独立记忆。内置 Explore 和 Plan 还会跳过 `CLAUDE.md` 与 Git 状态，以缩小启动 Context。主 Agent 读取它们的结果时仍处在自己的项目规则下；若某条约束对探索本身至关重要，委派消息要明确写出。

子 Agent 文件中的 Markdown 正文会成为它自己的 System Prompt，而不是附加到主 Claude Code 的完整 System Prompt 后面。`tools` 和 `disallowedTools` 可以限制其能力，`model` 可以选择模型，`maxTurns` 可以限制运行轮次，`skills` 则在启动时把指定 Skill 的完整正文直接注入。

因此，创建子 Agent 并不等于复制主 Agent。它更像根据任务重新构造一个缩小的 Agent 系统。

## 委派消息是 Context 接口

子 Agent 看不到主对话，主 Agent 必须在委派消息中提供足够信息。一个含糊请求：

```text
调查一下会话问题。
```

会让子 Agent 重新猜测目标。更有效的委派应说明：

```text
调查刷新 Token 后用户仍收到 401 的原因。
只读取认证和会话相关代码，不做修改。
返回：请求链路、最可能的首次错误位置、文件与行号、仍缺少的证据。
```

这里包含目标、范围、权限和返回契约。子 Agent 可以自己决定搜索路径，但交付结果能够直接支持主 Agent 的下一步。

这个设计对应业务系统中的任务拆分。故障调查主 Agent 可以让一个子 Agent 比较发布记录，让另一个检查错误分布；每个子 Agent 只拿到所需租户和时间范围，最终返回证据与未知项。

## 主 Context 的信息隔离

子 Agent 在自己的窗口里调用 Grep、Read 和 Bash。文件正文、搜索结果与命令输出进入它的消息历史。任务完成后，主 Agent 只收到最终文本和少量运行元数据；这些内容作为 Agent 工具的结果进入主 Context。

于是信息流形成一道明确边界：

```text
主 Agent 的任务描述
        ↓
子 Agent 独立搜索、读取和判断
        ↓
结论、证据与未知项返回主 Context
```

主 Agent 不再承担全部原始材料的 Token 成本，也减少了无关细节干扰后续决策的机会。代价是摘要可能遗漏重要信息，所以返回契约要要求证据位置，而不是只要一句结论。主 Agent 准备修改代码时，仍应读取相关文件的当前版本。

子 Agent 适合“中间过程很大，父任务只需要小型结论”的工作。若主任务随后必须逐字引用所有材料，隔离带来的收益会很小。

## 独立 Context 不等于独立环境

普通子 Agent 从主会话当前工作目录开始，与主 Agent 看到同一份仓库。Context 隔离只隔离模型消息，不自动隔离文件系统。

两个写入型子 Agent 同时修改相同文件会产生冲突。需要并行修改时，可以为子 Agent 配置独立 Git Worktree；只读搜索通常不需要额外环境。这个区别可以概括为：

- Context 隔离解决信息污染；
- 工具权限解决能力边界；
- Worktree 或容器解决执行环境冲突。

三者属于不同层次，不能因为“用了子 Agent”就假定数据和副作用已经隔离。

## 子 Agent 的恢复

每次新调用默认创建一个全新实例。需要在原调查上追问时，Runtime 可以根据 Agent ID 恢复原子 Agent。恢复后，它保留自己的完整会话历史，包括此前工具调用和结果，不必把调查过程重新塞进主 Context。

子 Agent Transcript 独立保存在当前 Session 下。主会话发生 Compact，不会压缩或删除子 Agent 的 Transcript；恢复同一主 Session 后，仍可继续可恢复的子 Agent。子 Agent 自己接近 Context 上限时，也会采用与主会话相似的自动压缩。

这是一种有状态的任务线程。若工作已经结束且后续问题不依赖原始过程，创建新实例通常更干净；若需要延续同一证据链，恢复旧实例能避免重新探索。

## Fork 与普通子 Agent 的差别

普通子 Agent 从新 Context 开始，只接收委派任务。Fork 则继承父会话当时的完整历史，适合从相同背景尝试另一条路径。

继承全部历史会保留更多上下文，也失去了“把大量细节留在父会话之外”的优势。因此，默认应把子 Agent 当作显式任务接口；只有子任务确实需要主会话大量细节时，才考虑 Fork。

## 子 Agent 的适用条件

是否使用子 Agent，可以看两个问题：

- 子任务能否用一句清楚的目标和交付物独立描述？
- 子任务会产生大量主任务以后不再需要的中间信息吗？

两者都成立时，隔离往往有价值，例如扫描多个模块、调研外部资料、比较几种独立候选。只读取一个已知文件、执行一步依赖前一步结果的操作，直接由主 Agent 完成通常更简单。

子 Agent 不是把一项任务随意切成更多“角色”。它通过重新组装 System Prompt、任务消息、工具和 Context，为一个边界明确的子问题建立独立工作区。

## 从子 Agent 设计带走什么

Claude Code 展示了多 Agent 系统中最容易忽略的一点：通信内容决定了系统质量。主 Agent 必须交付足够的任务上下文，子 Agent 必须返回可验证的结论，Runtime 还要隔离消息、约束工具并处理执行环境冲突。

当这些边界清楚时，子 Agent 可以减少主 Context 污染并支持并行；边界含糊时，只会让多个模型重复探索，再把不完整摘要交给彼此。
