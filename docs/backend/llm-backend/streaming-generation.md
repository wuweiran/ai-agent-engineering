---
layout: default
title: 流式生成与任务事件
parent: 大模型应用后端
grand_parent: 后端工程
nav_order: 1
permalink: /docs/backend/llm-backend/streaming-generation/
---

# 流式生成与任务事件

用户打开订单助手，输入：“我买的鞋为什么还没发货？”模型需要读取订单和物流信息，再逐步生成解释。若后端一直等到完整内容产生，用户会在数秒内看不到任何反馈。

这里涉及两个层级。[Agent Task]({{ site.baseurl }}/docs/backend/llm-backend/agent-task-state/)承载“查明未发货原因并给出解释”这个完整用户目标，内部可能查询订单、调用物流工具，并进行一次或多次模型 Generation。流式生成发生在其中某一次 Generation 内；SSE 既可以传输这次 Generation 的文本片段，也可以传输整个 Task 的工具调用和状态变化。

[流式生成和流式返回]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#streaming-generation-return)是两个相邻但不同的过程：

- **流式生成**：模型自回归地逐步产生 Token，而不是一次计算出完整回答；
- **流式返回**：应用收到上游生成的文本片段后，不等待完整回答，立即持续传给客户端。

完整链路是：

```text
模型逐步生成 Token
→ 模型 API 返回文本片段
→ 应用后端接收并转发
→ SSE 等协议推送到客户端
→ 客户端持续拼接和展示
```

这种方式让用户更早看到文字和任务进度，降低感知等待时间（[为什么大模型回答通常使用流式返回？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-streaming-purpose)），但不会缩短模型生成完整回答的时间。它也把一次调用变成持续一段时间的状态过程。**流是临时传输过程，持久状态才是系统可以重新确认和恢复的事实。**

## 后端先给生成一个稳定身份

用户提交问题时，后端确认身份和订单范围，再创建生成记录：

```text
generation_id: G-2048
conversation_id: C-81
user_id: U-204
status: starting
created_at: 2026-07-14T11:20:00Z
```

稳定 ID 把用户请求、模型调用、最终消息和错误关联起来。客户端重试提交时还应携带请求 ID，避免一次操作创建两项相同生成。

订单事实必须在当前用户权限范围内重新查询。历史回答提到 `O-1042`，不能代替本轮授权。

## SSE 传输有类型的事件

[SSE]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#agent-sse-events)（Server-Sent Events，服务器推送事件）是一种基于普通 HTTP 的单向流式协议。客户端发起请求后，服务器保持连接，并使用 `Content-Type: text/event-stream` 持续返回文本事件；每个事件由字段行组成，事件之间用空行分隔。

浏览器可以使用 `EventSource` 接收 SSE，并在连接中断后自动重连。SSE 只支持服务器向客户端推送，用户提交问题、取消任务等客户端操作仍通过普通 HTTP 请求完成。因此，它适合模型文本和 Agent 进度这类以服务端输出为主的场景。

一次 SSE 响应可以持续发送不同类型的事件：

```text
event: generation.started
data: {"generation_id":"G-2048"}

event: message.delta
data: {"generation_id":"G-2048","sequence":1,"text":"你的订单已经支付，"}

event: message.delta
data: {"generation_id":"G-2048","sequence":2,"text":"目前正在等待快递揽收。"}

event: message.completed
data: {"generation_id":"G-2048","message_id":"M-302"}
```

[事件类型和序号]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#sse-event-schema)使客户端能够区分片段、完成、失败和取消，并发现缺失与重复。用户消息仍可通过普通 POST 提交，服务器到浏览器的进度使用 SSE；只有产品需要频繁双向实时控制时，才需要考虑 WebSocket。

协议选择来自交互方向，不来自“AI 应用必须实时”。

## 文本片段不是最终回答

模型可能在输出“你的订单已经支付，但由于——”时超时或被用户取消。半句话适合当前页面临时展示，不应被其他代码当成已完成回答。

后端应区分：

```text
Delta：当前连接中的即时片段
Draft：后端已经收到但尚未提交的内容
Final Message：正常结束并通过检查后的最终结果
```

收到模型完成信号后，后端合并片段，检查长度和必要的结构化内容，再在同一个数据库事务中写入：

```text
最终消息记录
├─ message_id
├─ conversation_id
├─ generation_id
├─ 完整回答正文
├─ 模型与版本
└─ completed_at

生成记录
├─ status = completed
└─ final_message_id = message_id
```

高频 Token Delta 通常不逐条写入数据库；需要断线恢复时，可以定期保存草稿快照或当前序号。最终消息和完成状态提交成功后，服务端才发送 `message.completed`。

如果先通知完成，随后最终消息或生成状态写入失败，用户刷新页面就找不到刚才的回答，系统也无法确认这次生成是否真正完成。完成事件必须指向一个已经能够通过 `message_id` 或 `generation_id` 重新读取的结果。

这里保存的是**一次模型生成及其消息交付状态**，不等于完整的 [Agent 任务持久状态]({{ site.baseurl }}/docs/backend/llm-backend/agent-task-state/)。简单问答可能只有一次 Generation 和一条最终消息；一项 Agent Task 则可能包含多次模型 Generation、多个 Tool Call、用户确认和等待状态：

```text
Task T-301
└─ Run R-1
   ├─ Step S-1：模型生成 G-1，决定查询订单
   ├─ Step S-2：执行 Tool Call TC-1
   └─ Step S-3：模型生成 G-2，产生最终消息 M-302
```

因此，`message.completed` 只说明一条消息已经保存，不能证明整个 Agent 任务已经完成。Agent 还要检查工具副作用、待确认步骤和任务完成条件，才能提交 `task.completed`。具体实现也可以不单独建立 Generation 表，而是把模型调用记录为一种 Step；关键是只保留一个权威状态来源，避免 Generation 和 Step 各自维护互相冲突的完成状态。

## 用户取消和连接断开不是一回事

用户点击“停止”表达了明确意图：后端检查生成属于当前用户，保存取消状态，再尽力停止上游模型调用。已经产生的 Token 和已经发送的片段无法收回，重复取消应返回同一最终状态。

[浏览器断线不等于任务取消]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#disconnect-vs-cancel)。断线只说明当前订阅消失，可能来自切换网络或刷新页面。产品可以为短问答选择断线即取消，也可以让长任务继续；这项策略必须显式定义，不能偶然取决于哪个进程持有 Socket。

当任务独立继续时，传输通道与任务生命周期需要分离（[轮询、回调和 SSE 的选择]({{ site.baseurl }}/docs/interview/backend/api-application/#polling-webhook-sse)）：

```text
后台执行拥有任务生命周期
SSE 连接只订阅进度
用户取消才改变任务意图
```

这条区分对 Agent 更重要，因为一次任务可能运行数分钟并执行多个工具。

## 重连要补回可恢复的进度

SSE 不保证客户端收到断线期间的事件。[重连恢复]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#sse-reconnect-replay)需要使用稳定事件 ID 或序号，客户端携带最后位置重新订阅：

```http
GET /api/generations/G-2048/events?after=17
```

后端补回序号 18 之后的持久事件，再继续发送新事件。高频文本 Delta 不一定全部长期保存，也可以在重连时返回当前草稿快照；关键是不能重复拼接，也不能缺少中间内容却假装完整。

生成已经完成时，客户端直接读取最终消息，不需要重放所有 Delta。

## Agent 返回的是任务事件，不只是文字

单次模型回答主要产生文本片段。Agent 还会查询订单、等待确认和执行工具，前端需要看到任务级事件：

```text
task.started
tool.requested
tool.completed
confirmation.required
message.delta
task.completed
```

这些事件描述程序确认的执行状态。模型生成“我正在修改地址”只是一段文字，不能代替 `tool.completed`。

任务事件适合持久化和断线续传；Token Delta 数量大、价值短暂，可以只保留草稿与最终结果。事件流是观察任务的通道，不是任务本身的权威状态。

## 失败要保留发生在哪个阶段

身份查询失败、模型限流、流传输中断和最终消息保存失败具有不同含义。客户端只需要稳定错误码和是否允许重试，供应商请求 ID、模型版本、已接收 Token 和内部异常进入日志与 Trace。

新的尝试应有独立 `attempt_id`。它是从头生成一份新回答、继续后台任务，还是只恢复事件订阅，必须由后端状态决定，不能把新输出默认追加到旧草稿。

长连接还会改变容量：服务同时持有更多 SSE 订阅和模型调用，网关需要合适的空闲超时，客户端离开后订阅资源要及时释放。并发限制应针对真正昂贵的生成任务，而不只是 Socket 数量。

## 从流式回答到可恢复任务

一次流式交付可以压缩为：

```text
创建稳定生成或任务 ID
→ 通过事件发送临时进度
→ 区分片段、草稿和最终结果
→ 明确取消与断线的语义
→ 先持久化最终状态，再发送完成事件
→ 重连时从持久状态恢复
```

流式传输解决用户怎样看到过程；状态提交解决系统怎样确认结果。模型开始根据中间结果调用工具后，这个过程会从一次生成扩展为多轮 Agent 执行。
