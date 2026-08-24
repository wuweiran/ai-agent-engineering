---
layout: default
title: AKS 部署与生产运行
parent: My Outlook Agent 部署
grand_parent: 工作经历
nav_order: 4
permalink: /docs/career/my-outlook-agent/aks-production/
---

# AKS 部署与生产运行

这部分先说明 [My Outlook 项目概览]({{ site.baseurl }}/docs/career/my-outlook-agent/)中的 Component 在 AKS 里分别以什么形式运行，以及代码怎样变成实际运行的 Pod。后半部分再展开扩缩容、灰度、观测和事故处理。

## My Outlook 在 AKS 中是什么样

AKS 是 Azure 托管的 Kubernetes。My Outlook 并不是把所有 Component 放进一个进程，而是把 POS Service 和不同类型的 Worker 分成多个独立 Deployment。

```text
Outlook Client
  ↓
Kubernetes Service：稳定的集群内访问地址
  ↓
POS Service Deployment
└─ 多个 POS Service Pod：Agent Loop、Context Builder、任务编排
  ↓
Service Bus / Task Store：位于 AKS 外部
  ↓
Worker Deployments：按 POS 提供的 Step 参数执行
├─ Deep Scan Worker Pool：内置 Deep Scan Handler
├─ Synthesis Worker Pool：内置 Synthesis Handler
├─ LLM Worker Pool：调用模型 Endpoint
└─ Graph/API Worker Pool：内置业务 API Handler
  ↓
SDS / Annotation Store / Microsoft 365 APIs：位于 AKS 外部
```

各 Component 与 Kubernetes 对象的对应关系是：

| My Outlook Component | 在 AKS 中的形式 | 说明 |
| --- | --- | --- |
| **POS Service** | `Deployment` + `Pod` + `Service` | Deployment 管理 POS Service Pod；Service 为这些 Pod 提供稳定地址 |
| **Deep Scan Worker Pool** | 独立 `Deployment` + `Pod` | 按 POS 提供的参数调用 Deep Scan Handler |
| **Synthesis Worker Pool** | 独立 `Deployment` + `Pod` | 按 POS 提供的参数调用 Synthesis Handler |
| **LLM Worker Pool** | 独立 `Deployment` + `Pod` | 执行 POS 指定的模型调用参数 |
| **Graph/API Worker Pool** | 独立 `Deployment` + `Pod` | 执行 POS 指定的 Microsoft 365 或业务 API 调用 |

这些 Worker Pool 按资源类型和依赖隔离，但不负责业务编排。业务目标、Step 类型、工具名称和完整参数都由 POS Service 写入 Step；Worker 只调用内置 Handler 并返回结构化 Result。

**Region 容量快照：My Outlook 尚未 GA、用户量较小时，每个 Region 常态运行约 3 个 POS Service Pod 和 12 个 Worker Pod，其中 Deep Scan 4 个、Synthesis 2 个、LLM 2 个、Graph/API 4 个。**实际副本数会随在线请求和队列负载变化。

AKS 之外还有几类托管依赖：Azure Service Bus 传递 Worker 调度消息，Conversation Store 保存对话，Task Store 保存任务执行状态，Annotation Store 保存 Finding，SDS 保存生成产物，Microsoft 365 保存邮件和日历等权威业务数据。POS Service 和 Worker 通过 Endpoint 与托管身份访问这些系统；完整持久化关系见 [一个 Agent Run 的数据保存在哪里]({{ site.baseurl }}/docs/career/my-outlook-agent/runtime-task/#一个-agent-run-的数据保存在哪里)。

这里最重要的三个 Kubernetes 概念是：

- **Pod**：一个实际运行的应用实例，例如一个 POS Service 实例或一个 Deep Scan Worker 实例；
- **Deployment**：管理一组同类 Pod，声明使用哪个镜像、运行多少副本以及怎样滚动发布；
- **Service**：为一组需要接收网络请求的 Pod 提供稳定地址。POS Service 需要 Kubernetes Service，主动消费队列的 Worker 不需要。

## 从代码到 Pod 怎样部署

部署过程可以沿下面这条链路理解：

```text
Service / Worker 代码
→ 构建 Docker 镜像
→ 推送到 Azure Container Registry
→ Helm 将模板与环境 Values 渲染成 Kubernetes Manifest
→ Manifest 提交给 AKS
→ Deployment 创建并管理 Pod
→ Service 把在线请求转发到可用的 POS Service Pod
```

My Outlook 使用一套 Helm Chart 描述这些 Deployment、Service 和运行配置，再为测试 Ring 与生产环境提供不同 Values。发布新版本时更新镜像 Tag，Helm 生成新的 Manifest，Deployment 通过滚动发布逐步替换旧 Pod。Worker 启动后主动连接 Service Bus 消费任务，因此不需要对外暴露网络地址。

Kubernetes 负责运行和管理进程。Task 状态、Lease、幂等、重试和 Agent 完成条件由应用 Runtime 实现。

## AKS 部署单元

每类 Workload 使用独立镜像、Service Account、资源请求和上限。POS Service 跨可用区保留最小副本，配置 Readiness、Liveness 和 Startup Probe；Readiness 失败先停止接收新请求，Liveness 只用于无法自愈的进程状态，不能因为短暂下游故障反复重启 Pod。

Worker Pod 是无本地状态的。Task、Lease 和 Artifact 都保存在外部存储，因此节点替换后可以由其他 Pod 继续。Pod 终止时先停止领取新消息，在 `terminationGracePeriod` 内完成或交还当前 Lease；未完成的 Step 在 Lease 到期后重新调度。

发布配置通过 Helm 管理，镜像、Agent 配置、队列名称、并发上限和依赖 Endpoint 分开版本化。Secret 使用托管身份和外部 Secret Store 注入，不写入镜像或日志。

## Manifest、Helm Chart 和 Config 分别是什么

My Outlook 使用 Helm 管理部署配置。Helm Chart 中的 Template 定义 Deployment、Service 和 ConfigMap 等 Kubernetes 资源结构，不同环境的 Values 提供镜像版本、副本数、资源和 Endpoint 等参数。Helm 将 Template 与 Values 渲染成最终 Kubernetes Manifest，再提交给 AKS。

**Helm Template + 环境 Values → Kubernetes Manifest → AKS。**

非敏感运行参数进入 Values 或 ConfigMap，敏感信息通过 Managed Identity 和外部 Secret Store 提供。

## POS Service Deployment

用于 Workload Identity 的 Service Account 先绑定托管身份 Client ID：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-outlook-workload
  annotations:
    azure.workload.identity/client-id: <managed-identity-client-id>
```

Helm Template 渲染后的 POS Service `Deployment` 结构如下，配置值仍使用匿名化示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-outlook-pos
  # 省略：通用 labels
spec:
  replicas: <online-replicas>
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    # 省略：与 Pod labels 对应的 selector
  template:
    metadata:
      labels:
        app: my-outlook-pos
        workload: online
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: my-outlook-workload
      terminationGracePeriodSeconds: <grace-period>
      # 省略：可用区分布等通用 Pod 配置
      containers:
        - name: pos-service
          image: <registry>/my-outlook-pos:<version>
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: my-outlook-runtime
          # 省略：所有服务都有的 resources、日志和遥测配置
          startupProbe:
            httpGet: { path: /health/startup, port: http }
          readinessProbe:
            httpGet: { path: /health/ready, port: http }
          livenessProbe:
            httpGet: { path: /health/live, port: http }
          lifecycle:
            preStop:
              httpGet: { path: /admin/drain, port: http }
```

关键参数的作用是：

- `maxUnavailable` 和 `maxSurge`：控制滚动发布期间的可用实例数；
- Startup、Readiness 和 Liveness Probe：分别判断启动完成、能否接收流量和进程是否需要重启；
- `preStop + terminationGracePeriodSeconds`：先停止接收新请求，再让当前 Agent Loop 保存状态。

POS Service 使用 Kubernetes `Service` 暴露稳定的集群内地址：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-outlook-pos
spec:
  selector:
    app: my-outlook-pos
  ports:
    - name: http
      port: 80
      targetPort: http
  # 省略：内部 Service 的通用网络配置
```

## Worker Deployment

Worker 不暴露 HTTP Service。它启动后从 Service Bus 拉取 Step，Deployment 重点配置并发、资源、优雅退出和身份：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-outlook-deep-scan-worker
  # 省略：通用 labels
spec:
  replicas: <worker-replicas>
  selector:
    # 省略：与 Pod labels 对应的 selector
  template:
    metadata:
      labels:
        app: my-outlook-deep-scan-worker
        workload: background
    spec:
      serviceAccountName: my-outlook-workload
      terminationGracePeriodSeconds: <worker-grace-period>
      containers:
        - name: worker
          image: <registry>/my-outlook-worker:<version>
          args: ["--worker-type=deep-scan"]
          env:
            - name: QUEUE_NAME
              value: <deep-scan-queue>
            - name: MAX_CONCURRENT_STEPS
              value: <worker-concurrency>
            - name: LEASE_SECONDS
              value: <lease-seconds>
            - name: MAX_ATTEMPTS
              value: <max-attempts>
          # 省略：所有 Worker 都有的 resources、日志和遥测配置
          readinessProbe:
            exec:
              command: ["/app/worker-health", "--ready"]
          lifecycle:
            preStop:
              exec:
                command: ["/app/drain-worker", "--stop-receiving"]
```

**Worker 的 Readiness 表示是否还能安全领取新任务，不代表下游始终健康。**收到终止信号后先停止拉取消息，正在执行的 Step 尽量保存结果；未完成部分通过 Lease 到期重新调度。

## HPA 与 KEDA 配置

**HPA 根据 CPU、请求并发和延迟等在线负载调整 POS Service Pod 数量；KEDA 根据 Service Bus 队列积压调整 Worker Pod 数量。**

POS Service 使用 HPA，扩缩容信号包括 CPU、请求并发和延迟。下面的配置展示其中的 CPU Metric：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-outlook-pos
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-outlook-pos
  minReplicas: <min-online-replicas>
  maxReplicas: <max-online-replicas>
  metrics:
    # 关键区别：POS 按 CPU、请求并发和延迟等在线负载扩缩容
    - <online-load-metrics>
  # 省略：通用 scale-up / scale-down 稳定策略
```

Worker 使用 KEDA `ScaledObject` 根据 Service Bus Queue 扩缩容：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: deep-scan-worker
spec:
  scaleTargetRef:
    name: my-outlook-deep-scan-worker
  minReplicaCount: <min-worker-replicas>
  maxReplicaCount: <max-worker-replicas>
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: <deep-scan-queue>
        messageCount: <target-messages-per-pod>
        namespace: <service-bus-namespace>
      authenticationRef:
        name: service-bus-workload-identity
  # 省略：轮询、冷却和缩容稳定窗口
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: service-bus-workload-identity
spec:
  podIdentity:
    provider: azure-workload
```

`messageCount: "20"` 表示期望每个副本大约承接 20 条待处理消息，不是队列只能有 20 条。`cooldownPeriod` 和 Scale Down 稳定窗口防止队列短暂清空时立即缩容。**KEDA 负责 Pod 数量，应用内并发令牌负责真正的下游背压；最大副本仍受 Graph 与模型配额约束。**

## ConfigMap 与敏感配置

非敏感 Runtime 参数放在 `ConfigMap`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-outlook-runtime
data:
  TASK_STORE_ENDPOINT: <task-store-endpoint>
  ARTIFACT_STORE_ENDPOINT: <sds-endpoint>
  MODEL_GATEWAY_ENDPOINT: <model-gateway-endpoint>
  # 省略：其他非敏感 Runtime 参数
```

Service Bus、Graph 和存储凭据不直接写进 Manifest。Pod 使用 AKS Workload Identity，以 Service Account 绑定的托管身份换取短期 Token；如果必须读取 Secret，则通过 Key Vault CSI Driver 挂载，并限制 Namespace、Service Account 和访问 Scope。

另外配置 `PodDisruptionBudget`，避免节点维护时同时驱逐过多在线实例：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-outlook-pos
spec:
  minAvailable: <minimum-available-pos-pods>
  selector:
    matchLabels:
      app: my-outlook-pos
```

**Manifest 解决部署、资源、身份、健康检查和扩缩容；Task 一致性、Lease、幂等、重试预算和 Agent 完成条件仍由应用 Runtime 实现。**这些能力不会因为使用 AKS 或 Helm 自动获得。

## 扩缩容与背压

POS Service 使用 HPA 按 CPU、请求并发和延迟扩容。Worker 使用 KEDA 进行事件驱动扩缩容，主要看：

```text
queue_depth
oldest_message_age
active_workers
step_processing_time
upstream_quota
```

只看 Queue Depth 会误判。例如一个长时间模型调用和一百个短 Graph 查询对容量的含义不同，因此队列按 Workload Type 分开，并同时观察最老消息等待时间。扩容还受模型 Endpoint 与 Graph 配额限制：Worker Pool 有最大副本数，每个 Pod 有本地并发上限，模型网关再实施全局和租户级并发令牌。

过载时按优先级处理：先拒绝或延后低优先级后台刷新，保留用户主动发起的交互任务；Deep Scan 可以降低扫描深度或延迟执行，不能让推荐任务挤占用户正在等待的请求。重试任务进入带 `not_before` 的等待队列，避免立即重新消费形成热循环。

## 灰度和回滚

新版本先经过集成测试和固定 Agent Scenario，再部署到内部 Ring。POS Service 与 Worker 可以独立发布，但兼容性由 Task Schema 和 Worker Capability Version 约束：

- 新增可选字段保持旧 Worker 可读；
- 破坏性 Schema 变化必须新建版本并提供迁移；
- Task 记录创建它的 Agent 和 Worker 兼容版本；
- 回滚停止新任务进入问题版本，不删除已经运行的 Task；
- 旧 Task 继续由兼容 Worker 完成，或经过显式迁移后再接管。

发布观察不只看 Pod 健康，还比较 Task 完成率、各 Step 失败率、Queue Age、模型与 Graph 429、每个成功任务 Token 和用户主动重试。Service HTTP 200 正常但 Task 长时间不完成，仍然属于发布失败。

## 生产观测

观测使用同一组关联 ID：

```text
conversation_id
→ task_id
→ run_id
→ step_id
→ attempt_id
→ model_call_id / dependency_request_id
```

Metrics 分为四层：

1. **用户结果**：Task 完成率、放弃率、取消率、主动重试和等待时间；
2. **调度执行**：Queue Depth、Oldest Message Age、Lease 过期、Attempt 次数、Step 停留时间；
3. **依赖容量**：模型与 Graph 429、Retry-After、并发令牌、Token 和存储延迟；
4. **AKS 资源**：副本数、Pod 重启、CPU、内存、OOM、节点压力和 Probe 失败。

Log 和 Trace 记录状态、版本、资源 ID、耗时和错误码，不保存完整邮件、附件、Prompt 或用户 Token。需要定位内容质量时，通过受控权限读取 Artifact 与源数据，不能把敏感内容默认复制进普通生产日志。

## 这部分能够回答的面试题

- [生产级 Agent 架构怎样设计？]({{ site.baseurl }}/docs/interview/ai-agent/system-production/#production-agent-architecture)
- [模型并发限制应该放在哪里？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-concurrency-limits)
- [模型 API 限流后怎样重试？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-rate-limit-retry)
- [消息积压时能否直接增加消费者？]({{ site.baseurl }}/docs/interview/backend/message-queue/#scale-consumers-for-backlog)
- [线程池、连接池和队列怎样共同限制吞吐？]({{ site.baseurl }}/docs/interview/backend/performance-production/#thread-connection-pool-queue)
- [指标、日志和 Trace 分别解决什么问题？]({{ site.baseurl }}/docs/interview/backend/performance-production/#metrics-logs-traces)
- [大模型应用应该记录哪些可观测数据？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-observability)
