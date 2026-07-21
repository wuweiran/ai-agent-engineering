---
layout: default
title: 工具错误与执行控制
parent: Outlook Copilot Agent 能力开发
grand_parent: 工作经历
nav_order: 4
permalink: /docs/career/copilot-capabilities/tool-execution/
---

# 工具错误与执行控制

## 项目边界

附件提取、邮件读取、移动、归档和加旗标工具都是已有 Extension。它们的后端团队负责：

- 权限与资源版本校验；
- 超时和重试；
- Operation ID 与幂等记录；
- 批量执行和部分失败；
- 服务容量、监控和值班。

我的工作是理解这些工具的 Schema 和错误契约，将它们注册到 Declarative Agent，并在 Agent Instructions、Planning 和 Tool Result 处理中规定正确行为。

## 工具错误语义

已有 Extension 将 HTTP 和内部异常转换成稳定 Tool Result：

```json
{
  "status": "failed",
  "error": {
    "code": "version_conflict",
    "retryable": false,
    "message": "12 messages changed after the plan was created."
  },
  "data": {
    "affectedIds": ["msg-1042", "msg-1088"]
  }
}
```

Agent 根据错误语义决定下一步：

| 错误 | Agent 行为 |
| --- | --- |
| `invalid_arguments` | 修正工具参数，仍缺信息时向用户澄清 |
| `permission_denied` | 停止该对象的处理，不尝试其他身份或泄露对象信息 |
| `not_found` | 说明对象已不存在，从当前计划移除 |
| `version_conflict` | 重新读取对象，更新计划并重新确认 |
| `temporarily_unavailable` | 遵循平台与工具的重试结果，超过上限后降级或停止 |
| `result_unknown` | 不生成新的重复写入请求，等待工具查询或恢复结果 |
| `partial_success` | 保留成功项，只解释和处理失败、未知项 |

Instructions 明确禁止模型将错误转成绕过路径。例如 `permission_denied` 后不能改用更宽泛 Query 猜测未授权邮件，`result_unknown` 后不能改变参数再次调用写工具。

## 幂等契约

Tool Call ID 用于平台消息关联；写 Extension 使用 Operation ID 保护业务幂等。Operation ID 由平台或写 Extension 生成并在重试中复用，项目不实现幂等数据库。

我需要保证 Agent 正确使用幂等契约：

- 同一计划的重试保留原 Operation ID；
- 用户改变动作或目标后生成新计划，不复用旧 ID；
- `result_unknown` 时等待查询结果，不让模型创建新 Tool Call 绕过原操作；
- 批量部分成功只处理失败或未知项，不重放整个批次。

Agent 需要遵循 Tool Result 中的 Operation ID 和执行状态，不能通过生成新的写请求绕过原操作；Extension 内部的幂等存储不属于本项目实现范围。

## 资源版本

邮件写 Extension 接受 Message ID 和资源版本。用户确认后，邮件可能已被移动、删除或修改，Extension 会返回 `version_conflict`。

Agent 的正确行为是：

1. 不强制执行旧计划；
2. 重新读取当前邮件状态；
3. 从计划中移除已完成或不存在对象；
4. 动作或范围变化时生成新计划；
5. 重新取得平台确认。

资源版本、用户确认和 Operation ID 解决不同问题。Agent Instructions 和测试需要同时覆盖三者，不能把“已确认”理解成永久允许执行。

## 部分成功

批量邮件工具返回逐项结果：

```json
{
  "status": "partial_success",
  "succeeded": ["msg-1", "msg-2"],
  "failed": [
    {
      "messageId": "msg-3",
      "code": "version_conflict"
    }
  ],
  "unknown": ["msg-4"]
}
```

模型只负责解释结果和决定是否提出新计划：

- 成功项明确告诉用户已经完成；
- 失败项说明原因；
- 未知项不能宣称成功或失败；
- 若要更改失败项动作，重新计划和确认；
- 不重新提交已经成功的邮件。

## 降级与终止

工具不可用时不统一返回“稍后重试”，而是根据任务是否仍能提供可信结果决定降级：

- 搜索工具不可用：当前邮件已有足够证据时只回答当前邮件；跨邮箱任务明确说明无法完成，不伪造候选；
- 附件提取失败：保留已成功提取的附件或页面，并将结果标记为 partial；
- 写工具不可用：可以保留已确认计划和目标范围，但不能宣称已经执行；
- 权限错误：停止对应对象，不用更宽泛搜索猜测其内容；
- 相同错误重复出现且没有新信息：结束当前执行并说明未完成部分。

降级后的完成条件也必须改变。只能提供当前邮件答案时，不能把它表述为跨邮箱完整结果；只能生成计划时，不能把它表述为动作已完成。

## 线上关注点

项目侧关注每个工具的调用量、成功率、稳定错误分布、重复 Tool Call、用户重新确认和部分成功比例，重点判断 Agent 是否正确理解 Tool Result。工具服务内部的幂等命中、存储和资源指标由 Extension 团队负责。

一次候选工具与 Description 设计导致误选的问题见[工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)。工具选择、错误恢复和部分成功的完整离线评测统一进入 [Copilot Evaluation 与 Golden Set]({{ site.baseurl }}/docs/career/copilot-evaluation/)。
