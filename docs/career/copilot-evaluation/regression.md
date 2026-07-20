---
layout: default
title: 回归判断与问题定位
grand_parent: 工作经历
parent: Copilot Evaluation 与 Golden Set
nav_order: 4
permalink: /docs/career/copilot-evaluation/regression/
---

# 回归判断与问题定位

## SEVAL 怎样执行回归

SEVAL 是统一的 AI 评测中台，负责 Evaluation Job 的创建、排队、调度、状态和结果管理。项目团队向 Job 提供业务评测资产和待测版本：

```text
Evaluation Job
├─ Agent、Model、Instructions 与配置版本
├─ Golden Set Version
├─ Grounding Data Version
├─ Assertion Version
├─ Metric 集合
└─ 单轮或多轮执行配置
```

SEVAL 负责稳定地执行和汇总结果，但不定义邮件任务的成功标准。Golden Set、Grounding Data 和 Assertion 由项目维护，Metric 的业务含义和失败归因也由项目解释。Job 成功只表示执行完成，不等于 Agent 质量通过；Runtime 超时、数据缺失和 Metric 失败也分别记录，不能统一算作 Agent 零分。

## 怎样确定一个功能没有回归

“没有回归”不是候选版本在 SEVAL 中成功跑完，也不是总平均分高于旧版本。判断必须建立在受控的配对比较上：基线版本和候选版本使用相同的 Golden Set Query、Grounding Data、Assertion、Metric、Agent 执行配置和 User Simulator 版本，唯一变化是本次要发布的模型、Prompt、Context 或工具配置。

```text
相同评测资产与执行配置
├─ Baseline Job：当前生产版本
└─ Candidate Job：候选版本
             ↓
按同一 Query 配对比较
             ↓
功能切片 + 全量 Golden Set + 安全门禁
```

一次功能发布分两层运行：

1. **功能切片**：选择与该功能直接相关的 Query 和多轮 Scenario，验证预期修复以及本功能的主要路径；
2. **全量 Golden Set**：覆盖共享 Prompt、Context、Runtime 和工具可能影响的其他功能，防止局部修改造成跨功能退化。

只跑功能开发者新增的测试不能说明无回归。一个附件总结 Prompt 的修改可能改变通用 Instructions，邮箱整理工具的 Schema 更新也可能影响模型对其他工具的选择，所以功能切片通过后还要跑全量集。

## 质量门禁

候选版本需要同时满足绝对门禁和相对门禁。

### 绝对门禁

按照当前基线：

- `safety` 必须保持 **0.0%**；
- `lm_checklist` 不低于 **87.6%**；
- `citation` 不低于 **96.2%**；
- `tool_call` 不低于 **93.8%**；
- `latency_p95` 不超过 **7.4 秒**。

绝对门禁防止两个版本一起处于不可接受水平。安全失败不参与平均，出现一条就阻断发布。

### 相对门禁

即使候选版本仍高于绝对线，也不能相对生产版本明显退化。配对结果按 Query 分成：

```text
pass → pass：保持通过
fail → pass：修复
pass → fail：新增回归
fail → fail：遗留失败
```

重点查看 `pass → fail`。候选版本需要满足：

- 功能切片中不能出现未解释的新增失败；
- 全量 `lm_checklist` 相对基线下降不超过 **1 个百分点**；
- `citation` 和 `tool_call` 下降不超过 **0.5 个百分点**；
- `latency_p95` 增长不超过 **10%**；
- `safety` 不能新增任何失败。

总分相同也不一定通过。例如候选版本修复了十条简单 Query，却让一条高风险写操作从 pass 变成 fail，平均分可能变高，但仍然要阻断。

## 非确定性怎样处理

Agent 输出存在波动。新出现的 `pass → fail` Query 不能只运行一次就直接归因于版本，也不能通过反复重跑直到成功来消除失败。

对发生变化的 Query 使用基线和候选版本各重复 **5 次**：

```text
Query Pass Rate
= 通过 Run 数 / 5
```

如果候选版本稳定下降，再认定为回归；如果两边都在通过和失败之间波动，则标记为不稳定 Query，单独分析 Assertion、模型随机性和场景设计。User Simulator 多轮 Scenario 同样固定 Simulator 版本并重复执行，避免把模拟用户的差异归因于 Agent。

## 有问题时怎样定位

定位从失败 Query 开始，而不是先猜 Prompt。先完成有效性检查，再比较基线和候选版本的 Trace。

### 第一步：确认评测本身有效

先排除三类非 Agent 问题：

- **Job 问题**：超时、运行失败、候选版本没有正确加载；
- **数据问题**：CIQ 指向错误 Email、Grounding Data 缺失或邮件内容已变化；
- **Metric 问题**：Assertion 与邮件证据不一致，或者 LM Checklist 明显误判。

如果基线和候选版本都取得了错误邮件，修改 Agent 没有意义；如果 Agent Result 正确而 Metric 判错，应修复 Assertion 或 Metric。

### 第二步：做 Query 级配对

对 `pass → fail` Query 并排查看：

```text
Baseline
├─ Agent / Model / Prompt / Context / Tool Version
├─ CIQ 与 Runtime Context
├─ Model Turns
├─ Tool Calls 与 Tool Results
└─ Final Result / Metric

Candidate
└─ 同一组 Trace 节点
```

这一步先确认两个 Job 除候选变量外完全一致，再找输出第一次分叉的位置。

### 第三步：沿 Trace 找第一次偏离

排查顺序是：

```text
1. Query 输入是否一致
2. CIQ 是否传入同一 Current Email ID
3. Runtime 是否取得同一邮件 Context
4. Instructions、模型和工具 Schema 版本是否正确
5. 模型第一次决策是否发生变化
6. Tool Call 的工具和参数是否变化
7. Tool Result 是否成功、完整且版本一致
8. 最终回答如何使用证据
9. Metric 是否正确解释结果
```

第一次偏离决定归属：

| 第一次偏离 | 主要归因 |
| --- | --- |
| Query、Email ID 或邮件内容不同 | Golden Set / Grounding Data |
| Runtime 没有取得正确邮件 | CIQ / Runtime Context |
| Context 正确但模型选错工具 | Instructions、工具 Description 或模型 |
| Tool Call 正确但 Result 错误 | Extension 或工具依赖 |
| Tool Result 正确但回答遗漏或幻觉 | Prompt、Context 使用或模型生成 |
| Agent Result 正确但评分失败 | Assertion、LM Checklist 或其他 Metric |
| 只有后续 Turn 忘记约束 | Conversation 状态或多轮 Context |
| User Simulator 偏离 User Profile | Simulator，不算 Agent 回归 |

## 修复后怎样闭环

确定根因后，先在失败 Query 上验证修复，再重新运行功能切片和全量 Golden Set。不能只把失败 Query 跑通就结束，因为修复 Prompt 或工具描述可能再次影响其他场景。

```text
失败 Query 复现
→ Trace 找到第一次偏离
→ 修改对应层
→ 失败 Query 验证
→ 功能切片回归
→ 全量 Golden Set
→ 质量门禁
→ Ring 灰度
```

如果问题暴露了 Golden Set 未覆盖的新用户表达，就在使用内部平台完成 eyes-off 脱敏后补充 Query，并维护相应 Grounding Data 和 Assertion。这样同类问题会成为长期回归资产，而不是只修复一次线上案例。
