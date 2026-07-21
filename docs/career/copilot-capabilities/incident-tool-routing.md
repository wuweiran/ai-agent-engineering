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

一次 Instructions 更新把“证据不足时搜索邮件”写进了所有场景的通用规则。进入内部 Ring 后，划词解释也开始调用 `search_outlook_context`：模型已经拿到选区和前后文，仍然多搜索一次邮箱。

这个问题没有造成权限越界，但带来了直接影响：简单请求多了一次 Tool Call 和模型轮次，搜索到的其他邮件会干扰当前选区，Citation 也可能指向错误邮件。平台 Trace 中可以看到，旧版本直接回答，新版本先搜索再回答。

## 定位

排查时选择相同划词解释 Query 对比新旧 Agent 版本。Side Panel Context、模型版本和 Extension 版本没有变化，第一次差异出现在模型首轮决策。

问题由三个简单配置共同造成：

1. 通用 Instructions 中的搜索条件写得太宽；
2. 划词解释场景仍然暴露邮箱搜索工具；
3. 搜索工具 Description 没有说明它不适用于已知选区和当前 Message。

因此问题不在 Insight Extension，也不需要更换模型。

## 修复

Ring 先切回上一版 Agent Definition。新版本做了三项修改：

- 把搜索规则移回开放问答和邮箱整理的 Capability Prompt；
- 从划词解释的工具列表中移除邮箱搜索和写工具；
- 在搜索工具 Description 中补充“用于发现未知邮件，不替代当前选区或已知 Message ID”。

这次修改也减少了无关工具 Schema。划词解释平均输入从 **12,400 Token 降到 7,600 Token**，下降约 **39%**。

## 回归

回归集中增加了三类 Query：当前选区足够时不能搜索；已知 Message ID 时优先读取当前邮件；用户明确要求跨邮箱查找时仍必须搜索。

修复先运行这组工具选择 Query，再跑三个 Capability 的全量 Golden Set。`lm_checklist` 和 Citation 没有下降，工具误选 Query 恢复通过后，版本重新进入 Ring。当前整体 `tool_call` 基线为 **93.8%**。

## 结论

这次问题没有采用继续追加全局禁令的方式解决，而是把规则放回具体场景，并从简单场景中直接移除不需要的工具。对明确入口来说，缩小工具集合比要求模型每次自行判断“不该搜索”更稳定。
