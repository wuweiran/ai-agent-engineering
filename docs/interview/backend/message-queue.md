---
layout: default
title: 消息队列与 Kafka 常见面试题
parent: 后端面试题
grand_parent: 面试题库
nav_order: 4
permalink: /docs/interview/backend/message-queue/
---

# 消息队列与 Kafka 常见面试题

消息队列面试的核心不是背产品配置，而是说清业务写入、消息发送、消费和确认之间的故障窗口。页面先讨论通用消息机制和工程场景，再补充 Kafka 的分区、副本、消费组与事务语义。

## 通用消息机制

## 为什么使用消息队列？
{: #message-queue-purpose }

消息队列主要用于异步、解耦和削峰。异步让不影响当前响应的工作稍后完成；解耦让生产者不必同步调用每个下游；削峰用队列吸收短时流量，让消费者按可承受速度处理。

代价是系统要处理消息丢失、重复、乱序、积压和最终一致性。如果调用方必须立即取得结果，或者异步后的补偿成本不可接受，RPC 可能更合适。

## 消息队列和异步任务是什么关系？
{: #message-queue-vs-async-task }

异步任务表示一项工作需要跨越当前请求继续存在；消息队列是把待处理工作交给消费者的一种传输和调度机制。

任务目标、状态和结果通常还要持久化到业务数据库或任务系统。队列中的一条消息只能提醒 Worker 处理某个对象，不能代替完整任务状态和权威业务状态。

相关内容：[异步任务]({{ site.baseurl }}/docs/backend/async-tasks/)。

## At Most Once、At Least Once 和 Exactly Once 有什么区别？
{: #message-delivery-semantics }

At Most Once 最多处理一次，可能丢失但通常不会重做。At Least Once 至少有机会处理一次，通过确认和重试减少永久丢失，但可能重复。Exactly Once 希望最终效果只出现一次，保证范围取决于参与的系统边界。

生产系统常按 At Least Once 设计传输，再通过业务幂等得到“重复消息只产生一份业务结果”。

## 消息为什么会丢失？
{: #message-loss }

消息可能在生产者发送前、发送途中、Broker 持久化前，或者消费者处理前被错误确认。数据库提交后应用尚未发送消息就崩溃，也是常见故障窗口。

降低丢失需要生产端确认与重试、Broker 持久化和副本、消费成功后再确认，以及对账与补偿。没有单个配置能够覆盖整条业务链路。

## 消息为什么会重复？
{: #message-duplication }

生产者发送成功但没有收到确认时会重试；消费者完成业务操作后，可能在提交确认前崩溃，Broker 随后再次投递。Rebalance、超时和人工重放也会带来重复。

重复通常是可靠重试的结果。系统不应假设消息只出现一次，而要让同一业务意图能够被识别。

## 消费者怎样实现业务幂等？
{: #consumer-idempotency }

优先使用订单号、支付请求 ID、事件 ID 等稳定业务标识。可以通过唯一约束、条件状态转换、消费记录与业务写入同一事务等方式阻止重复结果。

只在 Redis 中短暂记录“已经消费”可能因过期、淘汰或写入时序重新放出重复。幂等最终要落到真正产生业务结果的位置。

## 怎样保证消息顺序？
{: #message-ordering }

先明确需要全局顺序，还是只要求同一订单、账户等业务键内有序。通常把同一业务键路由到同一有序队列或 Partition，并由单一执行序列处理。

失败重试、历史重放和多生产者仍可能带来迟到消息。事件应携带业务版本，消费者只接受合法状态转换，不能只相信到达顺序。

## 延迟消息和定时任务有什么区别？
{: #delayed-message-scheduled-task }

延迟消息由消息系统在指定时间后交付，适合规模较大、以事件触发的延后处理。定时任务由调度器按时间规则扫描或创建工作，更适合周期任务和可查询的业务期限。

无论采用哪种方式，消息到达时都要重新读取当前业务状态。三十分钟前安排的取消任务，不能直接覆盖刚刚完成的支付。

## 本地消息表解决什么问题？
{: #transactional-outbox }

本地消息表方案的完整名称是 Transactional Outbox，也常简称 Outbox。它把业务数据和“需要发送的事件”写入同一个本地数据库事务。独立发布器随后发送待处理事件，因此应用在业务提交后崩溃，也不会永久忘记发送。

发布器可能已经发送成功，却在标记事件前崩溃，所以本地消息表通常把“消息不丢”转换成“消息可能重复”，消费者仍要幂等。

## 为什么 Broker 的 Exactly Once 不能保证业务只发生一次？
{: #exactly-once-boundary }

Broker 只能在自己的记录、事务和消费位置边界内提供语义。订单数据库、短信平台和支付服务不会自动加入同一事务。

业务只发生一次通常来自稳定请求 ID、状态条件、唯一约束和结果查询。外部系统不支持幂等和查询时，系统无法同时承诺绝不遗漏且绝不重复。

## 什么是死信队列？
{: #dead-letter-queue }

持续重试仍无法处理的消息可以进入死信队列，避免一条坏消息永久阻塞正常消费。死信记录应保留原消息、业务标识、失败原因、消费者版本和尝试次数。

死信不是垃圾桶。团队需要告警、修复、审批和安全重放流程，而且重放仍必须幂等。

## 消息积压怎样处理？
{: #message-backlog }

先确认生产速度、消费速度和积压最老年龄，再定位瓶颈：流量突增、慢 SQL、下游变慢、锁竞争、线程池不足，或 Partition 数限制并行度。

止损可以限制上游、暂停非关键生产，并在业务允许并行时增加消费者。只增加实例未必有效；消费者数量超过可并行队列数，或者下游已经饱和，扩容只会增加竞争。

## 工程场景题

## 数据库已经提交，消息发送失败怎么办？
{: #database-committed-message-failed }

不能简单交换提交与发送顺序。先提交数据库会留下消息未发送窗口，先发送消息则可能让消费者看到最终回滚的业务。

常见方案是本地消息表：业务事实与待发送事件共同提交，再由发布器可靠发送。若业务与消息系统支持经过验证的事务消息，也要明确其回查和故障边界。

## 消费者完成数据库写入后，在确认消息前崩溃怎么办？
{: #consumer-crash-before-ack }

Broker 没有收到确认，只能重新投递。消费者再次收到消息时，应通过业务状态、事件 ID 或唯一约束识别前一次结果，不能重复创建业务对象。

先确认消息再写数据库会把重复风险换成丢失风险：确认后进程崩溃，消息不会再来，业务却没有完成。

## 重复的订单取消消息为什么不能重复释放库存？
{: #duplicate-cancel-release-stock }

消费者应先以状态条件把订单从待支付改为已取消，只有状态转换实际成功时才释放库存。重复消息更新零行，不再增加库存。

订单取消和本地库存释放位于同一数据库时，应在一个事务中完成；跨服务时还需要稳定请求 ID 与结果查询。

## 同一订单的消息乱序到达怎么办？
{: #out-of-order-order-events }

订单事件应携带订单 ID、事件版本和前置状态。消费者只接受符合当前状态机的转换，迟到的创建事件不能把已支付订单退回待支付。

使用订单 ID 作为分区键可以减少并发乱序，但历史重放、跨 Topic 和不同生产者仍要求消费者校验版本与权威状态。

## 下游故障时，消费者应该继续拉取消息吗？
{: #consumer-downstream-failure }

若消费者继续高速拉取并同步调用故障下游，会占满线程、连接和本地内存，还可能形成重试风暴。应降低消费并发、暂停相关 Partition 或把任务转入受控重试。

恢复时逐步放量，并观察下游容量和消息最老年龄。停止拉取只是背压手段，不能代替故障消息的状态记录。

## 一条消息持续失败，为什么不能无限重试？
{: #poison-message-retry }

参数错误、数据缺失和永久业务冲突不会随时间自动恢复。无限重试会占用消费能力，阻塞同一有序队列中的后续消息，并制造大量日志。

应区分可恢复错误和永久错误，有限退避后进入死信或人工处置。修复后使用原业务标识安全重放。

## 消息积压时能否直接增加消费者？
{: #scale-consumers-for-backlog }

只有队列具有更多可并行分片，而且瓶颈位于消费者自身时，增加消费者才会提高吞吐。Partition 不足、单业务键必须顺序处理，或者数据库已经饱和时，扩容不会解决问题。

扩容前先测量每个阶段的处理时间和等待资源，必要时优化批处理、SQL 与外部调用。

## 怎样判断消息链路已经恢复？
{: #message-recovery-complete }

不能只看队列长度，要同时确认三层信号：

- **传输恢复**：新消息延迟回到正常范围，最老积压持续下降；
- **异常收敛**：错误重试和死信得到处理，没有继续产生同类失败；
- **业务恢复**：订单、库存等最终结果重新满足业务承诺。

清理历史积压时还要限制追赶速度，使其低于系统剩余容量，避免刚恢复的下游再次被压垮。

## Kafka 专项

Kafka 是一个分布式追加日志系统。Producer 把记录写入 Topic 的某个 Partition，Broker 把 Partition 以日志文件形式持久化并复制到其他节点；Consumer 按 Offset 顺序读取，Consumer Group 把不同 Partition 分配给组内消费者。KRaft Controller 负责集群元数据和 Leader 管理。

以下问题针对当前 Kafka 架构。Kafka 4.x 已采用 KRaft，不再把 ZooKeeper 作为新集群的核心依赖；旧版本问题需要结合实际部署回答。

## Topic、Partition、Broker 和 Replica 分别是什么？
{: #kafka-core-concepts }

Topic 是消息的逻辑分类；Partition 是 Topic 的有序追加日志和并行单位；Broker 是保存分区并服务读写的 Kafka 节点；Replica 是同一 Partition 在不同 Broker 上的副本。

每个 Partition 有一个 Leader 处理读写，Follower 复制数据并在故障时参与接管。

## Kafka 为什么使用 Partition？
{: #kafka-partition-purpose }

Partition 让一个 Topic 的数据分布到多个 Broker，并允许生产与消费并行扩展。Offset 只在 Partition 内有意义，Kafka 也只保证单 Partition 内的记录顺序。

Partition 增加了并行度，也增加副本、文件、元数据和 Rebalance 成本，不是越多越好。

## Kafka 怎样选择 Partition Key？
{: #kafka-partition-key }

需要保持同一业务对象顺序时，通常使用订单 ID、账户 ID 等稳定业务键，使相关消息进入同一 Partition。

Key 过于集中会产生热点 Partition；没有 Key 时记录可以较均匀分布，但失去按业务键的顺序保证。扩容 Partition 后，默认哈希映射也可能变化。

## Consumer Group 怎样分配 Partition？
{: #kafka-consumer-group }

同一 Consumer Group 内，一个 Partition 在同一时刻只交给一个 Consumer 处理；一个 Consumer 可以负责多个 Partition。不同消费组可以独立读取同一 Topic。

消费者数量超过 Partition 数时，多出的消费者没有分区可处理。消费并行度因此受 Partition 数上限约束。

## Offset 是什么？什么时候提交？
{: #kafka-offset-commit }

Offset 表示记录在 Partition 中的位置。Consumer 当前读取位置与已经提交的位置不同；提交位置用于消费者重启或重新分配后恢复。

业务处理成功后再提交 Offset，通常得到 At Least Once：提交失败会重复消费。处理前提交可能得到 At Most Once：进程随后崩溃会丢失业务处理。

## 自动提交 Offset 有什么风险？
{: #kafka-auto-commit }

自动提交按时间周期保存消费者位置，它不一定与单条业务处理完成同步。Offset 已提交但业务尚未成功时，进程崩溃可能跳过消息。

要求可靠处理时，通常在业务成功后显式提交，或使用框架提供的手动确认与事务能力，同时接受提交失败带来的重复消费。

## Rebalance 为什么发生？有什么影响？
{: #kafka-rebalance }

消费者加入、离开、订阅变化、Partition 数变化，或者消费者长时间没有 Poll，都可能触发消费组重新分配。

Rebalance 会暂停或移动部分消费任务，未及时提交的记录可能被其他消费者再次处理。稳定成员、合作式分配和控制单批处理时间可以减少不必要的抖动。

## `session.timeout` 和 `max.poll.interval` 分别控制什么？
{: #kafka-consumer-timeouts }

`session.timeout` 与心跳机制判断消费者是否仍然存活；`max.poll.interval` 限制两次 Poll 之间允许的最长处理时间，防止消费者活着却长时间不继续拉取。

业务处理过慢可能超过 Poll 间隔并触发 Rebalance。解决方式应包括控制批量大小、拆分耗时工作和调整消费结构，而不只是不断增大超时。

## Leader、Follower 和 ISR 分别是什么？
{: #kafka-leader-follower-isr }

Leader 负责 Partition 的客户端读写，Follower 从 Leader 复制日志。ISR 是当前与 Leader 保持足够同步、具备接管资格的副本集合。

副本总数不等于 ISR 数。网络和磁盘变慢会让副本退出 ISR，降低集群继续容忍故障的余量。

## `acks=all` 和 `min.insync.replicas` 怎样共同工作？
{: #kafka-acks-min-isr }

`acks=all` 要求 Leader 等待 ISR 中规定范围的副本确认；`min.insync.replicas` 规定允许成功写入所需的最少同步副本数。

当 ISR 数低于最小要求时，Broker 会拒绝写入，以可用性换取数据可靠性。副本数、最小同步副本和故障容忍能力必须一起设计。

## Kafka 幂等生产者解决什么问题？
{: #kafka-idempotent-producer }

幂等生产者为发送批次维护 Producer ID、Epoch 和序列号，Broker 可以识别因网络重试产生的重复批次，并保持单 Partition 内的写入顺序。

它不能识别业务层主动创建的两条相同订单事件，也不能让外部数据库写入幂等。业务重复仍需稳定事件 ID 和业务约束。

## Kafka 事务和 Exactly Once 的边界是什么？
{: #kafka-transactions-exactly-once }

Kafka 事务可以把多个 Partition 的写入，以及“消费输入后产生输出”的 Offset 提交放进同一个事务。消费者使用 `read_committed` 时只读取已提交事务记录。

这主要覆盖 Kafka 到 Kafka 的处理链路。写外部数据库、调用支付服务等副作用不会自动加入 Kafka 事务，仍需本地消息表、幂等或其他协调机制。

## Kafka 为什么吞吐量高？
{: #kafka-high-throughput }

Kafka 使用追加日志、顺序磁盘访问、操作系统 Page Cache、批量发送与压缩，并通过多个 Partition 并行扩展。向消费者传输数据时还可以减少不必要的数据复制。

高吞吐依赖批量和等待，会与单条消息的最低延迟形成取舍。吞吐优势也不能替代消费者业务逻辑优化。

## Kafka 消费积压怎样排查？
{: #kafka-consumer-lag }

先按 Consumer Group 和 Partition 查看 Lag，判断是全局变慢还是少数热点 Partition。再检查消费耗时、错误重试、Rebalance、下游依赖，以及 Broker 磁盘与网络。

如果只有一个 Partition 积压，增加消费者通常无效；应检查 Partition Key 是否倾斜，或单业务键是否限制并行。

## 增加 Partition 有什么影响？
{: #kafka-add-partitions }

增加 Partition 可以提高未来消息的并行度，却不会自动重新分布已有记录。默认 Key 到 Partition 的映射可能变化，同一 Key 的新旧消息可能不再位于同一 Partition，影响跨扩容时点的顺序假设。

Partition 数只能增加，不能简单减少。扩容前要评估消费者并行度、顺序要求和集群元数据成本。

## Kafka 和 RabbitMQ 怎样选择？
{: #kafka-vs-rabbitmq }

Kafka 适合高吞吐事件流、日志保留、重放和多个消费组独立读取。RabbitMQ 更偏传统消息代理，擅长灵活路由、单条消息投递和任务队列语义。

选择应依据消息保留与重放、顺序、路由、吞吐、延迟和团队运维能力，不应只比较单项性能数字。
