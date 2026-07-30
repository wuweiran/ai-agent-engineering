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

排查时选择相同划词解释 Query 对比新旧 Agent 版本。Sydney 注入的 Context、模型版本和 Extension 版本没有变化，第一次差异出现在模型首轮决策。

问题由三处规则共同造成：

1. 通用 `instructions` 中的搜索条件写得太宽；
2. Sydney 已经注入选区和入口类型，但场景规则没有明确优先使用选区；
3. 搜索 Function Description 没有说明它不适用于已知选区和当前 Message。

因此问题不在 Insight Extension，也不需要更换模型。

## 修复

Ring 先切回上一版 Agent Definition。新版本按 `entryType=selection` 增加明确分支：以 `selectedText` 为主要对象，`surroundingParagraphs` 只用于局部语义；除非用户明确要求查找或比较其他邮件，否则不调用搜索。Context Config 删除完整邮件正文、Conversation 历史和附件列表，只保留选区、前后段落和必要邮件元数据；搜索 Function Description 同时补充“不用于解释选区或重新读取已知 Message ID”。具体规则见[一次根据评测完成的优化]({{ site.baseurl }}/docs/career/copilot-capabilities/prompt-context/#evaluation-driven-optimization)。

Context 缩减后，划词解释平均输入从 **12,400 Token 降到 7,600 Token**，下降约 **39%**。

## 回归

回归集中增加了三类 Query：当前选区足够时不能搜索；已知 Message ID 时优先读取当前邮件；用户明确要求跨邮箱查找时仍必须搜索。

修复先运行这组工具选择 Query，再跑三个 Capability 的全量 Golden Set。`lm_checklist` 和 Citation 没有下降，工具误选 Query 恢复通过后，版本重新进入 Ring。当前整体 `tool_call` 基线为 **93.8%**。

## 结论

这次问题没有继续追加模糊的全局禁令，而是利用入口类型把规则放回具体场景，并收紧搜索 Function Description。对明确入口来说，清楚说明 Context 优先级和工具适用边界，比让模型从通用规则中自行推断更稳定。
