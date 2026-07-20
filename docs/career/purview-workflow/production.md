---
layout: default
title: 部署与生产运行
parent: Microsoft Purview 工作流
grand_parent: 工作经历
nav_order: 4
permalink: /docs/career/purview-workflow/production/
---

# 部署与生产运行

## 服务拓扑

Purview Workflow 后端是运行在 Azure 上的多实例 Scala + ZIO 服务。服务本身保持无状态，实例可以同时处理 Workflow API、Definition 管理、Logic Apps Action 请求和结果回调。工作流的节点状态和调度由 Logic Apps 保存，Purview 只持久化产品层数据和异步 Action 状态。

```text
Purview Catalog 与管理界面
→ 区域入口与负载均衡
→ Scala + ZIO 服务实例
   ├─ Workflow Definition API
   ├─ Logic Apps Action API
   ├─ Expression Parser 与 Evaluator
   ├─ Workflow 概括与生成
   └─ Callback Worker
→ Cosmos DB 与 Azure Blob Storage
→ Azure Logic Apps 与 Azure OpenAI
→ Catalog 与组织信息等下游服务
```

生产环境按 Azure Geography 部署，租户请求进入数据所在区域。每个区域基线运行 **6 个实例**，高峰扩到 **20 个实例**；副本跨 Availability Zone 分布，避免单实例或单机架故障中断 Action 接收与回调补发。服务升级采用滚动发布，先用 **5% 流量**验证，再逐步替换其余副本。

## 计算资源

服务运行在 Linux 容器中，底层使用通用计算型虚拟机。单个生产实例配置 **4 vCPU 和 16 GB 内存**，JVM Heap 上限为 **10 GB**，剩余内存留给 Metaspace、线程栈、Direct Buffer、HTTP 客户端和容器运行时。

Scala + ZIO 的大部分任务是异步 I/O，正常吞吐不依赖大量线程。执行资源分成三部分：

- ZIO 默认执行器使用 **8 个计算线程**处理非阻塞业务逻辑；
- blocking executor 使用 **64 个线程**承载阻塞式 SDK 或 HTTP 调用；
- Catalog、组织信息服务和 Logic Apps Callback 使用独立连接池，单实例对每个下游的并发上限为 **100**。

扩容同时参考 CPU、内存和业务积压。CPU 不高但 Action P99、入口队列或回调最老年龄持续上升时，同样需要扩容或降低并发，因为瓶颈可能是线程、连接池或下游容量。

## 存储与数据边界

存储分成在线状态和历史归档两层。

**Cosmos DB** 保存在线查询和故障恢复需要的数据：

- Workflow Definition、版本和发布状态；
- Catalog 业务请求 ID 与 Logic Apps Instance ID 的映射；
- Purview 包装的实例摘要；
- Action 执行标识、业务执行状态和结果引用；
- Logic Apps 回调与 Catalog 回调的确认状态；
- 幂等记录和需要补发的任务。

**Azure Blob Storage** 保存已经执行过的 Workflow 历史。工作流到达终态后，归档任务把 Definition Version、运行输入、各 Action 状态与结果、错误和时间信息写入 Blob，供历史查询、审计和问题排查。在线接口不需要长期把完整历史保留在 Cosmos DB 中。

Expression 字符串属于 Workflow Definition，也会随对应 Definition Version 保留。历史归档记录当时实际使用的版本和执行结果，避免定义升级后无法还原旧实例。

## 分区设计

Cosmos DB 以租户作为数据隔离和主要访问边界。Definition 和实例记录使用 `tenantId` 作为逻辑分区入口，所有 API 都先从身份中取得租户，再访问对应分区，避免跨租户扫描。一个区域在线保存 **2000 万条** Definition、实例摘要和 Action 状态，单条状态记录为 **1～5 KB**。

高频 Action 执行和回调记录使用合成分区键：

```text
tenantId + hash(workflowInstanceId) % bucketCount
```

同一 Workflow Instance 的记录落在固定 Bucket，便于按实例读取；大型租户使用 **32 个 Bucket**，使不同实例分散到多个物理分区，避免一个租户成为热点。记录 ID 由 Workflow Instance ID、Action ID 和执行标识组成，用于条件写入和幂等去重。

这项设计的取舍是：按单个实例查询很直接，按租户统计则要跨 Bucket 聚合。因此线上报表和全局监控不直接扫描业务集合，而是从指标管道和异步聚合结果读取。

Blob 历史按区域和租户隔离，并按完成时间与 Workflow Instance ID 组织路径：

```text
region/tenantId/yyyy/mm/dd/workflowInstanceId.json
```

单个 Workflow 历史文件为 **20～200 KB**，包含大型 Action 输出时达到 **5 MB**。单区域每天完成 **5 万个 Workflow**，历史日增量为 **1～10 GB**，大型输出集中时达到 **20 GB**。单个实例的历史可以直接定位，按时间范围的审计任务也能批量读取。

Blob Lifecycle Policy 将 **90 天**以前的历史转到 Cool Tier，保留 **1 年**后删除。归档任务监控失败、积压和 Cosmos DB 终态实例与 Blob 历史之间的差异。

## 容量与扩缩容

容量规划分成入口、执行和回调三部分：

- **入口容量**：单区域每天 **5 万个 Workflow**，Workflow API 峰值 **20 RPS**；
- **执行容量**：每个 Workflow 有 **5～10 个 Action**，Action API 峰值 **100 RPS**，同时运行 **1000 个 Action**；
- **恢复容量**：正常待确认回调低于 **100 条**，恢复 Worker 每分钟处理 **1000 条**积压。

服务按 CPU、请求并发和队列长度自动扩缩容。CPU 连续 10 分钟超过 **65%**，或者入口队列超过 **500 条**时扩容；CPU 连续 30 分钟低于 **30%**且入口队列低于 **100 条**时缩容。副本增加不能突破下游容量，每个 Action 类型和下游服务都有独立并发上限；发生突发流量时先排队和限流，避免横向扩容把压力原样传给 Catalog 或组织信息服务。

发布前使用代表性 Workflow 做容量测试，覆盖较长顺序流程、多个网络 Expression 函数、Azure OpenAI 调用、下游延迟和 Logic Apps 重试。除了稳态吞吐，还要验证依赖恢复后，服务能否在不再次压垮下游的情况下消化积压。

## Azure OpenAI 部署与容量

每个 Geography 使用同区域的 Azure OpenAI Resource 和 `gpt-35-turbo` Deployment，Workflow Definition 不跨区域发送。Scala 服务通过 Microsoft Entra ID 托管身份取得访问令牌，Endpoint、Deployment、API Version 和 Prompt Version 由区域配置管理。

单区域每天处理 **2 万次 Workflow 概括**和 **5000 次 Workflow 生成**。概括平均使用 **2500 个输入 Token 和 400 个输出 Token**，生成平均使用 **4500 个输入 Token 和 1800 个输出 Token**。Azure OpenAI Deployment 的配额为 **100 万 TPM 和 300 RPM**，服务侧再按租户限制并发，防止单个租户占满区域配额。

模型调用使用独立 HTTP 连接池，单次 Deadline 为 **30 秒**。429、5xx 和临时网络错误最多退避重试 **2 次**；JSON 解析失败和 Definition Validator 失败不重试模型。Azure OpenAI 不可用时，概括与生成功能返回暂时不可用，用户仍然可以查看、编辑和发布普通 Workflow Definition，已经发布的 Workflow 也继续由 Logic Apps 执行。

模型月费用为 **$4,282.5**。预算按区域和 Deployment 统计，Token 或调用量超过月度预算的 **80%** 时告警，达到 **100%** 时限制新增生成请求，概括请求继续保留。

## 线上指标与监控

监控覆盖业务闭环、Workflow 实例、异步 Action 协议、下游依赖和服务资源。

### 业务与 Workflow

- Catalog 业务请求成功率为 **99.9%**，普通流程端到端 P95 为 **5 分钟**；
- Logic Apps 已完成但 Catalog 尚未完成的记录正常为 **0**，超过 **15 分钟**仍未收敛则告警；
- 单区域同时运行 **1000～3000 个 Workflow Instance**；
- 实例时长记录 P50、P95 和 P99，审批类 Workflow 与自动流程分开统计；
- HTTP Action 入口 P95 为 **200 ms**，P99 为 **1 秒**；
- `runAfter` 条件满足后超过 **5 分钟**仍未开始的节点进入异常统计。

指标按 Workflow 类型、Action 类型、Definition Version、区域和租户切分，避免全局平均值掩盖单一版本或区域故障。

### 异步 Action 与依赖

- Logic Apps 调用 Purview Action API 的重复请求率低于 **0.5%**；
- 回调待确认任务低于 **100 条**；
- 自动 Action 执行 P95 为 **2 秒**；
- Purview 回调 Logic Apps 超过 **10 分钟**仍未确认则告警；
- Expression 发布后的求值错误率低于 **0.1%**；
- `getUserName`、`getManager` 等网络函数的单次调用超时为 **3 秒**；
- 记录结果未知、状态查询和幂等命中次数。

积压数量要和最老记录年龄一起看。低流量下即使只有一条待确认记录，停留数小时也说明恢复链路失效。

### LLM 调用

- Workflow 概括和生成的请求量、成功率、P50、P95 和 P99；
- Azure OpenAI 的 429、5xx、网络超时和重试率；
- 输入 Token、输出 Token、TPM、RPM 和配额利用率；
- JSON 解析成功率、Definition Validator 通过率和用户重新生成率；
- Prompt Version、Model Deployment 和单次、每日及月度费用；
- 敏感 Parameter 过滤失败数保持为 **0**。

模型调用 P95 为 **8 秒**，P99 为 **20 秒**；429 比例低于 **0.5%**，JSON 解析成功率为 **98%**，Definition Validator 通过率为 **92%**。指标按概括与生成、区域、租户、Prompt Version 和 Deployment 切分。

### 服务资源

- 入口队列低于 **100 条**；
- CPU 保持在 **30%～50%**，GC P99 暂停低于 **200 ms**；
- 单实例运行 **5000～20000 个 ZIO Fiber**；
- HTTP、数据库连接池和 Cosmos DB RU 在高峰保留 **30%**余量；
- Blob 历史归档 P95 低于 **10 分钟**，在线终态与归档记录差异为 **0**；
- 记录部署版本、实例数、重启和异常退出。

结构化日志和 Trace 使用同一组关联标识：

```text
业务请求 ID
→ Definition Version
→ Logic Apps Instance ID
→ Action ID 与执行标识
→ Expression 求值
→ 下游请求与幂等键
→ Logic Apps Callback
→ Catalog 最终状态
```

告警优先覆盖端到端成功率下降、Action 失败或 P99 突增、节点停留过久、回调积压年龄增长、Logic Apps 与 Catalog 状态无法收敛，以及下游、连接池或 Cosmos DB 分区接近饱和。单次可恢复重试只记录指标；达到重试上限、积压持续增长或业务结果无法收敛时才升级值班告警。

## Action 延迟与回调积压事故

一次新版本发布后，Purview Action API 的 P99 开始上升，Logic Apps 中处于运行状态的 HTTP Action 增多。Logic Apps 因入口请求超时而重试，相同执行标识的重复请求增加；随后，业务已经完成但回调 Logic Apps 尚未确认的记录也开始积压。

CPU、JVM Heap 和 GC 没有明显异常，但服务吞吐下降。按 Definition Version 和 Action 类型拆分指标后，问题集中在使用 `getManager` 等网络 Expression 函数的 Action。Trace 显示主要时间消耗在 Expression 求值中的网络函数。

进一步检查线程和 ZIO 指标发现，这些函数使用阻塞式 HTTP 客户端，并直接运行在 ZIO 默认执行器上。下游延迟升高后，承载 Fiber 的底层线程被阻塞，其他 Action 的解析、状态更新和回调也无法及时获得执行机会。

```text
组织信息服务延迟上升
→ 阻塞调用占满 ZIO 默认执行器线程
→ Action API 与业务执行变慢
→ Logic Apps 超时重试
→ 重复请求继续增加负载
→ Logic Apps 回调无法及时执行
→ Action 和待确认回调持续积压
```

应急处理先暂停问题版本放量并限制相关 Action 并发。入口请求继续按执行标识去重；业务已经完成的任务只补发回调；下游结果未知的任务先查询状态。临时增加实例帮助消化存量，但不能代替根因修复。

最终将阻塞式调用移到专用 blocking executor，为组织信息服务配置独立连接池和并发上限，并收紧网络函数的单次超时、总预算和重试条件。后续逐步将高频网络函数迁移到非阻塞 HTTP 客户端。

修复在注入高延迟和超时的压测环境中验证，再逐步放量。观察重点包括网络函数与 Action 的 P95/P99、默认执行器和 blocking executor 的饱和度、连接池、重复入口请求、待确认回调的数量和最老年龄，以及普通 Action 是否受到影响。

事故后增加了执行器饱和和回调积压年龄告警。这次问题说明，Fiber 数量不等于真正的执行能力；阻塞式 I/O 没有隔离时仍会占满底层线程，而重试会把一个下游变慢放大成整个 Action 与回调链路的拥塞。
