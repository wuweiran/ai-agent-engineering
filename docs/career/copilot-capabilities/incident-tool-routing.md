---
layout: default
title: 工具误选与发布回滚
grand_parent: 工作经历
parent: Outlook Copilot Agent 能力开发
nav_order: 7
permalink: /docs/career/copilot-capabilities/incident-tool-routing/
---

# 工具误选与发布回滚

## 问题表现

一次 Agent Instructions 更新把“证据不足时优先搜索邮件”提升为全局规则，原意是改善开放问答和邮箱整理的召回。灰度后，划词解释和当前邮件问答也开始出现不必要的 `search_outlook_context` Tool Call：模型已经拥有选区或当前邮件 Context，仍然扩大到邮箱搜索。

这没有造成权限越界，但带来三个问题：

- 简单请求多了一次搜索和模型轮次，延迟明显上升；
- 搜索结果把其他邮件带入 Context，回答偶尔偏离当前选区；
- Citation 可能指向搜索到的邮件，而不是用户正在阅读的邮件。

线上首先表现为简单场景的工具调用数、完整生成延迟和用户重新提问同时高于原版本。依赖的 Insight Extension 本身成功率和延迟没有异常，因此问题不在工具后端。

## Trace 怎样定位

排查从受影响 Run ID 开始，将灰度版本与上一版本并排比较：

```text
同一类 Side Panel 入口
├─ 旧版：Selection Context → 直接回答
└─ 新版：Selection Context → search_outlook_context → 回答
```

第一次偏离发生在模型首轮决策。两边的选区、当前 Message ID、模型部署和 Extension Version 相同，变化只有 Agent Instructions。新版本的全局搜索规则覆盖了划词解释 Capability Prompt 中“优先使用当前选区”的局部约束。

进一步检查发现，三个因素叠加导致误选：

1. 全局 Instructions 把“证据不足”写得过于宽泛；
2. 划词解释场景仍向模型暴露了不需要的搜索工具；
3. 搜索工具 Description 只写了“查找相关邮件”，没有强调它不能替代当前邮件和选区 Context。

这说明问题不是简单的模型随机性，而是 Prompt 层级、候选工具集合和工具 Description 同时给出了错误信号。

## 怎样止损和修复

灰度阶段先把 Agent Definition 指针回滚到上一版本，避免继续放大。修复没有修改模型、Copilot 平台或 Insight Extension，而是在项目拥有的三层配置中完成：

- 将全局规则收敛为跨场景原则，搜索触发条件下沉到开放问答和邮箱整理的 Capability Prompt；
- 划词解释固定入口不再暴露 `search_outlook_context`，只有当前证据确实不足并转入开放问答时才允许搜索；
- 在搜索工具 Description 中明确“用于从邮箱发现未知邮件，不能替代已知 Message ID、当前选区或精确邮件读取”。

```text
全局规则：保持通用行为边界
Capability Prompt：定义当前任务何时需要外部证据
工具白名单：移除当前场景不需要的候选
Tool Description：说明何时使用和不能替代什么
```

修复后，模型在划词解释场景没有机会误选搜索工具；在开放问答和邮箱整理中，搜索能力仍然保留。

## Evaluation 怎样形成闭环

问题发生在 2025 年 Agent 开发与 Evaluation 工作重叠阶段。受影响的真实问题模式被整理为 Golden Set Query，并增加三类回归：

- 当前选区已经足够时，不应调用邮箱搜索；
- 当前 Message ID 已知且用户问精确事实时，优先读取当前邮件；
- 用户明确要求跨邮箱查找时，必须保留搜索工具。

固定 Query 使用 `tool_call` 检查工具与关键参数，`lm_checklist` 检查回答是否局限于正确 Grounding，Citation Metric 检查引用来源。修复先跑工具误选切片，再跑三个 Capability 的全量回归，确认收紧搜索没有破坏真正需要跨邮箱检索的任务。

发布时一次只切换 Instructions、场景工具白名单和 Description 组成的同一配置版本，在内部 Ring 验证后逐步放量。受影响切片中的 `pass → fail` Query 全部恢复通过，不需要邮箱搜索的场景不再产生搜索 Tool Call；全量 Golden Set 没有出现新的 Capability 回归，简单场景的工具调用和延迟也回到上一版本基线。Trace 中同时记录 Agent Definition、Scenario、Context 和 Extension Version，线上若再次出现工具调用增加，可以直接定位到具体配置版本。

## 这次问题的结论

这次问题说明，统一 Agent 中的工具选择不能只靠一份 System Prompt：

- Prompt 层级决定规则是否会跨场景污染；
- 工具白名单决定模型当前能看到什么能力；
- Description 和 Schema 决定相近工具怎样区分；
- Trace 和 Evaluation 决定误选能否被发现并防止回归。

模型选错工具不一定需要更换模型。先缩小候选集合、消除冲突规则并明确工具边界，通常比继续向全局 Prompt 追加禁令更稳定。
