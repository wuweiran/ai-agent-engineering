---
layout: default
title: Agent 任务的持久状态
parent: 大模型应用后端
grand_parent: 后端工程
nav_order: 3
permalink: /docs/backend/llm-backend/agent-task-state/
---

# Agent 任务的持久状态

订单助手查询了订单和物流，生成地址修改方案，用户确认后开始调用订单服务。就在响应返回时，Agent Worker 重启了。

用户重新打开页面，系统必须回答：地址是否已经修改，任务应继续生成解释、重新执行，还是暂停核对。只保存聊天消息无法给出这些答案。

状态、Context、会话历史和长期记忆各自承担什么职责，见[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)。这里从应用后端出发，讨论已经确定的任务状态怎样落到数据库，使调度、并发控制和故障恢复有可靠依据。

**持久状态要记录系统已经确认了什么，以及哪一步可以安全继续。**

## 持久对象从恢复问题倒推

订单服务掌握订单地址是否真的修改；Agent 数据库不复制这份业务真相，而是记录自己的执行事实：用户交付了什么目标，哪次运行正在推进，哪些步骤已经稳定完成，哪项远程动作仍然结果未知。

最小结构可以表达为：

```text
agent_task T-301
└─ agent_run R-17
   ├─ agent_step S-1：get_order succeeded
   ├─ agent_step S-2：get_shipment succeeded
   ├─ agent_step S-3：confirmation received
   └─ agent_step S-4
      └─ tool_call TC-408：change_shipping_address unknown
```

表名可以变化，恢复所需的四种事实不能丢失：Task 保存目标和总体状态，Run 区分执行尝试，Step 标记稳定执行边界，Tool Call 记录副作用是否已经确认。

## Task 是可以并发修改的状态机

Task 表示产品已经接受的一项用户目标：

```text
task_id: T-301
conversation_id: C-81
user_id: U-204
goal: 查询未发货原因，并在允许时修改地址
status: waiting_for_confirmation
active_run_id: R-17
version: 8
```

除了用户原话，还要保存已经确定的任务范围和限制，例如只处理 `O-1042`、不能取消订单、地址修改必须由用户确认。这样，恢复执行不需要依靠模型重新解释整段对话。

总体状态沿业务过程变化：

```text
queued → running → waiting_for_confirmation → running → completed
                   ├→ cancelled
                   └→ failed
```

用户确认、取消请求和 Worker 恢复可能同时修改任务。状态转换需要前置条件或版本号，例如只有仍在等待确认且版本为 8 的任务才能进入运行状态：

```sql
UPDATE agent_tasks
SET status = 'running',
    version = version + 1
WHERE task_id = 'T-301'
  AND status = 'waiting_for_confirmation'
  AND version = 8;
```

更新零行表示状态已经变化，调用方应读取当前结果，不能覆盖另一项操作。

## Run 固定一次执行使用的版本

模型服务超时或 Worker 崩溃后，任务仍是 `T-301`，系统可以创建新的执行尝试：

```text
T-301
├─ R-16 failed：模型服务持续超时
└─ R-17 running：从已确认步骤继续
```

Run 记录这次尝试使用的模型、Prompt、工具集合和 Agent 配置版本，以及开始时间、结束原因和用量。工程师回放失败任务时，才能知道当时真正运行的是哪套组合。

新的 Run 不表示所有动作从头再做。它读取此前已经确认的 Step，尤其不能重复执行真实业务副作用。长任务是否在中途升级版本，应由明确的迁移策略决定，不能随着任意 Worker 的当前配置漂移。

## Step 是稳定检查点，不是每个内部念头

任务过程可以保存为：

```text
S-1  tool    get_order succeeded
S-2  tool    get_shipment succeeded
S-3  human   confirmation received
S-4  tool    change_shipping_address unknown
```

Step 记录模型调用、工具执行或人工确认的顺序、状态和输入输出引用。它保存能够解释外部行为和恢复执行的事实，不保存模型未提供给应用的内部推理。

只有稳定完成的步骤才是恢复点。半句流式文本、仍在执行的工具和结果未知的写操作都不能标记为 `succeeded`。

创建下一项 Step 时，还要确认当前 Run 仍拥有任务执行权。若两个 Worker 在租约切换期间短暂重叠，数据库中的唯一约束、Task 版本和 Step 序号共同阻止它们各自推进一条执行历史。

并行只读步骤可以保存父步骤或依赖关系。恢复时根据依赖判断哪些分支已完成，不能仅凭写入顺序猜测执行关系。

## Tool Call 单独记录远程副作用

有副作用的 Tool Call 需要比普通模型步骤更明确的状态：

```text
tool_call_id: TC-408
step_id: S-4
tool_name: change_shipping_address
arguments_ref: ...
approval_id: AP-91
status: unknown
idempotency_key: address-change-AC-9021
business_request_id: AC-9021
```

`failed` 和 `unknown` 必须区分。前者表示已经确认动作没有成功，后者表示请求可能已经生效，只是 Agent 后端没有得到答案。把 `unknown` 当失败重试，可能再次退款、发信或修改业务数据。

远程调用前，后端先持久化 Tool Call 和稳定业务请求 ID；响应到达后再保存领域服务返回的业务 ID。跨服务无法依靠一个本地事务消除全部故障窗口，恢复时仍要使用同一请求 ID 查询目标服务。

订单服务确认地址已经修改，Agent 状态就补记 `succeeded`；明确未执行才允许再次提交。订单地址本身仍以订单服务为准，Agent 数据库不能成为另一份可以独立修改的地址真相。

## 租约与执行状态

Worker 取得一段有期限的任务租约，并定期续约。实例失联后，租约到期，其他 Worker 才能接管任务。租约避免多个实例长期同时工作，却不能保证外部动作只执行一次。

接管 `T-301` 的 Worker 按持久状态恢复：

```text
读取 Task，确认任务允许继续
→ 取得租约并创建新的 Run
→ 找到最近稳定完成的 Step
→ 检查 executing 或 unknown 的 Tool Call
→ 向领域服务核验可能发生的副作用
→ 用当前业务事实重新组装 Context
→ 创建新 Step 继续执行
```

订单服务若确认地址已修改，Worker 补记工具成功，再让模型生成最终说明。订单仍是旧地址且原请求明确未执行，才使用同一幂等键继续。

租约和心跳回答“谁暂时推进任务”；Tool Call 状态与领域服务回执回答“这项业务动作能不能再次执行”。两者不能互相代替。

## Event 和 Artifact 是状态的外部投影

Task 表保存当前状态，事件记录状态怎样变化，使前端能够展示进度并在断线后续传：

```text
task.started
tool.completed
confirmation.required
task.completed
```

重要事件应与状态变化可靠关联，避免数据库已经完成而前端永远收不到完成通知。可以在同一事务中写入任务状态与 Outbox 事件，再由发布器推送给订阅者。

不是每个 Token Delta 都值得长期保存。高频文字片段可以短暂传输或合并成草稿；任务状态事件使用稳定 ID 和任务内序号。

报告、文件或变更回执等 Artifact 应写入稳定存储并关联产生它的 Step。只有必要交付物已经可读取，任务才进入 `completed`。

## 保存足以恢复的信息，不复制所有过程

完整 Prompt、检索文档、工具原始响应和每个文字片段可能包含个人信息，也会快速增加存储成本。任务数据库只保留恢复、审计和用户交付需要的内容：稳定 ID、状态、版本、关键参数快照或受控引用、最终结果和错误类别。

凭据和访问令牌不能进入任务记录。Trace 可以保留更丰富的排障信息，但需要脱敏、访问控制和独立保留期限。用户删除请求也要覆盖会话、任务、Artifact 和观测副本。

## Agent 状态的后端骨架

一次可恢复的多轮任务需要守住四层事实：

```text
Task：目标和总体状态，使用条件更新处理并发
Run：一次执行尝试及其版本组合
Step：已经稳定完成到哪个执行边界
Tool Call：远程副作用是否已经得到确认
```

事件帮助外部观察任务，Artifact 保存最终交付物，领域服务继续掌握订单、支付等真实业务状态。

Agent 设计决定哪些信息值得成为状态；应用后端负责让这些状态在并发、断线和服务重启后仍然可信，并据此安全地继续执行。
