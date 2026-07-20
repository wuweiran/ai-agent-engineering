---
layout: default
title: 工作流运行与可靠性
parent: Microsoft Purview 工作流
grand_parent: 工作经历
nav_order: 3
permalink: /docs/career/purview-workflow/reliability/
---

# 工作流运行与可靠性

## 端到端可用性

Purview Workflow 的底层执行由 Azure Logic Apps 提供，但系统可用性需要从 Catalog 发起请求一直看到 Catalog 的业务状态最终更新。

```text
Catalog 触发请求
→ Purview Workflow API
→ Logic Apps 创建实例并根据 runAfter 调度 HTTP Action
→ HTTP Action 调用 Purview Action API
→ Purview 完成任务并回调 Logic Apps
→ Logic Apps 更新节点状态并继续调度
→ 固定完成 Action 回调 Catalog
→ Catalog 更新业务状态
```

HTTP 请求成功、Logic Apps 实例创建成功和最后一个 Action 执行成功都只是中间结果。只有 Catalog 的业务状态最终正确，这次流程才算完成。因此我们分别关注：

- **触发可用性**：一个 Catalog 请求能否可靠对应到唯一工作流实例；
- **调度可用性**：满足 `runAfter` 的 Action 能否及时开始；
- **执行可用性**：Logic Apps 调用 Purview、Purview 执行业务和回调 Logic Apps 后，Action 能否得到确定终态；
- **闭环可用性**：工作流结果能否最终同步回 Catalog。

## 实例状态与节点调度

Workflow Definition 通过 `runAfter` 描述 Action 依赖。Logic Apps 根据前序 Action 的终态决定后继节点是执行、等待还是跳过。Purview Workflow 当时只支持顺序和条件流程，不向用户开放并行分支与汇聚。

每个实例固定使用创建时的 Definition Version，正在运行的实例不会随新版本改变。排障时既要看 Purview 包装后的实例，也要下钻到 Logic Apps Instance、Action 状态和执行尝试。

节点状态必须持久化。Action 完成后，即使调度组件重启，也要能够根据持久化结果继续判断后继节点。同一个节点可能被重复投递或被多个执行者看到，因此执行层需要控制同一时刻的执行权；这只能避免并发执行，外部副作用仍要依靠业务幂等保护。

对于 Purview 的异步 Action，服务侧还需要保存：

```text
请求已接收
→ 业务执行中
→ 业务执行完成
→ Logic Apps 回调待确认
→ Logic Apps 回调已确认
```

这样服务重启后可以继续执行或补发结果，而不是重新开始整个 Action。

## 关键故障窗口

### Catalog 触发超时

Catalog 调用 Workflow API 后可能没有收到响应，但 Logic Apps 实例可能已经创建。Workflow API 使用稳定的业务请求 ID 去重，并保存业务请求与 Logic Apps Instance ID 的映射。超时后先查询已有映射，不能直接创建第二个实例。

### Logic Apps 调用 Purview 超时

大部分节点是 Logic Apps HTTP Action。请求中包含 Action 类型、Parameter、工作流实例和执行标识。接口超时时，Logic Apps 无法确认 Purview 是否已经接收任务。

Purview Action API 按执行标识去重。重复请求到达时，如果任务仍在执行就返回已有状态；如果已经完成就复用已有结果。Logic Apps 重试时也要沿用相同的执行身份，避免重复产生业务副作用。

### Purview 内部执行超时

Purview 接受 Action 后，还可能进行 Expression 求值、数据库操作或下游调用。下游超时属于**结果未知**：对方可能没有收到请求，也可能已经完成操作但响应丢失。

副作用操作使用稳定的幂等键，例如：

```text
Workflow Instance ID + Action ID + 执行标识 + 业务动作版本
```

超时后先查询真实业务状态。确认没有执行时才使用相同幂等键重试；如果已经成功，就复用已有结果，不能换新键再次调用。

### Purview 回调 Logic Apps 超时

Purview 完成业务后调用 Logic Apps 回调接口。如果回调超时，业务结果不能丢失，也不能重新执行 Action。Purview 保存 Action 结果和回调状态，使用相同执行标识补发；Logic Apps 对重复回调幂等处理，不能让同一节点多次完成或多次触发后继。

恢复任务扫描“业务已完成、回调仍未确认”的记录。超过自动重试上限后保留待恢复状态并告警，继续补发已有结果或由人工处理。

### Catalog 完成回调失败

这是比 Logic Apps 节点回调更外层的业务闭环。Logic Apps 可能已经完成全部 Action，但固定完成 Action 调用 Catalog 失败。Catalog 完成接口按业务请求 ID 幂等，系统定期对账工作流终态与 Catalog 状态，补发回调或进入人工处理。

## 超时与重试

异步 Action 的超时分为入口请求超时、业务执行 Deadline、Logic Apps 回调超时和整个 Workflow Deadline。入口请求结束不代表 Action 已完成，回调超时也不代表 Logic Apps 没有收到结果，各阶段必须分别保存状态。

重试也分为三类：Logic Apps 重试 Action API、Purview 重试内部业务操作、Purview 重试 Logic Apps 回调。三者沿用同一个 Action 执行身份，但处理方式不同：

- 入口重复请求返回已有任务或结果；
- 读取和确定性计算可以有限重试；
- 限流和临时连接失败使用退避、抖动和最大次数；
- 参数、权限、Expression 和确定性业务错误不重试；
- 下游结果未知先查询状态；
- Logic Apps 回调未确认时只补发已有结果。

超过重试上限后，不会统一删除任务或直接判定业务失败：

| 场景 | 处理方式 |
| --- | --- |
| Logic Apps 无法调用 Purview | HTTP Action 进入失败或超时终态，按 `runAfter` 进入失败分支 |
| Purview 得到明确失败 | 保存错误并向 Logic Apps 回调失败 |
| 下游结果未知 | 停止盲目重试，保留执行标识和幂等键，通过状态查询、对账或人工处理确认 |
| 业务完成但无法回调 Logic Apps | 保存结果和待确认状态，只补发回调，不重新执行业务 |
| 固定完成 Action 无法回调 Catalog | 保留状态差异，通过幂等补发、对账或人工处理收敛 |

Workflow Definition 配置了失败分支时，可以继续执行通知、清理或补偿 Action；没有失败分支时，实例进入失败终态。补偿只处理已经确认产生的副作用，结果未知时不能直接反向操作。

## LLM 功能的故障处理

Workflow 概括与生成发生在 Definition 编辑阶段，不是 Logic Apps 运行实例中的 Action。Azure OpenAI 故障只影响这两项辅助功能，不会阻塞已经发布的 Workflow，也不会进入 Action 回调和 Catalog 闭环状态机。

模型调用的失败分成三类：

- **429、5xx 和临时网络错误**：在 30 秒 Deadline 内退避重试 2 次，仍失败就返回功能暂时不可用；
- **响应不是合法 JSON**：生成请求直接失败，前端保留用户原始需求并允许重新生成；
- **JSON 合法但 Definition Validator 不通过**：显示验证错误，用户手动修改或重新生成，不再次调用模型修复。

概括失败时，用户仍然可以直接查看原始 Workflow Definition；生成失败时，用户仍然可以使用现有编辑器手动创建 Workflow。模型没有直接保存或发布权限，因此失败响应不会污染已发布 Definition。

请求超时后不会在后台继续生成并覆盖用户页面。每次用户重新生成都会创建新的请求 ID，旧请求即使晚到也会被丢弃。服务记录 Model Deployment、Prompt Version、Token、延迟、错误类型、JSON 解析和 Validator 结果，但在日志和 Trace 中过滤敏感 Parameter。

## 节点卡住与恢复

节点卡住时，我们从实例开始，寻找**第一个没有按预期完成的 Action**：

1. 检查 `runAfter` 是否引用正确节点和终态；
2. 检查 Action 是否已经就绪并取得执行权；
3. 检查 Logic Apps 是否成功调用 Purview Action API；
4. 检查 Purview 是否接收请求、业务是否执行完成；
5. 检查下游是否超时、限流或结果未知；
6. 检查 Purview 是否回调 Logic Apps，以及回调是否确认；
7. 检查固定完成 Action 是否更新 Catalog。

排查时通过业务请求 ID、Logic Apps Instance ID、Action ID、执行标识、幂等键和下游请求关联日志与 Trace。恢复动作可能是重新投递就绪节点、接管失效执行、重试临时失败、查询未知结果或补发回调。

不能看到实例卡住就创建新工作流，因为原实例可能已经完成部分 Action。新实例可能重复发送通知、创建资源或更新业务状态。已经发生的副作用需要显式补偿，补偿本身也要幂等、可重试并受到监控。

部署规格、存储分区、容量、线上监控和一次回调积压事故统一记录在 [部署与生产运行]({{ site.baseurl }}/docs/career/purview-workflow/production/)。
