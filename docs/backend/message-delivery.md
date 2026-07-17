---
layout: default
title: 可靠消息的发送与消费
parent: 后端工程
nav_order: 5
permalink: /docs/backend/message-delivery/
---

# 可靠消息的发送与消费

订单 `O-1042` 创建后，订单服务要安排一个“到期仍未支付就取消订单”的异步任务。它不直接寻找某个消费者，而是先把任务消息发送到消息队列：

```text
订单服务：产生异步任务
→ 消息中间件（Kafka / RabbitMQ）：接收并保存消息
→ 消费者：读取消息并执行取消订单逻辑
```

消息中间件负责接收、保存和投递消息，Kafka、RabbitMQ 都可以承担这个角色。消费者是接收消息并执行业务逻辑的后台执行者。

“发送消息”和“接收消息”是逻辑角色，不一定对应两个独立服务。同一个订单服务可以在处理 HTTP 请求时发送消息，也可以在后台运行消费者接收其他消息；消息中间件负责在两者之间保存待处理工作。

第一种故障发生在生产端：订单数据库已经提交，订单服务却在消息发给消息中间件前崩溃，用户看得到订单，队列里没有后续任务。另一种故障发生在消费端：消费者已经取消订单，却在向消息中间件确认消费成功前重启，消息中间件只能再次投递同一条消息。

消息链路因此跨越订单数据库、消息中间件和消费者。跨越这些系统时，没有一个普通本地事务能让所有参与者共同知道“业务完成并且消息已经确认”。

## 业务提交和消息发布之间存在空隙

先提交订单、再发布消息：

```text
提交订单事务
    ↓
发布 order.expiration_scheduled
```

服务可能在两步之间崩溃。交换顺序也不安全：消息先发出而订单事务随后失败，消费者会收到一个并未成立的业务事实。

问题不是数据库或消息队列不可靠，而是它们各自提交，普通代码无法让两个系统原子地共同成功（[数据库已提交，消息发送失败怎么办？]({{ site.baseurl }}/docs/interview/backend/message-queue/#database-committed-message-failed)）。这个故障窗口也是[分布式系统中的状态协调]({{ site.baseurl }}/docs/backend/distributed-state-coordination/)需要处理的一类跨系统提交问题。

## 本地消息表先把“需要发送”保存成事实

订单服务可以在创建订单的同一事务中写入[本地消息表]({{ site.baseurl }}/docs/interview/backend/message-queue/#transactional-outbox)。这种方案的完整名称是 Transactional Outbox，也常简称 Outbox：

```sql
BEGIN;

INSERT INTO orders (
  order_id, buyer_id, status, reservation_expires_at
) VALUES (
  'O-1042', 'U-204', 'pending_payment', '2026-07-14T10:50:00Z'
);

INSERT INTO outbox_events (
  event_id, event_type, aggregate_id, payload, status
) VALUES (
  'EV-7001',
  'order.expiration_scheduled',
  'O-1042',
  '{"order_id":"O-1042"}',
  'pending'
);

COMMIT;
```

订单和本地消息表必须使用同一个数据库事务。消息表插入失败时，整个事务回滚，订单也不会创建；只有两次写入都成功，事务才提交。因此提交成功后，订单和待发送事件要么同时存在，要么同时不存在。

独立发布进程扫描 `pending` 记录，把事件发送到消息中间件。即使原请求所在实例重启，数据库仍然记得有一条消息尚未发布。

[本地消息表]({{ site.baseurl }}/docs/interview/backend/distributed-system/#outbox-vs-transactional-message)解决的不是“如何更快发消息”，而是：

> **把未来必须完成的发送意图，与产生它的业务事实共同提交。**

## 为了不丢，发布可能重复

发布进程可能完成这三个动作中的前两个：

```text
读取 EV-7001
成功发送到消息中间件
尚未标记 sent 时崩溃
```

重启后它会再次发送 `EV-7001`。如果先标记再发送，又会重新打开消息丢失窗口。

可靠系统通常选择保留稳定的 `event_id` 并允许重复，因为重复还有机会被识别，永久缺失却可能永远无人察觉。

消息中间件的生产者幂等可以减少某些重复，无法替代端到端业务设计。消息跨入另一个数据库或外部服务后，又出现新的确认边界。

## 消费者完成业务后也可能再次收到消息

消费者的处理过程是：

```text
从消息中间件取得消息
→ 修改订单数据库
→ 消费者向消息中间件发送消费确认
```

消费确认表示“这条消息已经处理完成，可以推进消费位置或停止投递”。RabbitMQ 等队列通常称为 ACK，Kafka 中通常表现为提交消费 Offset。

如果消费者已经提交订单数据库，却在消息中间件收到消费确认前崩溃或断网，消息中间件只能看到这条消息仍未确认，无法知道订单数据库已经修改成功。恢复后，消息中间件会把消息再次交给当前或其他消费者，避免一条可能没有处理完成的消息永久丢失。

这就是常见的[至少一次投递]({{ site.baseurl }}/docs/interview/backend/message-queue/#message-delivery-semantics)：消息有机会被处理，但同一事件也可能出现多次。消费者必须让同一业务意图重复到达时产生同一个结果。

## [消费幂等]({{ site.baseurl }}/docs/interview/backend/message-queue/#consumer-idempotency)要落实到业务结果

订单过期事件可以通过状态条件天然去重：

```sql
UPDATE orders
SET status = 'cancelled',
    cancel_reason = 'payment_timeout'
WHERE order_id = 'O-1042'
  AND status = 'pending_payment'
  AND reservation_expires_at <= CURRENT_TIMESTAMP;
```

第一次执行更新一行。重复事件再次到达，订单已经不是 `pending_payment`，更新零行，不会重复取消。

若订单取消和本地库存释放位于同一数据库，它们应在一个事务中完成，并且只有订单状态实际转换成功时才释放库存。正确实现需要根据第一条更新的受影响行数决定是否执行后续写入，不能无条件增加库存。

有些动作没有天然状态门槛。例如消费者收到事件后创建一张站内优惠券，可以对 `consumer_name + event_id` 建立唯一约束，并让优惠券与消费记录在同一事务中提交。

## 外部副作用仍然留下结果未知

发送短信不在本地数据库事务中。服务可能已经发送成功，却在保存消费记录前崩溃。

最理想的接口允许把 `event_id` 作为外部幂等键：重复请求返回同一发送结果。外部服务不支持幂等和结果查询时，系统无法凭空获得“绝不漏发且绝不重复”的保证，只能根据业务风险选择更可接受的一边，并通过人工核对或补偿降低影响。

这也是 [Exactly Once 的边界]({{ site.baseurl }}/docs/interview/backend/message-queue/#exactly-once-boundary)。消息中间件可以在自己的边界内提供很强的生产和消费语义，但不能自动让订单数据库、短信平台和支付系统参加同一个事务。

**业务上只发生一次，是稳定身份、状态条件、唯一约束和结果查询共同形成的效果。**

## 乱序消息不能让状态倒退

同一订单可能产生：

```text
order.created
order.paid
order.cancelled
```

分区、重试和不同生产者可能改变到达顺序。使用 `order_id` 作为分区键可以减少乱序，却不能消除历史消息重放等情况（[消息顺序怎样保证？]({{ site.baseurl }}/docs/interview/backend/message-queue/#message-ordering)）。

事件应携带聚合 ID 和版本，消费者只接受符合当前状态的转换。迟到的 `order.created` 不能把已支付订单退回待支付；缺少必要前置状态时，可以回查权威订单，而不是猜测事件顺序。

## 积压和死信也要连接业务结果

持续无法处理的消息不应无限重试并阻塞后续工作。有限重试后，可以把原事件 ID、消费者版本、错误和业务资源保存到异常任务或[死信队列]({{ site.baseurl }}/docs/interview/backend/message-queue/#dead-letter-queue)，修复后仍使用原身份重新处理。

观察消息链路时，队列长度只是线索。本地消息表最老未发布时间、消息处理延迟和死信数量能说明传输情况，订单是否真的过期取消并释放库存才说明业务承诺是否完成（[怎样判断消息链路恢复？]({{ site.baseurl }}/docs/interview/backend/message-queue/#message-recovery-complete)）。

## 可靠消息的因果链

消息系统之所以同时出现丢失和重复风险，是因为业务写入、发布、消费和确认跨越多个独立提交点：

```text
业务事实 + 本地消息表共同提交
        ↓
发布可能重复，携带稳定事件 ID
        ↓
消费可能重复，用业务幂等守住结果
        ↓
外部结果未知时查询或补偿
```

**可靠消息并不承诺网络永远只交付一次，而是让消息在丢失、重复和重放发生时，业务结果仍然能够确认与恢复。**
