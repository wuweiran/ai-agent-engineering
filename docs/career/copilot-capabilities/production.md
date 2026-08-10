---
layout: default
title: 发布与生产运行
grand_parent: 工作经历
parent: Outlook Copilot Agent 能力开发
nav_order: 6
permalink: /docs/career/copilot-capabilities/production/
---

# 发布与生产运行

## 发布的是什么

项目不部署自己的 Agent Runtime 或 Extension 服务。实际发布的是带版本的 Agent Definition，包括 Declarative Agent Manifest、Instructions、API Plugin 引用和场景规则；Sydney Context Config 随请求发送。发布包由 Microsoft 365 Copilot 平台托管和加载。

已有 Extension 的容量、扩缩容和服务值班由对应团队负责。项目值班主要处理 Agent 配置、Context、工具选择和最终回答的问题。

## 发布前检查

一次发布按下面的顺序进行：

```text
Agent Definition 和 Extension 引用校验
→ 失败 Query 验证
→ 功能切片回归
→ 全量 Golden Set
→ 内部 Ring
→ 逐步扩大范围
```

配置检查只关注实际会影响行为的项目：Agent Manifest 和 Plugin Manifest 能否加载，`instructions` 的场景条件是否正确，Context Config 是否注入必要信息，工具参数是否与 Schema 一致，写操作是否仍受平台确认保护。

完整离线评测在 SEVAL 中完成。任务是否完成由功能对应的 Assertion 定义，再由 `lm_checklist` 对 Agent Result 或完整 Scenario Transcript 评分。发布时按功能切片统计任务通过率，不能用大量简单的划词解释掩盖邮箱整理的失败：

| 场景 | LM Checklist 任务通过率 | 主要 Assertion |
| --- | ---: | --- |
| 划词解释 | 92.5%（74 / 80） | 正确解释选区并使用必要上下文，不误调邮箱搜索 |
| 附件总结 | 89.2%（107 / 120） | 覆盖必需内容，正确处理完整或部分提取结果，并保留有效 Citation |
| 邮箱整理 | 84.7%（61 / 72） | 找对目标邮件和动作，取得确认，并以真实 Tool Result 完成写入 |

`safety` 同样来自 LM Checklist 对禁止项 Assertion 的判断，再单独汇总严重违规率。权限泄露、未确认写入或超范围操作命中一条就阻断发布，不能被整体任务通过率抵消。

其余指标用于解释任务为什么成功或失败，以及成功任务付出了多少代价：

| Metric | 当前基线 |
| --- | ---: |
| `lm_checklist` | 87.6% |
| `citation` | 96.2% |
| `tool_call` | 93.8% |
| `safety` | 0.0% |
| `latency_p95` | 7.4 秒 |

严重安全违规必须保持为零。其他指标除了比较整体基线，还会检查同一 Query 是否从通过变成失败，避免平均值掩盖单个能力回归。

## 版本和回滚

Agent Definition、场景 Prompt 和 Context 配置跟随同一个发布版本，Trace 同时记录当前 Agent、Extension 和模型版本。这样线上出现变化时，可以先判断是本次 Agent 配置、依赖 Extension，还是平台模型发生了变化。

Ring 中出现未确认写入、权限泄露或明显的工具误选时停止放量，并把 Agent Definition 切回上一版本。Extension 故障时可以暂停包含该 Action 的版本或执行降级，但不能修改依赖团队的 Handler。

没有采用固定的 5%、25%、100% 作为所有版本的发布比例。实际放量范围由功能风险、当次回归结果和平台发布安排决定。

## 线上观测
{: #online-observability }

线上业务指标重点看**邮箱整理任务完成率**。它不是根据模型最后是否说“完成”来判断，而是把同一任务中的 Plan、用户确认和 Extension 逐项结果串起来，以 Outlook 写操作的真实状态作为业务终态。

一次任务使用 BizChat Conversation ID、当前 Plan 版本和 Operation ID 关联以下事件：

```text
mailbox_task_started：识别为受支持的移动、归档或加旗标任务
→ plan_presented：生成包含目标 Message ID、动作和 Folder 的计划
→ plan_confirmed / task_cancelled：用户确认或主动取消
→ operation_submitted：写 Extension 接受请求并返回 Operation ID
→ item_result：每封邮件返回 succeeded、failed 或 unknown
→ task_completed：当前确认计划中的全部邮件最终 succeeded
```

只有 `task_completed` 才进入完成数。模型生成成功文案、HTTP 返回 200 或 Tool Call 已发出都不能单独算完成；`partial_success` 要继续关联剩余项，`result_unknown` 要使用原 Operation ID 查到最终状态。用户主动取消和删除、发送等未支持请求单独统计，不进入完成率分母；没有形成可确认计划、确认后没有完成全部操作或 30 分钟内仍未收敛到终态，则记为未完成。

```text
线上任务完成率
= 产生 task_completed 的邮箱整理任务数
÷ 受支持且未主动取消的 mailbox_task_started 数
```

这仍然只能证明系统完成了用户确认的邮件操作，不能完全证明用户主观满意。任务后短时间内重新整理同一批邮件、撤销操作或重复表达同一意图，需要作为补充信号观察。

这条链路再拆成两个阶段观察：

| 阶段 | 初版 | 当前版本 |
| --- | ---: | ---: |
| 到达并确认有效计划 | 69.3% | 81.5% |
| 确认后全部操作完成 | 90.0% | 96.4% |
| 端到端线上任务完成 | 62.4% | 78.6% |

第一阶段主要通过减少无效追问、按需读取候选和在用户修改条件后重建计划提升；第二阶段主要通过处理 `version_conflict`、`partial_success` 和 `result_unknown` 提升。离线邮箱整理 Scenario 通过率从 **68.1% 提升到 84.7%**，用于发布前阻止回归；线上任务完成率才用于判断真实用户任务是否走到业务终态。

Tool Call 错误、确认与计划修改用于定位过程损失，模型轮次、Token 和延迟用于观察运行效率。严重安全违规保持为 **0**，不参与平均分。

底层 CPU、内存、连接池和工具服务可用性属于平台或 Extension 团队。项目只使用这些依赖指标判断问题是否来自工具后端。

## 排查方法

线上问题从 BizChat Conversation ID 开始，依次检查入口和 Sydney Context、Agent 版本、模型第一次决策、Tool Call、Tool Result、平台确认和最终回答。

- 首轮就选错工具，检查入口类型、`instructions` 中的场景条件和 Function Description；
- Tool Call 正确但返回错误，与对应 Extension 团队协同；
- Tool Result 正确但回答遗漏，检查 Prompt 和进入本轮的证据；
- 平台确认行为异常，与 Microsoft 365 Copilot Agent Runtime 团队协同。

Trace 只记录参数摘要、资源 ID、版本、延迟和错误状态，不记录完整邮件、附件、选区或用户 Token。修复后先验证原失败 Query，再跑功能切片和全量 Golden Set，最后重新进入 Ring。
