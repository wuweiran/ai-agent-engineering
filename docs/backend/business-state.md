---
layout: default
title: 业务状态与并发控制
parent: 后端工程
nav_order: 2
permalink: /docs/backend/business-state/
---

# 业务状态与并发控制

运动鞋 `SKU-8812` 只剩一件。小林和小周几乎同时打开商品页面，两个人都看到“有货”，随后分别提交订单。

每个请求都可能先读到库存为 1，再各自认为可以购买。问题不是某条 SQL 写错了，而是两个请求共同修改同一项事实时，系统没有把“检查”和“写入”放在不可分割的边界中。

## 数据库保存系统相信的事实

订单系统至少需要表达三类事实：

```text
orders
├─ order_id
├─ buyer_id
├─ status
├─ total_amount_minor
├─ idempotency_key
└─ version

order_items
├─ order_id
├─ sku_id
├─ quantity
├─ unit_price_minor
└─ discount_amount_minor

sku_inventory
├─ sku_id
├─ available_stock
└─ version
```

订单明细保存成交时的商品和价格快照，库存表保存当前还能出售多少件。页面、缓存、日志和消息可以复制这些信息，却不能各自决定交易结果。

设计数据模型时要先区分：哪些字段描述现在，哪些字段保留过去发生的事实。商品以后调价，历史订单不能跟着变化；库存被其他订单扣减，当前可售数量则必须变化。

## 约束让错误状态无法轻易写入

应用代码会检查订单金额、商品数量和幂等键，数据库也应守住不会因入口不同而改变的底线：主键提供唯一身份，非空与检查约束守住必要字段和数值范围，唯一约束使同一业务意图只产生一份结果，外键在同一数据边界内保护引用关系。

“优惠券是否适用于当前商品”属于会演进的业务规则；“库存不能小于零”是任何写入路径都不能破坏的约束。

**越接近最终数据、越必须始终成立的规则，越应该让存储层共同执行。**

## 先查再改的并发窗口

直观代码通常写成：

```text
读取库存
如果库存大于零
    库存减一
    创建订单
```

两个请求可能这样交错：

```text
小林读取 1
小周读取 1
小林写入 0
小周也写入 0
两边各自创建订单
```

最终库存没有变成负数，却产生了两张都声称买到最后一件商品的订单。读取结果在真正修改时已经过期。

数据库应在写入发生的同一刻重新判断条件：

```sql
UPDATE sku_inventory
SET available_stock = available_stock - 1,
    version = version + 1
WHERE sku_id = 'SKU-8812'
  AND available_stock >= 1;
```

受影响一行，当前请求取得库存；更新零行，说明库存已经被别的请求拿走。页面几秒前显示“有货”只是当时的读取结果（[如何防止超卖？]({{ site.baseurl }}/docs/interview/backend/database/#prevent-overselling)）。

**会被并发改变的条件，必须在写入时再次确认。**

## 事务让共同成立的事实一起提交

扣减库存以后还要写订单和明细。如果库存成功减少，订单写入却失败，商品会无故少一件；如果订单先创建，库存扣减失败，又会留下无法履约的订单。

当这些数据位于同一个数据库并共同构成“下单成功”时，可以放进一个[数据库事务]({{ site.baseurl }}/docs/interview/backend/database/#database-transaction)：

```text
开始事务
├─ 条件扣减库存，确认更新了一行
├─ 写入订单主记录
└─ 写入成交明细
全部成功后提交
```

任何一步失败，事务回滚。事务边界来自业务承诺：只有订单记录和库存资格同时成立，系统才能返回下单成功。

事务不能自动覆盖支付平台、短信服务等数据库外部系统。跨出本地事务后，需要稳定请求 ID、状态查询、消息和补偿处理结果未知，而不是无限延长数据库事务（[为什么不应在事务中等待远程调用？]({{ site.baseurl }}/docs/interview/backend/database/#remote-call-inside-transaction)）。

## 版本号防止修改互相覆盖

待支付订单允许修改地址。用户在手机和电脑上同时打开订单，两边都读取到 `version = 7`。手机先提交新地址：

```sql
UPDATE orders
SET shipping_address_id = 'ADDR-73',
    version = version + 1
WHERE order_id = 'O-1042'
  AND buyer_id = 'U-204'
  AND status = 'pending_payment'
  AND version = 7;
```

成功后版本变成 8。电脑仍带版本 7 提交旧页面中的地址，更新零行。服务要求客户端读取最新订单，而不是覆盖手机刚完成的修改（[怎样避免并发修改互相覆盖？]({{ site.baseurl }}/docs/interview/backend/database/#prevent-lost-update)）。

这称为[乐观并发控制]({{ site.baseurl }}/docs/interview/backend/database/#optimistic-vs-pessimistic-locking)：读取时取得版本，写入时确认期间没有其他修改。它适合冲突不频繁、冲突后可以刷新重试的场景。大量请求长期争用同一资源时，锁、串行处理或重新设计业务竞争方式可能更合适。

## 状态转换也要带着前置条件

订单状态不是可以任意赋值的字符串：

```text
pending_payment → paid → shipped → completed
        └──────→ cancelled
```

取消只能处理仍待支付的订单。更新时把前置状态写入条件：

```sql
UPDATE orders
SET status = 'cancelled',
    version = version + 1
WHERE order_id = 'O-1042'
  AND buyer_id = 'U-204'
  AND status = 'pending_payment';
```

如果支付回调已经把订单改成 `paid`，取消请求更新零行。业务层读取最新状态，再决定拒绝取消还是进入退款流程。

先读取状态、在内存判断、随后无条件写入，会重新打开并发窗口。合法状态路径应由业务代码集中定义，关键前置条件还要落实到最终写入。

## 权威状态决定争议时向哪里确认

下单成功后，订单数据库中的记录和库存数据库中的可售数量是当前权威状态。商品页缓存、应用日志和队列事件可能延迟、重复或丢失，只能作为副本和线索。

大型系统可能由不同服务分别掌握事实：订单服务掌握订单状态，库存服务掌握可售数量，支付服务掌握真实支付结果。其他组件通过稳定 ID 引用这些事实，不能维护一套可以独立修改的最终答案。

判断一个设计时，应能明确回答：

```text
系统最终相信哪份数据？
哪些状态绝不能出现？
哪些修改必须共同提交？
并发写入时，条件在哪里重新确认？
```

数据库的核心作用不只是保存字段，而是通过约束、条件写入和事务，使多个请求最终得到一组能够共同成立的业务事实。状态复制到多个节点，或者一项业务分散到多个服务以后，还需要进一步处理[分布式系统中的状态协调]({{ site.baseurl }}/docs/backend/distributed-state-coordination/)。
