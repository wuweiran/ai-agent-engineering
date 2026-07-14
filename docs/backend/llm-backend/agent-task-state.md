---
layout: default
title: 一次 Agent 任务需要保存哪些状态
parent: 大模型应用怎样改变后端
grand_parent: 后端工程
nav_order: 3
permalink: /docs/backend/llm-backend/agent-task-state/
---

# 一次 Agent 任务需要保存哪些状态

订单助手查询了订单和物流，生成地址修改方案，用户确认后开始调用订单服务。就在响应返回时，Agent Worker 重启了。

用户重新打开页面，系统必须回答：地址是否已经修改，任务应继续生成解释、重新执行，还是暂停核对。只保存聊天消息无法给出这些答案。

以下从这个模拟恢复场景倒推 Agent 的最小持久状态。核心原则是：**模型 Context 帮助下一轮判断，任务状态帮助系统确认已经发生了什么，以及接下来能安全做什么。**

## 对话、任务和业务事实不是同一种状态

会话消息记录用户和助手说过什么。一项 Agent 任务还可能在没有新消息时由后台 Worker 连续执行：

```text
conversation C-81
└─ task T-301：调查物流并修改地址
   ├─ 查询订单
   ├─ 查询物流
   ├─ 等待用户确认
   └─ 修改地址
```

系统要区分：

- 会话历史用于呈现交流；
- 任务状态说明目标进行到哪里；
- 订单服务掌握地址和物流的真实业务状态；
- Context 是下一轮临时选择给模型的信息。

完整聊天记录不能证明工具是否执行成功，数据库全部历史也不应不加选择地塞进模型请求。

## Task 保存目标和总体阶段

任务表示产品对用户承诺完成的事情：

```text
task_id: T-301
conversation_id: C-81
user_id: U-204
goal: 查询未发货原因，并在允许时修改地址
status: waiting_for_confirmation
active_run_id: R-17
version: 8
```

除了用户原话，任务还应保存已经确认的范围和限制，例如只处理 `O-1042`、不能取消订单、地址修改必须由用户确认。

总体状态可以沿实际业务变化：

```text
queued → running → waiting_for_confirmation → running → completed
                   ├→ cancelled
                   └→ failed
```

状态转换要带前置条件。用户确认、用户取消和 Worker 恢复可能同时发生，版本号或条件更新可以防止它们互相覆盖。

## Run 区分同一任务的执行尝试

模型服务持续超时或 Worker 崩溃后，任务仍是 `T-301`，系统可能创建新的执行尝试：

```text
T-301
├─ R-16 failed：模型服务超时
└─ R-17 running：从已确认步骤继续
```

Run 记录这次尝试使用的模型、Prompt、工具集合和 Agent 配置版本，以及开始时间、结束原因和用量。行为发生变化时，工程师才能知道是代码、模型还是工具契约发生了变化。

新的 Run 不表示所有动作从头再做。它要读取此前已经确认的步骤，尤其不能重复执行真实业务副作用。

## Step 标记可以恢复的执行边界

任务过程可以表达为：

```text
1  model   请求查询订单
2  tool    get_order succeeded
3  model   请求查询物流
4  tool    get_shipment succeeded
5  model   生成地址修改方案
6  human   confirmation received
7  tool    change_shipping_address unknown
```

Step 保存模型调用、工具执行或人工确认的顺序、状态和输入输出引用。它不需要保存模型未提供给应用的内部推理，只保存能够解释外部行为和恢复执行的事实。

只有稳定完成的步骤才是恢复点。半句流式文本、仍在执行的工具和结果未知的写操作都不能伪装成 `succeeded`。

如果两个只读查询并行执行，还要保存它们的依赖关系或调用 ID，不能只凭行号推断谁依赖谁。

## Tool Call 记录副作用是否已经确认

工具调用是恢复中最重要的状态：

```text
tool_call_id: TC-408
step_id: S-7
tool_name: change_shipping_address
arguments_ref: ...
approval_id: AP-91
status: unknown
idempotency_key: address-change-AC-9021
business_request_id: AC-9021
```

`failed` 和 `unknown` 必须区分。前者表示已经确认动作没有成功，后者表示请求可能已经生效，只是 Agent 后端没有得到答案。把 `unknown` 当失败重试，可能再次退款、发信或修改业务数据。

有副作用的服务调用前，先保存稳定 Tool Call 和幂等键。执行后保存业务系统返回的 ID。跨服务无法依靠一个本地事务消除所有故障窗口，恢复时仍要用同一业务请求 ID 查询目标服务。

订单服务确认地址已经修改，Agent 状态就补记 `succeeded`；明确未执行才允许再次提交。订单地址本身仍以订单服务为准，Agent 数据库不能成为另一份可独立修改的地址真相。

## Event 和 Artifact 按需要出现

任务状态回答“现在怎样”，事件记录状态怎样变化，使前端可以展示并在断线后续传：

```text
task.started
tool.completed
confirmation.required
task.completed
```

不是每个 Token Delta 都值得长期保存。高频文字片段可以短暂传输或合并成草稿，重要状态事件使用稳定 ID 和任务内序号。

Agent 的交付物也不一定是一条消息。报告、代码文件或变更回执可以保存为 Artifact，并关联产生它的 Step。只有内容写入稳定存储并通过必要校验后，才把任务标记完成。

Event 和 Artifact 服务具体产品需求，不需要为了表结构完整而让每项任务都生成一套相同记录。

## Worker 重启后怎样继续

接管 `T-301` 的 Worker 不会简单重放全部对话。它按持久状态恢复：

```text
读取 Task，确认允许继续
→ 取得执行租约
→ 找到最近稳定完成的 Step
→ 检查 executing / unknown 的 Tool Call
→ 向领域服务核验可能发生的副作用
→ 用当前业务事实重新组装 Context
→ 创建新 Step 继续执行
```

订单服务若确认地址已修改，Worker 补记工具成功，再让模型生成最终说明。订单仍是旧地址且原请求明确未执行，才使用同一幂等键继续。

租约和心跳决定谁暂时推进任务；Tool Call 状态与领域服务回执决定业务动作能否再次执行。两者不能互相代替。

## 保存状态也要限制范围

完整 Prompt、检索文档、工具原始响应和每个文字片段可能包含个人信息，并且会快速累积成本。任务存储只保留恢复、审计和用户交付需要的内容：稳定 ID、状态、关键参数快照或受控引用、最终结果和错误类别。

凭据和访问令牌不能进入任务记录。Trace 可以保留更丰富的排障信息，但应脱敏、限制权限并采用独立保留期限。用户删除请求也要覆盖会话、任务、Artifact 和观测副本。

## Agent 状态的最小骨架

多轮任务最先需要四类对象：

```text
Task：用户要完成什么，现在处于哪个总体阶段
Run：这次执行尝试使用了什么版本，怎样结束
Step：已经稳定完成到哪一步
Tool Call：真实业务动作是否已经确认
```

事件帮助用户观察过程，Artifact 保存最终交付物，领域服务继续掌握真实业务状态。

**只保存最终回答，系统只能展示结果；保存目标、步骤和工具结果，系统才能在断线、重试和服务重启后解释并继续执行。**
