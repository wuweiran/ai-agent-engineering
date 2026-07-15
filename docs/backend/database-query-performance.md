---
layout: default
title: 数据库查询性能
parent: 后端工程
nav_order: 3
permalink: /docs/backend/database-query-performance/
---

# 数据库查询性能

订单系统刚上线时只有几万条记录，用户查询待支付订单几乎立即返回。一年后订单表增长到数亿行，同一接口开始需要几秒，数据库 CPU 和磁盘读取也持续升高。

代码没有变，查询表达的业务问题也没有变。变化的是数据库为了找到答案需要检查的数据量。

**SQL 描述想得到什么，执行计划决定数据库怎样找到它。**

## 一条查询实际要做多少工作

用户查看最近的待支付订单：

```sql
SELECT order_id, status, total_amount_minor, created_at
FROM orders
WHERE buyer_id = 'U-204'
  AND status = 'pending_payment'
ORDER BY created_at DESC
LIMIT 20;
```

没有合适索引时，数据库可能扫描大量订单，逐行判断买家和状态，再把结果排序后取前 20 条。最终只返回 20 行，不代表只读取了 20 行（[为什么查询只返回 20 行仍可能很慢？]({{ site.baseurl }}/docs/interview/backend/database/#small-result-slow-query)）。

查询速度首先取决于访问路径：数据库为了产生结果，扫描了多少行、读取了多少数据页、进行了多少次排序和随机访问。

数据量较小时，全表扫描可能很快，所以性能问题常在业务增长后才暴露。不能只用开发环境中的少量数据判断生产查询成本。

## 索引缩小查找范围

可以为常见访问方式建立联合索引：

```sql
CREATE INDEX idx_orders_buyer_status_created
ON orders (buyer_id, status, created_at DESC);
```

这份索引先按买家组织，再按状态区分，并在同一范围内按创建时间排列。数据库可以直接定位 `U-204 + pending_payment` 的最新位置，顺序取出 20 条，不必扫描整张订单表再排序。

联合索引不是把几个常用字段随意放在一起。字段顺序要匹配查询怎样逐步缩小范围：

```text
先确定一个买家
再确定订单状态
最后按时间取最近记录
```

若查询只按 `created_at` 搜索全站最近订单，这份索引未必合适；若经常只按 `buyer_id` 查询，它通常仍能利用[最左侧前缀]({{ site.baseurl }}/docs/interview/backend/database/#composite-index-leftmost-prefix)。索引设计来自真实访问模式，不能为每个字段各建一份后期待数据库自动组合出最佳路径。

## 执行计划让判断有证据

工程师不应只凭 SQL 外观猜测性能。数据库的 `EXPLAIN` 或执行计划会显示：

- 选择了全表扫描还是某个索引；
- 预计和实际读取了多少行；
- 过滤发生在索引阶段还是读取数据以后；
- 是否进行了额外排序、临时表或大规模聚合；
- 多表连接采用什么顺序和算法；
- 时间主要花在哪个算子。

一条查询有索引却仍然慢，可能因为条件匹配了表中大部分数据，优化器认为顺序扫描更便宜；也可能因为统计信息过旧，估算行数与实际差异很大（[为什么查询可能不使用索引？]({{ site.baseurl }}/docs/interview/backend/database/#index-not-used)）。

排查慢查询要把 SQL、参数、数据分布和[实际执行计划]({{ site.baseurl }}/docs/interview/backend/database/#query-execution-plan)放在一起。只看平均耗时会忽略某些租户或时间范围触发的坏计划。

## 索引也有成本

[索引]({{ site.baseurl }}/docs/interview/backend/database/#database-index)是额外的数据结构。创建订单、修改状态时，数据库除了写业务表，还要更新相关索引。索引越多：

- 写入需要维护的结构越多；
- 占用的内存和磁盘越大；
- 缓存中更难保留真正热门的数据页；
- Schema 迁移和重建耗时越长。

因此，索引不是越多越好。它用写入和存储成本换取特定读取路径的速度。长期无人使用、与其他索引重复的结构应该通过监控确认后清理。

## 返回太多数据会抵消索引收益

即使定位很快，接口一次读取十万条订单，数据库、网络和应用内存仍要处理十万条结果。查询应该只选择当前页面需要的字段，并使用稳定分页。

[[大偏移分页]({{ site.baseurl }}/docs/interview/backend/api-application/#offset-vs-cursor-pagination)]({{ site.baseurl }}/docs/interview/backend/database/#deep-pagination)常写成：

```sql
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;
```

数据库通常仍要走过前面十万条记录才能丢弃它们。更稳定的方式是使用上一页最后一条记录作为游标：

```sql
WHERE buyer_id = 'U-204'
  AND status = 'completed'
  AND (created_at, order_id) < (:last_created_at, :last_order_id)
ORDER BY created_at DESC, order_id DESC
LIMIT 20;
```

`order_id` 作为相同时间下的稳定排序补充。游标分页让每页从明确位置继续，成本不会随着页码线性增加。

## N+1 查询把一次列表变成大量往返

订单列表先查 20 张订单，代码随后为每张订单分别查询商品明细：

```text
1 次查询订单列表
20 次查询订单明细
```

这称为 [N+1 查询]({{ site.baseurl }}/docs/interview/backend/database/#n-plus-one-query)。单条 SQL 都很快，总延迟却包含大量数据库往返，连接池也会被占用。

修复方式取决于数据关系：可以批量使用 `WHERE order_id IN (...)` 查询明细，也可以在合适时连接查询或预加载。目标不是追求“一条 SQL 完成所有事情”，而是避免调用次数随着结果条数无意增长。

ORM 能简化数据访问，也可能把属性访问悄悄变成查询。工程师仍要观察一次 API 实际执行了多少 SQL。

## 慢查询也可能在等待

执行计划良好，查询仍可能慢，因为时间花在取得连接、等待锁或读取冷数据，而不是执行 SQL 本身。

一次数据库调用应区分：

```text
等待连接池
+ 数据库排队或锁等待
+ SQL 执行
+ 结果传输与反序列化
```

只记录“DAO 耗时 2 秒”无法判断应该增加索引、缩短事务，还是限制连接并发。长事务持有锁、应用扩容带来过多连接，都可能让本来快速的查询排队（[执行计划正常，查询为什么仍可能很慢？]({{ site.baseurl }}/docs/interview/backend/database/#slow-query-with-good-plan)）。

## 数据增长后的演进

索引和查询优化应先解决已经观察到的访问成本。只有单表规模、写入吞吐、保留期限或隔离要求形成真实限制时，才考虑更大的变化：归档历史数据、按时间分区、建立只读副本，或把分析负载移出交易数据库。

这些机制不会修复低效查询。把全表扫描搬到分区表上，如果查询仍然命中所有分区，成本只是换了表现形式。

查询性能可以用一条主线复述：

```text
业务问题
→ SQL 与参数
→ 数据库选择访问路径
→ 扫描、连接、排序和等待形成成本
→ 用执行计划找到主要工作
→ 通过索引、查询形状和数据组织减少工作量
```

专家不会先问“要不要加索引”，而会先问数据库为了返回这些行究竟做了多少工作，以及这份工作为什么随数据增长。
