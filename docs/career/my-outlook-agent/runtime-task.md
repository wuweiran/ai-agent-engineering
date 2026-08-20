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

## 为什么不能把整个 Agent Loop 都放在 HTTP 请求中

My Outlook 同时处理两类工作：用户对话中的短决策，以及 Deep Scan、Synthesis、模型调用和 Graph 查询等可能持续较久的后台任务。二者如果都绑在一个 HTTP 请求和 Service Pod 上，会出现三个问题：客户端断线导致任务生命周期不明确；Pod 发布或重启后执行上下文丢失；Worker 等待模型限流或 Graph `Retry-After` 时持续占用计算资源。

因此，**短决策留在 POS Service，长任务持久化后交给 Worker**。若下一步能够在短时间内完成，Service 可以同步执行并返回；需要等待、重试或跨实例继续的步骤则持久化为 Task，返回 `202 + task_id`，由调度器投递给 Worker。客户端通过状态接口或事件流观察进度，**断线不等于取消**。

```text
POST /agent/tasks
→ 创建 Task 与初始 Run
→ POS Service 执行一次 Agent Decision
   ├─ Final / Clarify：保存结果并返回
   └─ Async Step：持久化 Step → 发布 task_id → Worker 执行
```

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

Task 状态使用受控转换，例如：

```text
created → running → waiting_dependency → queued → running
                  ├→ waiting_user
                  ├→ completed
                  ├→ failed
                  └→ cancelled
```

Worker 不能只根据队列消息决定执行。消息只携带 `task_id` 和 `step_id`，Worker 获取消息后重新读取 Task Store，校验 Step 仍然可执行、依赖已经完成、任务没有取消且 Deadline 尚未耗尽。

## 状态与消息怎样保持一致

POS Service 或 Worker 需要先把下一步写入 Task Store，再向 Service Bus 发布一条只携带 `task_id`、`step_id` 和 `step_version` 的唤醒消息。数据库与 Service Bus 不是一个事务，因此后台 Dispatcher 会周期性扫描已经进入 `queued`、但尚未记录成功投递的 Step 并补发；消费者始终重新读取 Task Store，并根据 Step Version 判断消息是否仍然有效。

```text
Task Store：Step waiting → queued
→ Dispatcher 发布 step_ready(step_id, step_version)
→ Worker 重新读取并校验 Step
```

如果数据库已经提交但进程在发送前崩溃，Dispatcher 会补发；如果消息已经发出但投递标记尚未更新，消息可能重复，因此**整个链路按 At Least Once 设计，Task Store 才是权威状态**。这个实现也可以替换为本地消息表，关键是不假设数据库更新与消息发送天然原子。

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

重试受四个预算约束：单 Step 最大 Attempt、整个 Task Deadline、模型或 Graph 的全局重试预算，以及当前租户的并发配额。每一层都自行重试会造成放大，因此只有负责该 Step 的 Worker 决定是否重试，上层 Agent Loop只接收结构化的最终失败语义。

## 任务取消与版本发布

取消只阻止未开始的 Step，并向正在运行的可取消调用发送信号；已经完成的外部动作不会因为 Task 标记为 `cancelled` 自动撤销。需要撤销时必须由业务 API 提供合法补偿。

Run 固定 `agent_version`、`model_config` 和 `context_policy_version`。发布新版本后，**新 Task 使用新版本，存量 Task 继续使用创建时的兼容版本**，避免执行中途改变 Prompt、工具契约或 Context 规则。若旧 Worker 已经回收，则必须运行显式 State Migration，而不是让任务静默漂移到新版本。

## 这部分能够回答的面试题

- [短任务和长任务的 Agent Runtime 怎样部署？]({{ site.baseurl }}/docs/interview/ai-agent/system-production/#short-long-agent-runtime-deployment)
- [Task、Run、Step 和 Tool Call 分别表示什么？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#task-run-step-tool-call)
- [Worker 重启后怎样恢复 Agent 任务？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#agent-worker-recovery)
- [租约和业务幂等分别解决什么问题？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#agent-lease-vs-idempotency)
- [消息队列在 AI Agent 系统中解决什么问题？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#message-queue-agent-tasks)
- [模型 API 限流后怎样重试？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-rate-limit-retry)
- [长任务怎样跨越版本发布？]({{ site.baseurl }}/docs/interview/backend/performance-production/#long-running-task-versioning)
