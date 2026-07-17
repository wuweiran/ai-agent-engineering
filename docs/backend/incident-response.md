---
layout: default
title: 监控与故障处理
parent: 后端工程
nav_order: 10
permalink: /docs/backend/incident-response/
---

# 监控与故障处理

线上运行可以分成三个环节：**监控**持续观察业务和系统指标并触发告警；**问题排查**使用指标、日志和 Trace 缩小范围、定位原因；**故障处理**还包括止损、恢复、历史状态清理和复盘。

可观测性是支撑排查的系统能力，强调能否根据外部输出解释内部状态。它通常由指标、日志和 Trace 组成，不等同于监控，也不是故障处理的完整流程。

运动鞋促销开始二十分钟后，客服收到反馈：订单已经创建，支付页面却长时间显示“正在确认”。HTTP 成功率仍然正常，支付服务的平均延迟也没有明显变化。

值班工程师发现，处于 `payment_confirming` 的订单持续增加，少数支付请求需要十几秒。最终原因不是支付服务处理变慢，而是新配置允许过多并发请求争抢订单服务的支付连接池。

以下是一个模拟故障。处理从用户未完成的业务结果出发，再用指标、日志和 Trace 找到第一个异常位置。

## 用户结果与影响范围

“支付系统有问题”还不能指导行动。值班工程师先[确定影响范围]({{ site.baseurl }}/docs/interview/backend/performance-production/#incident-impact-scope)，把影响描述成可以验证的事实：

```text
开始时间：10:20
受影响流程：创建订单后的支付确认
影响范围：约 18%，集中在支付方式 W
用户结果：长时间停在“正在确认”，无法继续支付
当前风险：订单占用库存，暂未发现重复扣款
```

[接口返回 200 也可能代表业务失败]({{ site.baseurl }}/docs/interview/backend/performance-production/#business-failure-with-http-success)：它只表示协议返回了“处理中”，用户仍然没有完成支付。事故严重程度应由交易完成率、受影响用户和资金风险决定，而不是由 500 数量决定。

## 业务指标、状态指标与系统指标

支付完成率从 82% 降到 61%，`payment_confirming` 数量持续增长，最老订单等待时间不断增加。这些指标说明问题正在影响哪段业务。

系统指标进一步显示，订单服务取得支付连接的 99 分位等待时间从 20 毫秒升到 1.8 秒。平均延迟变化不大，因为多数请求仍然很快，少量极慢请求被平均值稀释。

业务指标和系统指标回答不同问题：

```text
用户结果：交易是否完成
业务状态：工作积压在哪个阶段
系统资源：请求卡在哪个组件
```

[告警]({{ site.baseurl }}/docs/interview/backend/performance-production/#alert-design)应指向需要采取行动的变化。CPU 短暂升高而用户结果正常，可以观察；支付完成率下降或中间状态最老年龄超过承诺期限，需要立即处理。

## 资源饱和的定位方法

CPU、内存和磁盘告警都只说明某项资源异常，排查顺序仍然是：先确认用户影响和异常开始时间，再定位节点、容器和进程，最后进入线程、对象或文件等具体现场。

| 资源 | 先区分什么 | 继续定位什么 |
| --- | --- | --- |
| CPU | 用户态、系统态还是 I/O Wait；单核还是整体 | 高 CPU 进程、线程以及调用栈或性能剖析 |
| 内存 | 进程内存还是文件缓存；持续增长还是流量峰值 | Java 堆、直接内存、Metaspace、线程栈或本地进程内存 |
| 磁盘 | 磁盘空间、inode 用尽还是 I/O 饱和 | 大目录、已删除仍被占用的文件、读写进程、设备延迟与队列 |

常见命令只是取得证据的入口：`top`、`pidstat` 查看进程与线程；`free`、`vmstat` 查看内存和换页；`df`、`du`、`lsof` 查看空间；`iostat` 查看磁盘吞吐、延迟和队列。容器环境还要同时检查容器 Limit 与宿主机资源。

资源恢复后仍要回到业务指标确认用户结果，并结合发布、配置和流量变化验证根因。CPU 很高不一定要直接扩容，内存很高不一定是泄漏，磁盘告警也要先区分空间不足和 I/O 变慢。

相关问题：[CPU 占用过高怎样排查？]({{ site.baseurl }}/docs/interview/backend/performance-production/#high-cpu-troubleshooting)、[内存占用过高怎样排查？]({{ site.baseurl }}/docs/interview/backend/performance-production/#high-memory-troubleshooting)、[磁盘异常怎样排查？]({{ site.baseurl }}/docs/interview/backend/performance-production/#high-disk-troubleshooting)。

## 结构化日志还原单笔业务

工程师从受影响订单中选择 `O-1042`，按稳定标识查到结构化日志：

```json
{
  "request_id": "REQ-301",
  "order_id": "O-1042",
  "payment_request_id": "PAYREQ-7001",
  "operation": "create_payment",
  "dependency": "payment-service",
  "duration_ms": 2000,
  "result": "timeout"
}
```

[结构化日志]({{ site.baseurl }}/docs/interview/backend/performance-production/#structured-logging)中的稳定字段可以按支付方式、服务版本和错误类别聚合。日志中的“timeout”仍不能证明支付失败，它只说明这次调用没有及时得到结果。

访问令牌、银行卡等敏感信息不能为了排查写入日志。订单 ID 和支付请求 ID 足以让有权限的人回到权威系统查询。

## Trace 定位跨组件耗时

异常请求的 [Trace]({{ site.baseurl }}/docs/interview/backend/performance-production/#distributed-tracing) 显示。Trace 是一次请求跨组件执行形成的分布式调用链：

```text
POST /orders/O-1042/pay            2.4s
├─ validate_order                  12ms
├─ acquire_payment_connection      1.8s
├─ POST payment-service/payments   580ms
└─ save_payment_confirming         8ms
```

支付服务只处理了 580 毫秒，主要时间花在订单服务等待连接。扩容支付服务不会解决这个问题。

对比正常请求和版本信息后，工程师发现新配置把每实例支付并发从 20 提高到 100，连接池仍只有 30。请求开始排队，超时又触发确认查询，确认任务继续争抢同一个连接池，形成自我放大。

[指标、日志和 Trace]({{ site.baseurl }}/docs/interview/backend/performance-production/#metrics-logs-traces)分别找到变化范围、还原具体结果并拆开一次调用的时间；版本和配置说明为什么只有部分实例出现异常。三者要围绕同一笔业务关联，而不是分别浏览仪表盘。

## 故障止损

连接池正在耗尽，团队没有必要等完整调查报告才行动。事故发生后应[先限制新影响]({{ site.baseurl }}/docs/interview/backend/performance-production/#incident-first-response)：暂停扩大新版本流量，把支付并发恢复到已验证值，限制新的确认查询，并停掉占用同一资源的非核心任务。

页面明确显示支付繁忙，订单和原 `payment_request_id` 被保留。不能为了让积压数字下降，就把 `payment_confirming` 全部标记失败或重新创建支付，那会破坏仍然未知的资金结果。

这些动作可逆，并且直接保护用户结果和权威状态。

**事故处理先停止影响扩大，再在稳定环境中确认根因。**

## 根因验证

团队在少量实例恢复旧并发配置，保持连接池和测试流量不变。连接等待迅速下降，新订单不再大量进入 `payment_confirming`。重新应用高并发配置后问题可复现，假设得到证据支持。

如果同时扩容连接池、修改超时、关闭重试并升级 SDK，即使指标恢复，也不知道哪项真正有效。[单一变量的对照]({{ site.baseurl }}/docs/interview/backend/performance-production/#root-cause-verification)能把相关性推进为可重复的因果证据。

## 流量恢复与历史状态清理

回滚配置只阻止新问题继续发生。事故期间已经积累的订单可能有三种真实结果：支付单已经创建、明确没有创建、仍然无法确认。

恢复任务使用原 `PAYREQ-7001` 查询支付服务：找到支付单就保存同一个 `payment_id`；明确不存在才重新创建；仍然未知则继续退避等待或进入人工核对。

积压不能一次性全量查询，否则会让刚恢复的依赖再次过载。清理速度要低于剩余容量，同时观察新流量和历史任务。

只有新请求尾部延迟恢复、中间状态不再增长、历史积压持续下降、支付完成率回升，并且抽查没有重复扣款，[事故才真正结束]({{ site.baseurl }}/docs/interview/backend/performance-production/#incident-business-recovery)。底层资源图变绿通常早于用户结果恢复。

## 故障复盘与系统改进

这次事故暴露出支付并发与连接池容量没有共同校验，确认查询和创建支付又共享资源。长期修复应让同类错误更难发生、影响更早被发现、恢复更快执行：配置需要版本和灰度，两个路径使用独立预算，中间状态数量和最老年龄触发告警，恢复动作可以被演练。

[事故复盘]({{ site.baseurl }}/docs/interview/backend/performance-production/#incident-postmortem)不是列出“加强监控、谨慎操作”。个人配置错误能够轻易影响全部流量，说明发布和容量边界本身缺少保护。

## 线上故障处理流程

事故处理沿着业务因果链推进：

```text
用户没有得到什么结果？
        ↓
指标显示影响从何时、在哪个阶段扩大？
        ↓
日志和 Trace 把异常定位到哪个具体等待或错误？
        ↓
什么可逆动作能够先限制新影响？
        ↓
怎样验证根因并恢复新流量？
        ↓
哪些已有业务状态仍要清理？
```

观测不是为了收集更多数据，而是为了支持判断和行动。生产系统恢复的标准，是新交易重新完成、历史状态得到处置，并且同类故障增加了更早的防线。
