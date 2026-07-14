---
layout: default
title: 一次 Agent 任务需要保存哪些状态
parent: 大模型应用怎样改变后端
grand_parent: 后端工程
nav_order: 3
permalink: /docs/backend/llm-backend/agent-task-state/
---

# 一次 Agent 任务需要保存哪些状态

订单助手已经查询订单和物流，生成了地址修改方案，并等待用户确认。用户确认后，Worker 调用订单服务；就在响应返回时，应用实例重启了。

用户重新打开页面，只保存会话消息的系统无法回答：地址到底有没有修改，Agent 应该继续解释结果、重新执行，还是停下来等待人工处理。

多轮 Agent 任务的状态不只是一段对话。以下继续使用简化的模拟订单助手，观察数据库需要保存哪些事实，才能让任务可查询、可恢复，并避免重复产生业务副作用。

## 会话消息为什么不够

普通 Chat 常以 `conversation` 和 `message` 为中心：用户发送一条消息，模型生成一条回复。Agent 任务还包含模型调用、工具执行、人工确认和最终产物，它们的生命周期并不等于消息生命周期。

一个会话中可以先后发起多项任务；一项长任务也可能在没有新消息时由后台 Worker 继续运行。例如：

```text
conversation C-81
├─ message：用户询问未发货原因
├─ task T-301：调查物流并尝试修改地址
│  ├─ run R-17
│  ├─ 多个 model step
│  ├─ 多个 tool step
│  └─ 等待一次用户确认
└─ message：助手说明最终结果
```

最终回答适合展示，却无法代替这些执行事实。系统需要区分：

- **会话历史**：用户和助手交流了什么；
- **任务状态**：目标目前进行到哪里；
- **业务状态**：订单、物流和地址实际上是什么；
- **模型 Context**：下一轮临时选择哪些信息给模型。

数据库可以保存前三类信息，模型 Context 则由运行时按当前步骤组装。把完整会话无限追加给模型，既不能保证恢复正确，也会持续增加成本和噪声。

## 从任务、运行和步骤建立骨架

一套通用产品很难共用完全相同的表结构，但可以先用几个对象表达不同生命周期：

```text
agent_task
└─ agent_run
   ├─ agent_step
   │  ├─ model_call
   │  └─ tool_call
   ├─ task_event
   └─ artifact
```

| 对象 | 回答的问题 |
| --- | --- |
| `agent_task` | 用户交付了什么目标，当前总体状态是什么 |
| `agent_run` | 这项任务进行过哪次执行尝试，使用了哪版配置 |
| `agent_step` | 模型判断、工具执行和人工确认按什么顺序发生 |
| `model_call` | 某轮模型看到了哪些输入来源，产生了什么可观察输出和用量 |
| `tool_call` | 模型请求了什么，后端是否批准，真实执行结果是什么 |
| `task_event` | 状态怎样变化，前端断线后需要补回哪些进度 |
| `artifact` | 任务最终生成了哪份报告、文件或其他可交付物 |

这些对象不要求全部放在一个关系数据库。大文本和文件可以进入对象存储，Trace 可以进入观测系统，业务表只保存稳定 ID、状态和引用。关键是每类信息有明确所有者和保留期限。

## Task 保存用户目标和总体状态

任务记录描述产品对用户作出的执行承诺：

```text
task_id: T-301
conversation_id: C-81
user_id: U-204
tenant_id: TENANT-7
task_type: investigate_and_change_address
goal: 查询未发货原因，并在允许时修改地址
status: waiting_for_confirmation
active_run_id: R-17
version: 8
created_at: 2026-07-14T11:20:00Z
```

`goal` 不是把用户原话复制一遍就结束。应用还可以保存经过确认的范围、允许使用的资源和交付条件，例如只处理 `O-1042`、不得取消订单、地址修改必须由用户确认。

任务状态表达用户可理解的执行阶段。一个简化生命周期可以是：

```text
queued → running → waiting_for_confirmation → running → completed
                   ├→ cancelled
                   └→ failed
```

状态转换应由后端命令执行，而不是让任意组件直接写字符串。例如，只有仍在 `waiting_for_confirmation` 且确认方案尚未过期的任务，才能进入 `running`。

可以通过版本条件避免用户确认、取消和 Worker 恢复互相覆盖：

```sql
UPDATE agent_tasks
SET status = 'running',
    version = version + 1
WHERE task_id = 'T-301'
  AND status = 'waiting_for_confirmation'
  AND version = 8;
```

更新零行时，调用方读取最新状态，确认任务是否已经取消、完成或被另一执行者接管。

## Run 区分任务与执行尝试

任务是用户要完成的事，运行是系统为完成它进行的一次执行尝试。模型供应商超时、Worker 崩溃或配置发布后重新执行，都可能产生新的 Run：

```text
T-301
├─ R-16 failed：模型服务持续超时
└─ R-17 running：从已确认步骤恢复
```

Run 可以保存：

```text
run_id
task_id
attempt_no
status
agent_config_version
model_and_prompt_version
toolset_version
started_at
ended_at
stop_reason
usage_summary
```

版本信息使工程师能够解释行为变化。任务昨天成功、今天失败，原因可能不是应用代码，而是模型、Prompt、工具 Schema 或 Context 组装策略发生了变化。

新的 Run 不表示所有业务步骤从头再做。它要读取此前已确认的工具结果，决定哪些信息可以复用、哪些查询需要刷新，以及哪些结果未知的副作用必须先核验。

## Step 记录可以恢复的执行边界

Agent 的每轮工作可以记录为有序步骤：

```text
1  model   请求查询订单
2  tool    get_order succeeded
3  model   请求查询物流
4  tool    get_shipment succeeded
5  model   生成地址修改方案
6  human   waiting_for_confirmation
7  tool    change_shipping_address unknown
```

Step 不需要保存模型内部的隐含思考。系统应保存能够解释和恢复执行的外部事实：本轮输入来源、模型产生的文本或工具请求、工具校验结果、状态变化和结束原因。模型未提供给应用的私有推理既不是业务事实，也不应被当作恢复依据。

一个步骤通常需要这些字段：

```text
step_id
run_id
sequence
step_type
status
input_ref
output_ref
started_at
completed_at
error_code
```

大块输入输出可以放在单独存储中，通过引用关联。`sequence` 表示任务观察到的顺序，但并行查询还要保存父子关系或依赖关系，不能仅凭行号推断因果。

只有达到稳定边界的步骤才适合作为恢复点。模型流式生成到半句、工具仍在执行、外部写入结果未知，都不能被伪装成已完成步骤。

## Tool Call 要保存请求、决定和真实结果

工具调用最容易产生重复副作用，需要比普通日志更明确的状态：

```text
tool_call_id: TC-408
step_id: S-7
tool_name: change_shipping_address
tool_version: 3
arguments_ref: blob://agent-inputs/TC-408
validation_status: passed
approval_id: AP-91
execution_status: unknown
idempotency_key: address-change-AC-9021
business_request_id: AC-9021
result_ref: null
```

工具记录至少要区分：

```text
requested
validated
waiting_for_approval
executing
succeeded
rejected
failed
unknown
```

`failed` 表示已经确认动作没有成功，满足条件时可以重试；`unknown` 表示请求可能已经生效，只是应用没有取得结果。二者混在一起，会让恢复程序把一次成功的退款、发信或地址修改再做一遍。

调用有副作用的服务前，应用先保存稳定 `tool_call_id` 和 `idempotency_key`。业务服务执行后保存返回的业务 ID 与结果。数据库事务不能跨越外部订单服务，因此中间仍可能崩溃；恢复时要携带同一幂等键重查或重试，而不是生成新的调用。

工具参数中可能包含个人信息和敏感业务数据。数据库可以保存加密后的参数、经过裁剪的快照或受控对象引用，不应为了调试无限期保留完整原始内容。

## 业务状态仍由领域服务掌握

Agent 数据库记录 `change_shipping_address succeeded`，不应成为订单地址的权威来源。订单服务仍然掌握订单当前地址、版本和是否已经出库。

不同状态回答不同问题：

| 状态 | 权威来源 | 用途 |
| --- | --- | --- |
| Agent 任务是否完成 | Agent 任务存储 | 展示、调度和恢复执行 |
| 工具请求是否已确认 | 工具调用记录及目标服务回执 | 避免重复执行 |
| 订单地址现在是什么 | 订单服务 | 业务查询与后续交易 |
| 模型下一轮看到什么 | 本轮 Context 组装结果 | 支持模型继续判断 |

服务恢复后，即使工具记录显示成功，也可以在关键动作完成前重新读取订单状态。反过来，若本地记录是 `unknown`，订单服务查询到地址版本已经更新，就应补记成功，而不是相信旧的本地状态再次提交。

**Agent 状态说明任务怎样推进，领域状态说明业务世界实际上发生了什么。**

## 当前状态、事件和 Trace 不要混成一份数据

一项任务通常同时需要三种记录方式。

### 当前状态用于快速回答“现在怎样”

`agent_task.status`、当前 Run 和最近 Step 让 API 能快速返回：任务正在运行、等待确认还是已经完成。调度器也据此寻找可执行任务。

### 任务事件用于回答“状态怎样变化”

前端可能收到：

```text
task.started
model.started
tool.requested
tool.completed
confirmation.required
task.completed
```

需要断线恢复的事件应有稳定 `event_id`、任务内序号和有限保留期。前端带着最后事件 ID 重连时，后端补回遗漏事件。

事件记录不一定保存每个文本 Delta。高频片段可以只在实时通道短暂传输，定期保存草稿；确认完成后再保存最终消息或 Artifact。否则一个长回答可能产生数千行低价值事件，增加写入和重放成本。

### Trace 用于回答“为什么这样”

Trace 可以记录每轮 Context 来源、模型延迟、Token、工具校验、依赖调用和错误细节。它面向排障与评测，数据量大、敏感度高，访问权限和保留期限通常不同于用户可见任务记录。

当前状态不是完整历史，事件流不是调试数据库，Trace 也不应成为产品查询的唯一来源。分开以后，每类数据才能按自己的读取方式和成本治理。

## Artifact 保存最终交付物

Agent 完成任务后，产物不一定是一条聊天消息。它可能生成售后报告、修改后的文件、数据表或审批方案。

Artifact 记录可以包含：

```text
artifact_id: A-551
task_id: T-301
type: address_change_receipt
storage_uri: object://agent-artifacts/A-551.json
content_hash: sha256:...
created_by_step: S-9
status: committed
```

Artifact 与生成草稿要区分。只有经过必要校验并成功写入稳定存储的内容才标记为 `committed`。任务发出 `completed` 事件前，应确保用户能够重新读取最终结果。

对于“修改地址”这样的任务，订单服务中的新地址是业务结果，Artifact 可以只是面向用户的变更回执。不要复制出另一份能够与订单状态冲突的业务真相。

## 怎样保证状态变化和事件不会互相丢失

假设 Worker 把任务改成 `waiting_for_confirmation`，却在发布前端事件前崩溃。数据库状态已经变化，用户却看不到确认按钮。

一种实现是在同一数据库事务中更新任务并写入待发布事件：

```text
事务开始
├─ 更新 agent_task.status
├─ 创建 approval 记录
└─ 写入 task_event / outbox 记录
事务提交
```

独立发布程序随后把事件推送到 SSE 服务或消息系统，并记录投递进度。即使事件重复发布，稳定 `event_id` 也能帮助订阅端去重。

若任务状态和事件分别提交，中间故障会制造无法解释的缺口。这里可以复用普通后端中的事务 Outbox 思路，但事件仍只表示 Agent 状态变化，不代替订单服务中的业务事实。

## 实例重启后怎样决定从哪里继续

Worker 接管 `T-301` 时，不应简单把全部历史再次发给模型。它先从持久状态还原执行边界：

1. 读取任务状态，确认没有取消且允许继续；
2. 取得执行租约，避免多个 Worker 同时推进；
3. 找到最近一个稳定完成的 Step；
4. 检查未完成 Tool Call，特别是 `executing` 和 `unknown`；
5. 向领域服务核验可能已经发生的副作用；
6. 根据当前业务状态重新组装下一轮 Context；
7. 创建新的 Step，继续模型或工具执行。

例如，订单服务确认地址已经修改，恢复程序先把 `TC-408` 补记为 `succeeded`，再让模型生成最终解释。若订单仍是旧地址且原调用明确未执行，才可以使用相同幂等键重新提交。

租约、心跳、队列投递和 Worker 重试属于后台执行机制。数据库先把任务、运行、步骤和工具结果表达清楚，后台系统才知道接管的究竟是什么。

## 哪些数据值得长期保存

为了“以后也许有用”保存所有 Prompt、检索文档、工具原始响应和文本片段，会快速累积成本与隐私风险。保留策略应根据用途区分：

- 任务目标、最终状态和业务回执按产品与审计要求保存；
- 工具参数和结果只保留恢复、争议处理所需字段；
- 高频事件设置较短保留期，完成后可压缩成摘要；
- Trace 和评测样本经过脱敏，并限制访问范围；
- 凭据、访问令牌和工具内部秘密不进入任务存储；
- 用户删除请求要覆盖会话、任务、Artifact 和观测副本。

模型 Context 也不应通过“整份数据库记录”直接构造。运行时按任务与权限选择信息，必要时生成可验证摘要，同时保留来源引用。

## 一次 Agent 任务最终要记住什么

要让多轮任务在刷新页面、服务发布和 Worker 重启后仍能继续，系统需要保存几类不同事实：

- Task 保存用户目标、范围和总体状态；
- Run 区分多次执行尝试及其模型、Prompt 和工具版本；
- Step 标出模型判断、工具执行与人工确认的稳定边界；
- Tool Call 保存候选请求、校验、审批、幂等键和真实结果；
- Event 支持界面观察状态变化和断线续传；
- Artifact 保存已经提交的最终交付物；
- 领域服务继续掌握订单、退款等真实业务状态。

**模型 Context 帮助下一轮作出判断，持久任务状态帮助系统确认已经发生了什么以及接下来能安全做什么。**

这些状态准备好以后，Agent 才能离开一次 HTTP 请求，由队列和 Worker 在后台继续。租约、心跳、取消与重试将建立在这套持久状态之上，使长时间执行不依赖某个进程或浏览器连接。
