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

附件提取、邮件读取、移动、归档和加旗标都是已有 Extension。对应团队负责接口实现、权限检查、资源版本、重试和幂等存储。我负责把工具 Schema 和 Description 接入 Declarative Agent，并规定模型收到 Tool Result 后怎样继续。

项目没有把 HTTP 状态码和内部异常直接交给模型，而是使用 Extension 返回的稳定状态。邮箱整理重点处理三种写入结果：

| 状态 | Agent 处理 |
| --- | --- |
| `version_conflict` | 重新读取发生变化的邮件，更新计划并重新确认 |
| `partial_success` | 保留成功项，只向用户说明和处理剩余项 |
| `result_unknown` | 保留原 Operation ID 查询结果，不创建新的写请求 |

其他常见结果比较直接：参数缺失时修正或澄清，权限不足时停止处理该对象，工具暂时不可用时返回未完成结果。

## 资源版本

邮件写工具接收 Message ID 和资源版本。用户确认之后，邮件仍可能被移动、删除或修改，这时 Extension 返回 `version_conflict`。

Agent 不强制执行旧计划，而是重新读取冲突邮件。已经不存在的对象从计划中移除；动作范围发生变化时，由平台生成新计划并重新确认。用户确认解决“是否同意执行”，资源版本解决“对象是否仍然是确认时的状态”，两者不能相互替代。

## 部分成功

一次批量写入测试包含 12 封目标邮件，其中 9 封移动成功、2 封版本冲突、1 封结果未知。Agent 保留 9 封成功项，只对剩余 3 封继续处理，没有重新提交整个批次。

这条规则很重要，因为重新提交整个批次可能重复处理已经成功的邮件。项目不实现底层幂等数据库，只保证 Agent 不绕过 Extension 提供的 Operation ID 和逐项结果。

## 结果未知

`result_unknown` 通常表示调用超时或响应丢失，不能据此判断操作没有发生。Agent 不改参数重新调用写工具，而是保留原 Operation ID，等待平台或 Extension 查询最终状态。

这样可以避免网络结果不明时重复移动、归档或标记同一封邮件。若状态一直无法确定，任务以“结果待确认”结束，不向用户宣称成功。

## 降级

只读工具不可用时，Agent 只能依据当前邮件回答，并明确不能完成跨邮箱查找；附件只提取成功一部分时，结果标记为 `partial`；写工具不可用时，可以展示计划，但不能说操作已经完成。

相同错误连续出现且没有新信息时停止，不让模型反复调用同一个工具。错误分布和重复 Tool Call 通过平台 Trace 观察，具体服务恢复仍由 Extension 团队负责。

工具选择与执行结果通过 `tool_call` 回归，当前基线为 **93.8%**。完整的邮箱整理恢复链路见[邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)。
