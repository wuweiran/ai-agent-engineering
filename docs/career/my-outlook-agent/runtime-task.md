---
layout: default
title: Agent Runtime 与任务恢复
parent: My Outlook Agent 部署
grand_parent: 工作经历
nav_order: 1
permalink: /docs/career/my-outlook-agent/runtime-task/
---

# Agent Runtime 与任务恢复

这部分展开 [My Outlook 项目概览]({{ site.baseurl }}/docs/career/my-outlook-agent/)中的 Service—Queue—Worker 链路，重点说明任务为什么能够跨请求、Pod 重启和版本发布继续执行。

## 一次请求怎样进入 Agent Loop

Outlook Client 把用户输入和 `conversation_id` 发送给 POS Service。POS Service 创建或读取 Task 与 Run，保存用户消息，创建第一个待执行 Step，然后统一返回 `202 + task_id`。后续 Agent Loop 由持久状态和 Step Result 驱动，不占用原始 HTTP 请求。

POS Service 是唯一的业务编排者。它根据当前 Task State 决定创建 Model、Deep Scan、Synthesis 或 Graph/API Step，并把 Step 类型、工具名称、版本、完整参数和输入 Reference 保存下来。Worker 不判断业务路径，只按 Step 参数调用内置 Handler 并写回 Result。Task Scheduler 在 Result 持久化后恢复 POS Service 中的 Agent Loop，由 POS 决定下一步。

```text
Outlook Client
→ POST /agent/tasks 或 POST /agent/tasks/{task_id}/messages
→ POS Service 保存消息并创建或读取 Task / Run
→ POS Service 创建 Step 并提供 Type + Tool + Version + Arguments
→ 返回 202 + task_id
→ 对应 Worker 执行 Step → Result 写回 Task Store
→ Task Scheduler 恢复 POS Service 中的 Agent Loop
   ├─ Final：保存最终回答并完成 Run
   ├─ Clarify：保存问题并等待用户输入
   └─ Continue：创建下一个 Model / Deep Scan / Synthesis / Graph/API Step
→ 客户端通过 Task API 查询进度与结果
```

伪代码如下：

```text
handleRequest(request):
   task = taskStore.loadOrCreate(request.taskId, request.conversationId)
   conversationStore.append(request.message)
   run = taskStore.startOrResumeRun(task.id)
   resumeAgentLoop(task, run)
   return accepted(task.id)

resumeAgentLoop(task, run):
   lastResult = taskStore.readCurrentStepResult(run.id)

   if lastResult is ModelResult:
      if lastResult.isFinal() or lastResult.needsClarification():
         conversationStore.append(lastResult.message)
         taskStore.finishOrWait(task, run, lastResult)
         return

      toolStep = taskStore.createStep(
         run.id,
         stepType = "tool",
         toolName = lastResult.toolCall.name,
         toolVersion = toolCatalog.version(lastResult.toolCall.name),
         arguments = validate(lastResult.toolCall.arguments)
      )
      queue(toolStep)
      return

   context = contextBuilder.build(task, lastResult)
   modelStep = taskStore.createStep(
      run.id,
      stepType = "model",
      arguments = { contextRef: context.reference }
   )
   queue(modelStep)

queue(step):
   taskStore.markQueued(step)
   serviceBus.publish(step.id, step.version)

onWorkerCompleted(step, result):
   taskStore.saveStepResult(step, result)
   taskScheduler.resume(step.taskId)
```

Worker 只执行 Step 和保存 Result，不负责决定 Agent 的下一步。下一步始终由 POS Service 中恢复后的 Agent Loop 根据最新持久状态决定。客户端断线不会取消已经持久化的 Task。

保存 Synthesis Artifact 等固定流程不需要模型产生 Tool Call：POS 直接创建确定性的 Step，或者 Worker 在完成当前 Step 时写入 SDS。只有执行与否需要模型判断的能力，才作为工具提供给模型。

## Task、Run、Step 和 Attempt

任务模型不是聊天记录，而是可以恢复的执行状态：

```text
Task
├─ task_id、conversation_id、user_id、tenant_id
├─ goal、status、current_step_id、deadline
├─ confirmed_facts、pending_items
└─ created_version

Run
├─ run_id、task_id、agent_version
├─ model_config、context_policy_version
└─ started_at、ended_at、stop_reason

Step
├─ step_id、run_id、step_type、status
├─ input_ref、output_ref、dependency_ids
├─ idempotency_key、not_before
└─ attempt_count、last_error
```

- **Task** 保存用户目标和总体状态；
- **Run** 表示一次执行尝试，并固定 Agent、模型和 Context 策略版本；
- **Step** 是能够独立持久化和恢复的模型、Deep Scan、Synthesis 或 Graph/API 步骤；
- **Attempt** 记录某个 Worker 对 Step 的一次实际处理，不等于新的业务动作。

## 一个 Agent Run 的数据保存在哪里

一次 Run 不会把全部 Context 保存成一个大字段。交互、执行状态、候选发现、生成产物和业务原文按照各自的生命周期分开持久化：

| 数据 | 保存位置 | 保存内容 |
| --- | --- | --- |
| 用户与 Agent 消息 | **Conversation Store** | 完整消息、顺序、角色和消息版本 |
| Task、Run、Step、Attempt | **Task Store** | 目标、状态、当前步骤、版本、Lease、重试信息和 Step Result |
| Deep Scan Finding | **Annotation Store** | 从邮件、日历和组织信号中提取的重点、承诺、待办和标注 |
| Briefing、Catch-up、Draft、Recommendation | **SDS Artifact Store** | Synthesis 生成的产物、版本、Citation 和 Artifact Reference |
| 邮件、日历、联系人原文 | **Microsoft 365 权威系统** | 原始业务对象；系统保存 Reference，需要时由 Graph/API Worker 读取 |
| Worker 调度消息 | **Azure Service Bus** | 携带 `task_id`、`step_id` 和 `step_version`，通知对应 Worker 有 Step 已就绪；不保存权威任务状态 |

它们通过稳定 ID 关联：

```text
conversation_id
→ task_id
→ run_id
→ step_id
→ attempt_id
→ artifact_ref / evidence_ref
```

每次调用模型前，Context Builder 根据当前 `task_id`、`run_id` 和 `step_id`，从这些持久化来源读取必要内容，选择、替换和压缩后生成本轮 Model Input。**Model Input 是临时工作集；能够恢复 Run 的依据是上述持久状态和 Reference。**

Task 状态使用以下受控转换：

```text
created → running → waiting_dependency → queued → running
                  ├→ waiting_user
                  ├→ completed
                  ├→ failed
                  └→ cancelled
```

Worker 不能只根据队列消息决定执行。消息只携带 `task_id` 和 `step_id`，Worker 获取消息后重新读取 Task Store，校验 Step 仍然可执行、依赖已经完成、任务没有取消且 Deadline 尚未耗尽。

## 状态与消息怎样保持一致

POS Service 或 Worker 需要先把下一步写入 Task Store，再向 Service Bus 发布一条只携带 `task_id`、`step_id` 和 `step_version` 的 Step 就绪消息。对应 Worker 收到消息后重新读取 Task Store，确认 Step 仍然可以执行。数据库与 Service Bus 不是一个事务，因此后台 Dispatcher 会周期性扫描已经进入 `queued`、但尚未记录成功投递的 Step 并补发；Worker 根据 Step Version 判断消息是否仍然有效。

```text
Task Store：Step waiting → queued
→ Dispatcher 发布 step_ready(step_id, step_version)
→ Worker 重新读取并校验 Step
```

如果数据库已经提交但进程在发送前崩溃，Dispatcher 会补发；如果消息已经发出但投递标记尚未更新，消息可能重复，因此**整个链路按 At Least Once 设计，Task Store 才是权威状态**。消费者依靠 Step Version 和持久状态处理重复消息，不假设数据库更新与消息发送天然原子。

## Worker 怎样领取任务

Worker 领取 Step 时写入 `lease_owner`、`lease_until` 和递增的 `lease_version`。执行期间定期续租；只有持有当前 Lease Version 的 Worker 才能提交 Step Result。**Lease 解决执行权，业务幂等解决重复副作用，二者不能互相替代。**

```text
queued
→ acquire lease
→ running
→ persist result
→ completed / retry_wait / terminal_failed
```

如果 Worker 在执行中失联，Lease 到期后 Step 可以重新入队。旧 Worker 即使稍后恢复，也会因 Lease Version 过期而无法覆盖新结果。对于创建 Draft、修改日历等有副作用动作，还必须把稳定业务请求 ID 传给下游；仅靠 Lease 不能避免网络分区边缘的短暂重复调用。

## Worker 重启后怎样恢复

**恢复不是重放 Conversation，也不是从第一步重新执行。**具体过程是：

1. Worker 获取当前 Step 的新 Lease；
2. 创建新的 Attempt，读取最近一个已持久化的稳定 Step；
3. 检查上一次 Step 是未开始、可重试失败，还是结果未知；
4. 根据 Task State、Artifact Reference 和最新业务状态重建 Context；
5. 跳过已经完成的步骤，从明确的 Pending Step 继续。

模型调用没有外部副作用，响应未保存时可以创建新 Attempt；Graph 只读查询可按预算有限重试；有副作用调用若超时，必须先用业务请求 ID查询真实结果。只有确认上一次未发生，才能重新提交。

## 等待为什么不占住 Worker

模型或 Graph 返回 `429 + Retry-After` 时，Step 转成 `waiting_dependency` 并保存 `not_before`。调度器到期后重新发布 Step，**Worker 不在进程内 `sleep`**。等待用户输入也采用相同原则：Task 进入 `waiting_user`，释放 Worker；新消息到达后创建下一 Step 并重新调度。

重试受四个预算约束：单 Step 最大 Attempt、整个 Task Deadline、模型或 Graph 的全局重试预算，以及当前租户的并发配额。POS Service 在 Step 中写入重试策略和预算，Worker 按该策略处理暂时失败并返回结构化结果，不自行改变业务路径。

## 任务取消与版本发布

取消只阻止未开始的 Step，并向正在运行的可取消调用发送信号；已经完成的外部动作不会因为 Task 标记为 `cancelled` 自动撤销。需要撤销时必须由业务 API 提供合法补偿。

Run 固定 `agent_version`、`model_config` 和 `context_policy_version`。发布新版本后，**新 Task 使用新版本，存量 Task 继续使用创建时的兼容版本**，避免执行中途改变 Prompt、工具契约或 Context 规则。若旧 Worker 已经回收，则必须运行显式 State Migration，而不是让任务静默漂移到新版本。

## 运行问题与修复
{: #runtime-bug-fixes }

### Worker 重启后重复生成 Artifact

**问题。**Synthesis Worker 已经把 Artifact 写入 SDS，但在 Step Result 写回 Task Store 前发生 Pod 重启。Lease 到期后，新 Worker 看到 Step 仍为 `running`，重新执行同一个 Step，产生重复 Artifact 和额外模型调用。

**发现。**Trace 中同一个 `step_id` 出现两个 `attempt_id`。第一次 Attempt 有 SDS 写入记录却没有 Step Result，第二次 Attempt 在 Lease 到期后重新开始。由此确认副作用已经发生，Task Store 却没有记录完成。

**修复。**执行前生成稳定的 `artifact_id` 和幂等键，Synthesis Worker 按该 ID 写入或更新 Artifact。恢复时先查询 SDS，找到已完成产物后补写 Step Result；Task Store 同时校验 Lease Version，拒绝旧 Worker 的迟到结果。修复后，Worker 重启会复用已有 Artifact并从持久状态继续。

### 迟到 Attempt 覆盖当前结果

**问题。**第一次 Attempt 调用超时后触发重试，第二次 Attempt 已经成功；第一次调用随后返回，并把旧结果覆盖到同一个 Step，Agent 因此根据过期结果继续决策。

**发现。**Trace 显示 Step Result 的更新时间晚于成功 Attempt，但写入记录来自更早的 `attempt_id`。同一 Step 的状态出现 `succeeded → running` 或成功结果被超时结果替换，说明结果写入没有校验当前执行权。

**修复。**Worker 提交结果时同时携带 `attempt_id` 和 Lease Version，Task Store 通过条件更新只接受当前 Attempt。Context Builder 只读取已经提交的当前 Step Result，迟到结果保留在 Attempt 记录中用于诊断，但不能推进任务状态。

### 滚动发布后旧 Task 无法继续

**问题。**Task 使用旧版 Tool Schema 和 Context Policy 创建，滚动发布后被新版 Worker 领取。新版 Worker 无法解析旧 Step，或者使用新契约继续执行，导致 Task 失败或长期停在队列中。

**发现。**失败集中出现在发布窗口，且具有相同的 `created_version`。Trace 显示消息投递和 Lease 都正常，错误发生在 Worker 读取旧 Step 后的反序列化或能力检查阶段，因此问题来自版本不兼容。

**修复。**Task 和 Run 固定 Agent、Context Policy 与 Worker Capability Version；调度器只把 Step 交给兼容 Worker。新增字段保持向后兼容，破坏性变化使用新版本并执行显式 State Migration。回滚时停止创建问题版本的新 Task，存量 Task 继续由兼容 Worker 完成。

## 这部分能够回答的面试题

- [短任务和长任务的 Agent Runtime 怎样部署？]({{ site.baseurl }}/docs/interview/ai-agent/system-production/#short-long-agent-runtime-deployment)
- [Task、Run、Step 和 Tool Call 分别表示什么？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#task-run-step-tool-call)
- [Worker 重启后怎样恢复 Agent 任务？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#agent-worker-recovery)
- [租约和业务幂等分别解决什么问题？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#agent-lease-vs-idempotency)
- [消息队列在 AI Agent 系统中解决什么问题？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#message-queue-agent-tasks)
- [模型 API 限流后怎样重试？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-rate-limit-retry)
- [长任务怎样跨越版本发布？]({{ site.baseurl }}/docs/interview/backend/performance-production/#long-running-task-versioning)
