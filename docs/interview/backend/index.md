---
layout: default
title: 后端面试题
parent: 面试题库
nav_order: 1
has_children: true
permalink: /docs/interview/backend/
---

# 后端面试题

后端面试既会考察 Java、JVM、事务、索引和缓存等基础概念，也会通过超时、并发、消息重复和系统过载等场景判断工程能力。

[后端工程]({{ site.baseurl }}/docs/backend/)正文沿一次业务结果怎样产生、保存和跨越故障展开；这里换成面试视角，把常见术语放回同一条系统链路，帮助读者看清它们为什么会一起出现。

## 后端面试关键词图谱

```text
请求进入系统
→ 进程内执行业务逻辑
→ 数据库提交权威状态
→ 缓存和异步机制分担压力
→ 多服务、多节点共同推进业务
→ 在容量、故障和版本变化中持续运行
→ 大模型作为新的不确定计算环节接入同一套边界
```

- **请求入口与安全边界**
  - **[API 与应用层]({{ site.baseurl }}/docs/interview/backend/api-application/)**：REST、HTTP 方法与状态码、入参校验、业务校验、分页、Idempotency-Key、批量接口、`202 Accepted`、轮询、回调、SSE、API 版本兼容
  - **[后端安全与鉴权]({{ site.baseurl }}/docs/interview/backend/security-auth/)**：认证、授权与鉴权，Session 与 JWT、Access Token 与 Refresh Token、OAuth 2.0、OIDC、PKCE、RBAC、ABAC、对象级授权、多租户隔离、SQL 注入、XSS、CSRF、CORS、防重放
  - 这一层回答：**谁在请求、能操作什么、输入是否合法、重复请求是否得到同一结果。**

- **进程内执行与 Java 运行时**
  - **[Java 技术栈]({{ site.baseurl }}/docs/interview/backend/java-jvm/)**：集合、HashMap、ConcurrentHashMap、泛型、异常、进程/线程/协程、线程池参数与拒绝策略、`synchronized`、Lock、volatile、CAS、ABA、ThreadLocal、JMM、happens-before
  - **Spring**：IoC、Bean、AOP、`@Transactional`、事务传播、Spring Boot 自动配置
  - **JVM**：类加载与双亲委派、运行时内存、GC Roots、垃圾回收算法、G1、ZGC、Shenandoah、CPU 过高、Full GC、OOM
  - 这一层回答：**一个服务实例怎样组织代码、控制并发并管理运行资源。**

- **权威状态与数据库**
  - **[数据库与 MySQL]({{ site.baseurl }}/docs/interview/backend/database/)**：事务、ACID、隔离级别、脏读、不可重复读、幻读、MVCC、乐观锁、悲观锁、死锁、超卖
  - **查询与索引**：B+ 树、联合索引、最左前缀、覆盖索引、索引失效、执行计划、N+1 查询
  - **MySQL 专项**：InnoDB、聚簇索引、二级索引回表、当前读、快照读、binlog、redo log、undo log
  - 这一层回答：**哪些数据是事实，以及并发和故障发生时怎样守住这些事实。**

- **读取加速与 Redis**
  - **[缓存与 Redis]({{ site.baseurl }}/docs/interview/backend/cache/)**：Cache Aside、缓存一致性、TTL、穿透、击穿、雪崩、缓存预热、大 Key、热 Key
  - **Redis 专项**：String、Hash、List、Set、Sorted Set，过期删除策略、RDB、AOF
  - 缓存保存的是可重建副本，不是新的权威状态；缓存故障时还要保护数据库。

- **跨越请求的异步工作与 Kafka**
  - **[消息队列与 Kafka]({{ site.baseurl }}/docs/interview/backend/message-queue/)**：异步、削峰、解耦，At Most Once、At Least Once、消费幂等、消息顺序、延迟消息、本地消息表、死信队列、消息积压
  - **Kafka 专项**：Broker、Topic、Partition、Replica、Partition Key、Consumer Group、Rebalance、Offset、Leader、Follower、ISR、`acks`、`min.insync.replicas`、幂等生产者、Kafka 事务
  - 这一层回答：**请求结束后，未完成工作怎样继续，以及消息重复、丢失、乱序时怎样恢复。**

- **跨服务与多节点协调**
  - **[分布式系统]({{ site.baseurl }}/docs/interview/backend/distributed-system/)**：服务发现、负载均衡、CAP、BASE、超时不等于失败、Deadline、重试、指数退避、随机抖动、重试放大、熔断、降级、线程池隔离、信号量隔离
  - **分布式状态**：2PC / XA、TCC、Saga、最终一致性、状态机、分布式锁、租约、Fencing Token、全局唯一 ID
  - **流量控制**：固定窗口、滑动窗口、漏桶、令牌桶、本地配额、共享配额
  - 这一层回答：**没有一个本地事务覆盖全链路时，各节点依据什么继续、重试、补偿或停止。**

- **容量、故障与系统演进**
  - **[性能与生产运行]({{ site.baseurl }}/docs/interview/backend/performance-production/)**：QPS、吞吐量、平均延迟、P95、P99、利用率与排队、线程池、连接池、背压、容量规划
  - **监控与排障**：四个黄金信号、RED、USE、结构化日志、Trace、CPU、内存、磁盘、止损、根因验证、恢复与复盘
  - **发布演进**：灰度发布、蓝绿发布、滚动发布、Schema 兼容、扩展—迁移—收缩、配置变更、回滚与业务状态
  - 这一层回答：**系统在高负载、依赖故障和新旧版本并存时，怎样限制影响并恢复服务。**

- **大模型应用后端**
  - **[大模型应用后端]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/)**：流式生成、流式返回、Token Delta、草稿、最终消息、SSE 与 WebSocket、断线重连
  - **Agent 执行**：Tool Call、Schema 校验、业务校验、Prompt Injection、确认、幂等、结果未知
  - **任务状态**：Session、Conversation、Task、Run、Step、Generation、Tool Call、Event、Artifact、Worker 恢复、租约、取消与补偿
  - **生产运行**：Context、Token、并发预算、模型限流、成本、可观测性、模型与 Prompt 版本、评测与安全发布
  - 大模型改变了计算方式，但认证、授权、权威状态、业务幂等、故障恢复和容量控制仍属于同一套后端责任。

## 贯穿图谱的工程关键词

有些概念不会只属于一个页面，它们是连接整套后端知识的主线：

- **权威状态**连接数据库、缓存、消息和 Agent Task：副本、事件和模型文本都不能代替业务事实。
- **幂等**连接 API、消息消费、分布式重试和 Tool Call：稳定业务身份决定重复执行是否安全。
- **结果未知**连接远程调用、支付、消息确认和 Agent 工具：超时只说明没有收到答案，不自动等于失败。
- **状态机**连接订单、异步任务、Saga 和 Agent Task：恢复必须依据持久状态，而不是重新猜测历史。
- **容量边界**连接线程池、连接池、队列积压、限流和模型并发：等待会占用资源，重试还会放大压力。
- **版本**连接 API、数据库 Schema、消息、模型、Prompt 和工具：发布完成不代表旧数据与旧任务已经消失。
