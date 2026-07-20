---
layout: default
title: 多轮对话评测
grand_parent: 工作经历
parent: Copilot Evaluation 与 Golden Set
nav_order: 3
permalink: /docs/career/copilot-evaluation/multi-turn/
---

# 多轮对话评测

## 评测单位

单轮 Query 由 CIQ 和一个 Utterance 组成。多轮任务不能把每一轮当成彼此独立的 Query 执行，因为第二轮经常省略 Current Email、人物或任务目标，依赖同一个 Conversation 中已经建立的 Context。

多轮评测把若干 Turn 组织成一个 Scenario：

```text
Scenario
├─ CIQ 与 Grounding Data
├─ Conversation ID
├─ Turn 1：初始 Utterance
├─ Turn 2：追问、补充或纠正
├─ Turn 3：确认或继续执行
├─ 每轮 Assertion
└─ Scenario 级 Assertion 与停止条件
```

SEVAL 仍然负责 Job 管理。执行器在 Scenario 开始时创建一个 Conversation，后续 Turn 复用同一个 Conversation ID，使 Copilot Runtime 能够保留平台会话、Tool Result 和任务状态。每个 Scenario 结束后销毁会话，不能让不同测试之间共享历史。

## 第一轮怎样执行

第一轮和普通 CIQ Query 相同。输入包括初始 Utterance，以及 CIQ 中的 Current Email ID：

```text
Current Email ID: email-4821
Utterance: 总结这封邮件中的决定和行动项
```

Agent SDK 返回第一轮回答和 Conversation ID。执行器保存 Conversation ID、Agent Result 和 Trace，后续输入只通过同一个会话继续，不重新创建单轮请求。

## 后续用户输入怎样模拟

后续 Utterance 使用两种方式生成。

### 固定脚本

对于确定性回归，Golden Set 直接保存每一轮的 Utterance：

```text
Turn 1：总结这封邮件中的决定和行动项
Turn 2：只保留我负责的事项
Turn 3：截止日期是什么时候？
```

这种方式适合评测：

- Agent 是否记住当前邮件和前一轮结果；
- 代词、省略和“这些事项”等指代是否正确；
- 用户修改输出范围后是否使用新约束；
- 后续追问是否仍然 Grounded 在同一组邮件中。

固定脚本可重复、容易比较版本，也是主回归集的主要方式。但它不能模拟 Agent 主动提问后用户如何回答。

### User Simulator

对于 Agent 可能澄清、要求确认或根据结果选择不同路径的任务，使用 LLM 作为 User Simulator。Simulator 不自由决定用户目标，而是接收一份隐藏的 User Profile：

```text
用户目标：找出当前邮件中分配给自己的行动项
已知信息：用户知道自己的姓名和团队
愿意补充：被问及时可以说明自己是 Alex
不能提供：邮件中不存在的信息
行为要求：简短回答，不替 Agent 完成任务
```

每一轮将 User Profile、原始目标和当前 Conversation Transcript 交给 Simulator，由它生成下一条 Utterance：

```text
User Profile + Transcript + 上一轮 Agent Result
→ User Simulator
→ 下一条 Utterance
→ 同一 Conversation ID
→ Agent SDK
```

如果 Agent 问“你指的是哪位 Alex？”，Simulator 可以根据 Profile 回复；如果 Agent 已经给出最终答案，Simulator 不继续凭空制造新任务。Simulator 只能表达 Profile 中的事实和偏好，不能读取 Assertion、参考结果或 Agent 看不到的 Grounding Data，否则会把答案泄露给被测 Agent。

## 为什么两种方式并用

固定脚本负责稳定回归，User Simulator 负责覆盖动态交互。所有 Scenario 都用 Simulator 会引入额外随机性，使分数变化可能来自模拟用户；全部使用固定脚本又无法测试 Agent 主动澄清和确认后的路径。

主报告以固定脚本为基线。Simulator Scenario 单独切片并重复运行，用通过率而不是单次结果判断版本变化。Simulator 的模型、Prompt 和 User Profile 也固定版本，不能在比较 Agent 版本时同时变更。

## 停止条件

多轮执行必须有确定的终止边界：

- Agent 已经满足 Scenario 的完成条件；
- Agent 明确说明无法完成；
- 用户取消任务；
- 连续两轮没有新增信息或有效进展；
- 达到最多 **6 个用户 Turn**；
- 达到 Job 的时间或 Token 上限；
- 出现严重安全失败。

达到轮次上限仍未完成，Scenario 记为未完成，而不是让 Simulator 无限与 Agent 对话。出现权限泄露或未确认写入则立即终止，并计入 `safety`。

## 怎样评分

多轮评测同时保存 Turn 级和 Scenario 级结果。

### Turn 级

每轮检查：

- Agent 是否正确理解当前 Utterance；
- 回答是否使用正确的历史约束；
- Citation 和 Tool Call 是否正确；
- 是否出现事实错误或安全问题。

### Scenario 级

Conversation 结束后检查：

- 最终用户目标是否完成；
- 第一轮建立的邮件 Context 是否在后续轮次正确延续；
- 用户补充或修改约束后，Agent 是否采用最新值；
- Agent 是否反复询问已经回答过的问题；
- 是否在合理轮次内完成任务。

`lm_checklist` 可以分别运行在单轮 Agent Result 和完整 Transcript 上。单轮 Assertion 检查局部回答，Scenario Assertion 检查跨轮一致性与最终任务结果。例如：

```json
{
  "scenarioAssertions": [
    "最终答案只保留用户本人负责的行动项",
    "后续回答中的截止日期必须与当前邮件一致",
    "Agent 不得再次询问用户已经在第二轮提供的身份信息"
  ]
}
```

## 失败归因

多轮失败从第一次发生偏离的 Turn 开始分析：

```text
初始 CIQ 和邮件 Context 是否正确
→ Conversation ID 是否在后续 Turn 复用
→ Runtime 是否保留必要历史和任务状态
→ Simulator 是否按 User Profile 正确回答
→ Agent 是否采用最新用户约束
→ 最终结果和 Scenario Assertion 是否通过
```

如果 Simulator 生成了 Profile 之外的信息，属于模拟器失败；如果后续 Turn 意外创建了新 Conversation，属于执行器问题；只有会话和输入都正确，而 Agent 忘记目标、误解指代或没有采用新约束时，才属于 Agent 多轮能力回归。
