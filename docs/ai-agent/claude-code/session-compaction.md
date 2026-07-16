---
layout: default
title: Claude Code 长会话
parent: Claude Code 的 Agent 实现
grand_parent: AI Agent 工程
nav_order: 3
permalink: /docs/ai-agent/claude-code/session-compaction/
---

# Claude Code 长会话

Claude Code 已经读取几十个文件、运行多轮测试。终端里还能向上翻到最早的消息，不代表下一次模型调用仍能逐字看到这些内容。模型受 Context Window 限制，Claude Code 还在磁盘上保存完整会话、文件检查点和跨会话记忆。

理解长任务，先要把四种存储分开（[Claude Code 怎样管理当前 Context、会话记录和跨会话记忆？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#claude-code-context-memory)）。

## 四种信息各自解决什么

| 机制 | 保存什么 | 主要用途 |
| --- | --- | --- |
| Context Window | 当前模型调用实际可见的指令、工具定义和消息 | 支持下一步判断 |
| Session Transcript | 用户消息、模型响应、工具调用、结果和元数据 | Resume、回看和审计会话 |
| Checkpoint | Claude 文件编辑工具改动前的文件状态 | 撤销代码修改 |
| Auto Memory | 跨会话仍有价值的项目经验 | 在未来会话中复用 |

Transcript 默认以 JSONL 保存在 `~/.claude/projects/<project>/<session-id>.jsonl`。这是完整会话记录，但其内部格式可能随版本变化。官方建议使用 `/export` 或结构化命令接口，而不是让业务程序依赖 JSONL 字段。

Context Window 是这份记录经过选择后的工作集。`/compact` 会改变模型随后看到什么，不会删除磁盘上的原始 Transcript。反过来，Transcript 中存在一段文件内容，也不保证它仍在当前 Context 中。

## Context 快满时发生什么

随着 Read、Grep 和 Bash 输出不断追加，Context 会接近模型上限。Claude Code 先清理较旧的工具输出；仍然不足时，把对话历史压缩成结构化摘要。用户也可以手动执行 `/compact`，并指定摘要重点，例如：

```text
/compact 保留登录问题的根因、已修改文件和未通过的测试
```

摘要通常保留用户目标、关键技术结论、检查或修改过的文件、重要代码片段、错误与修复、待办和当前工作。逐字文件内容、完整命令输出和中间探索会被替换。

这是一种有损转换。摘要可能记得“`auth.ts` 修改了 Token 轮换顺序”，却不再拥有当时读取的完整函数。模型准备继续编辑时，应重新读取磁盘上的当前文件。

**压缩保留任务语义，外部环境保留可重新取得的细节。**这比尝试把所有历史永久塞在模型窗口中更可靠。

## 压缩后的信息恢复

Claude Code 对不同来源采用不同恢复规则：

| 内容 | `/compact` 之后 |
| --- | --- |
| System Prompt 和 Output Style | 不属于普通消息历史，保持不变 |
| 项目根目录 `CLAUDE.md` 与无路径条件的 Rules | 从磁盘重新注入 |
| Auto Memory | 从磁盘重新注入 |
| 子目录中的 `CLAUDE.md` | 不立即恢复；再次读取该目录文件时加载 |
| 带 `paths:` 的 Rules | 不立即恢复；再次读取匹配文件时加载 |
| 已调用 Skill 的正文 | 按最近使用顺序重新注入；每个最多保留前 5,000 Token，合计最多 25,000 Token |
| 仅用于发现的 Skill 描述列表 | 不重新注入 |
| 旧文件内容和工具输出 | 由摘要保留结论，原文通常不再在 Context 中 |

这里体现了两种持久方式。根目录规则和 Memory 通过重新读取磁盘恢复；临时观察通过摘要保存。需要长期可靠遵守的规则应进入前一种机制，不能只在很早的聊天消息中说一次。

Skill 的重新注入也不是无限的。过长正文会被截断，调用过多 Skill 时较早的内容会被丢弃。因此，最重要的适用条件和约束应该放在 `SKILL.md` 前部，详细材料继续按需读取。

## `/clear`、`/compact` 和 Resume 不一样

三个操作容易混淆：

- `/compact` 保持当前 Session，把历史替换为摘要后继续；
- `/clear` 开始一个空白 Context，之前的会话仍保存并可以恢复；
- `--resume` 或 `/resume` 重新打开某个已保存 Session，在同一 Session ID 下继续追加消息。

Resume 恢复的是会话连续性。项目文件仍以当前磁盘为准；外部编辑、切换分支或依赖变化不会被旧 Transcript 自动回滚。模型需要重新读取容易变化的事实。

`/branch` 或 `--fork-session` 则复制已有会话历史并创建新的 Session ID，适合从同一调查点尝试另一种方案。原会话不会被后续消息修改。

## Checkpoint 与 Git 的边界

Claude Code 在每条用户 Prompt 前建立 Checkpoint，并记录文件编辑工具造成的变化。`/rewind` 可以分别恢复代码、恢复对话，或同时恢复二者。

Checkpoint 只覆盖 Claude Code 文件编辑工具跟踪到的修改。通过 Bash 执行的 `rm`、`mv`、数据库更新、远程 API 调用和部署都无法借此撤销。人工或其他并发进程修改的文件也通常不在当前 Checkpoint 的保护范围内。

因此，Checkpoint 是会话级 Undo，Git 才是长期版本记录。对业务 Agent 来说，这个边界更加重要：退款、上传和消息发送需要业务补偿机制，不能把“回退对话”误认为“回退现实动作”。

## Memory 与摘要承担不同责任

压缩摘要服务当前会话，重点是让正在进行的任务继续。Auto Memory 服务未来会话，只应保存以后仍值得使用的经验。

例如：

- “刚才测试仍有两个用例失败”属于当前状态，应出现在压缩摘要；
- “这个仓库的集成测试需要先启动 Redis”可能值得写入 Memory；
- “用户要求这次不要修改数据库”属于当前任务边界，不应自动变成所有未来任务的偏好；
- 强制执行的禁止规则应放进权限设置或 Hook，不能只依赖二者。

把摘要、Memory 和业务状态混在一起，会产生两个方向的错误：临时结论污染未来任务，或当前任务恢复时缺少准确进度。

## 对长任务设计的启示

Claude Code 没有依靠无限 Context 维持长任务。它组合了活动 Context、完整 Transcript、可重读的文件、结构化摘要、Checkpoint 和跨会话 Memory。

业务 Agent 也需要这种分层。订单记录和平台任务 ID应放在权威数据库；大量日志留在外部系统；当前 Context 只取得本轮判断所需的片段；压缩摘要保存任务目标、证据和未解决问题；经过验证的经验才进入长期记忆。

长任务能否继续，取决于关键信息是否有合适的外部落点。单纯延长模型能够保留的对话轮次，无法替代权威状态和可重新读取的数据。
