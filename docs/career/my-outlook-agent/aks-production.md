---
layout: default
title: AKS 部署与生产运行
parent: My Outlook Agent 部署
grand_parent: 工作经历
nav_order: 3
permalink: /docs/career/my-outlook-agent/aks-production/
---

# AKS 部署与生产运行

这部分展开 [My Outlook 项目概览]({{ site.baseurl }}/docs/career/my-outlook-agent/)中的 AKS 部署，重点说明 Workload 隔离、依赖配额、扩缩容、灰度和事故处理。

## 为什么 Service 和 Worker 分开部署

POS Service 面向 Outlook Client，重点是稳定接入和快速返回；Deep Scan、Synthesis、LLM 与 Graph/API Worker 的资源形态和依赖配额不同。如果放在同一个 Deployment 中，扫描任务的 CPU 和内存峰值、模型调用的长等待以及 Graph 限流会共同影响在线 Agent 请求，也无法针对真实瓶颈独立扩容。

因此它们分别部署为 AKS Deployment：

| Workload | 主要资源和限制 | 扩缩容信号 |
| --- | --- | --- |
| POS Service | 请求并发、CPU、连接数、Agent Loop 延迟 | RPS、并发请求、P95、CPU |
| Deep Scan Worker | CPU、内存、Graph 读取吞吐 | Queue Depth、Oldest Message Age、CPU/Memory |
| Synthesis Worker | Artifact 数量、模型和 CPU | Queue Depth、处理时长、LLM 配额 |
| LLM Worker | 模型 Endpoint 并发与 Token | In-flight Calls、429、Token/minute |
| Graph/API Worker | Graph 配额和网络连接 | In-flight Requests、429、Retry-After |

**扩容目标不是让队列永远为零，而是在下游配额允许范围内控制排队时间。**模型或 Graph 已经限流时，盲目增加 Worker 只会制造更多并发请求。

## AKS 部署单元

每类 Workload 使用独立镜像、Service Account、资源请求和上限。POS Service 跨可用区保留最小副本，配置 Readiness、Liveness 和 Startup Probe；Readiness 失败先停止接收新请求，Liveness 只用于无法自愈的进程状态，不能因为短暂下游故障反复重启 Pod。

Worker Pod 是无本地状态的。Task、Lease 和 Artifact 都保存在外部存储，因此节点替换后可以由其他 Pod 继续。Pod 终止时先停止领取新消息，在 `terminationGracePeriod` 内完成或交还当前 Lease；未完成的 Step 在 Lease 到期后重新调度。

发布配置通过 Helm 管理，镜像、Agent 配置、队列名称、并发上限和依赖 Endpoint 分开版本化。Secret 使用托管身份和外部 Secret Store 注入，不写入镜像或日志。

## Manifest、Helm Chart 和 Config 分别是什么

这里可以把术语分清楚：

- **Kubernetes Manifest**：最终提交给 AKS API Server 的 YAML 资源定义，例如 `Deployment`、`Service`、`ConfigMap`、`HorizontalPodAutoscaler` 和 KEDA `ScaledObject`；
- **Helm Chart**：参数化生成这些 Manifest 的模板包，包含 `Chart.yaml`、`values.yaml` 和 `templates/`；
- **Config**：泛指运行参数。非敏感配置放 `values.yaml` 或 `ConfigMap`，敏感信息通过 Managed Identity 和 Key Vault 等外部 Secret Store 提供。

**My Outlook 使用一套 Helm Chart，为不同环境提供独立 Values，再生成最终 Manifest。**它不手工维护每个环境的一整套重复 YAML：

```text
charts/my-outlook/
├─ Chart.yaml
├─ values.yaml
├─ values-ring0.yaml
├─ values-prod.yaml
└─ templates/
   ├─ pos-service-deployment.yaml
   ├─ pos-service-service.yaml
   ├─ deep-scan-worker-deployment.yaml
   ├─ synthesis-worker-deployment.yaml
   ├─ service-account.yaml
   ├─ config-map.yaml
   ├─ hpa.yaml
   ├─ keda-scaled-object.yaml
   └─ pod-disruption-budget.yaml
```

下面是简化后的 `values-prod.yaml`。它保存环境差异，不保存访问令牌：

```yaml
global:
  environment: prod
  region: westus2
  imageRegistry: outlookprod.azurecr.io
  serviceAccountName: my-outlook-workload

posService:
  image:
    repository: my-outlook-pos
    tag: "2026.02.18.3"
  replicas: 6
  resources:
    requests: { cpu: "1", memory: 2Gi }
    limits: { cpu: "2", memory: 4Gi }
  autoscaling:
    minReplicas: 6
    maxReplicas: 30
    targetCPUUtilization: 60
  runtime:
    requestTimeoutSeconds: 30
    maxActiveAgentLoops: 80
    reservedOutputTokens: 3000
    terminationGracePeriodSeconds: 60

workers:
  deepScan:
    imageTag: "2026.02.18.3"
    queueName: deep-scan-prod
    minReplicas: 2
    maxReplicas: 40
    targetQueueLength: 20
    maxConcurrentStepsPerPod: 4
    leaseSeconds: 120
    maxAttempts: 4
    resources:
      requests: { cpu: "2", memory: 6Gi }
      limits: { cpu: "4", memory: 10Gi }
  synthesis:
    queueName: synthesis-prod
    minReplicas: 2
    maxReplicas: 20
    targetQueueLength: 10
    maxConcurrentStepsPerPod: 2

agent:
  taskStoreEndpoint: https://taskstore.prod.internal
  artifactStoreEndpoint: https://sds.prod.internal
  modelGatewayEndpoint: https://modelgateway.prod.internal
  graphConcurrencyPerPod: 8
  modelConcurrencyPerPod: 2
  taskDeadlineMinutes: 30
  retry:
    baseDelaySeconds: 2
    maxDelaySeconds: 60
    jitterPercent: 25
```

生产中的实际数字需要通过容量测试和下游配额确定。**副本和资源属于部署配置，队列、Lease、并发和重试属于 Worker Runtime 配置，Endpoint 与版本属于环境配置。**

## POS Service Deployment 大致长什么样

用于 Workload Identity 的 Service Account 先绑定托管身份 Client ID：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-outlook-workload
  annotations:
    azure.workload.identity/client-id: 11111111-2222-3333-4444-555555555555
```

Helm Template 渲染后的 POS Service `Deployment` 可以简化为：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-outlook-pos
  labels:
    app: my-outlook-pos
    ring: ring0
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: my-outlook-pos
  template:
    metadata:
      labels:
        app: my-outlook-pos
        workload: online
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: my-outlook-workload
      terminationGracePeriodSeconds: 60
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-outlook-pos
      containers:
        - name: pos-service
          image: outlookprod.azurecr.io/my-outlook-pos:2026.02.18.3
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: my-outlook-runtime
          resources:
            requests:
              cpu: "1"
              memory: 2Gi
            limits:
              cpu: "2"
              memory: 4Gi
          startupProbe:
            httpGet: { path: /health/startup, port: http }
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            httpGet: { path: /health/ready, port: http }
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /health/live, port: http }
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop:
              httpGet: { path: /admin/drain, port: http }
```

关键参数的作用是：

- `maxUnavailable: 0`：滚动发布期间不主动减少可用 POS Pod；
- `maxSurge: 1`：每次最多额外创建一个新 Pod，控制发布资源峰值；
- `topologySpreadConstraints`：把在线实例分散到可用区；
- `requests`：调度和 HPA 计算的资源基线，`limits` 防止单 Pod 无界占用节点；
- `startupProbe`：允许应用完成较慢初始化，避免启动阶段被 Liveness 反复重启；
- `readinessProbe`：决定是否接收新请求，不检查 Graph 或模型是否短暂可用；
- `preStop + terminationGracePeriodSeconds`：先 Drain 新请求，再让当前 Agent Loop 保存状态。

POS Service 还需要一个 `Service` 暴露稳定集群地址，并由内部 Ingress 或服务网格接入：

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
```

## Worker Deployment 大致长什么样

Worker 不暴露 HTTP Service。它启动后从 Service Bus 拉取 Step，Deployment 重点配置并发、资源、优雅退出和身份：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-outlook-deep-scan-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-outlook-deep-scan-worker
  template:
    metadata:
      labels:
        app: my-outlook-deep-scan-worker
        workload: background
    spec:
      serviceAccountName: my-outlook-workload
      terminationGracePeriodSeconds: 150
      containers:
        - name: worker
          image: outlookprod.azurecr.io/my-outlook-worker:2026.02.18.3
          args: ["--worker-type=deep-scan"]
          env:
            - name: QUEUE_NAME
              value: deep-scan-prod
            - name: MAX_CONCURRENT_STEPS
              value: "4"
            - name: LEASE_SECONDS
              value: "120"
            - name: MAX_ATTEMPTS
              value: "4"
          resources:
            requests:
              cpu: "2"
              memory: 6Gi
            limits:
              cpu: "4"
              memory: 10Gi
          readinessProbe:
            exec:
              command: ["/app/worker-health", "--ready"]
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/app/drain-worker", "--stop-receiving"]
```

**Worker 的 Readiness 表示是否还能安全领取新任务，不代表下游始终健康。**收到终止信号后先停止拉取消息，正在执行的 Step 尽量保存结果；未完成部分通过 Lease 到期重新调度。

## HPA 与 KEDA 配置

POS Service 可使用 HPA。最基本的 CPU 配置如下，生产中也可以接入请求并发或延迟等自定义 Metric：

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
  minReplicas: 6
  maxReplicas: 30
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Percent
          value: 20
          periodSeconds: 120
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
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
  minReplicaCount: 2
  maxReplicaCount: 40
  pollingInterval: 15
  cooldownPeriod: 300
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 600
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: deep-scan-prod
        messageCount: "20"
        namespace: my-outlook-prod
      authenticationRef:
        name: service-bus-workload-identity
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
  TASK_STORE_ENDPOINT: https://taskstore.prod.internal
  ARTIFACT_STORE_ENDPOINT: https://sds.prod.internal
  MODEL_GATEWAY_ENDPOINT: https://modelgateway.prod.internal
  TASK_DEADLINE_MINUTES: "30"
  GRAPH_CONCURRENCY_PER_POD: "8"
  MODEL_CONCURRENCY_PER_POD: "2"
```

Service Bus、Graph 和存储凭据不直接写进 Manifest。Pod 使用 AKS Workload Identity，以 Service Account 绑定的托管身份换取短期 Token；如果必须读取 Secret，则通过 Key Vault CSI Driver 挂载，并限制 Namespace、Service Account 和访问 Scope。

另外配置 `PodDisruptionBudget`，避免节点维护时同时驱逐过多在线实例：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-outlook-pos
spec:
  minAvailable: 5
  selector:
    matchLabels:
      app: my-outlook-pos
```

**Manifest 解决部署、资源、身份、健康检查和扩缩容；Task 一致性、Lease、幂等、重试预算和 Agent 完成条件仍由应用 Runtime 实现。**这些能力不会因为使用 AKS 或 Helm 自动获得。

## 扩缩容与背压

POS Service 使用 HPA 按 CPU、请求并发和延迟扩容。Worker 使用 KEDA 类的事件驱动扩缩容，主要看：

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

## 一次 Worker 扩容事故

一次 Deep Scan 范围调整后，待扫描 Task 增加，Queue Depth 与 Oldest Message Age 快速上升。值班最初提高 Deep Scan Worker 最大副本数，队列短暂下降，但 Graph `429` 很快升高，Step 重试使队列再次增长，同时用户主动发起的 Catch-up 请求也开始变慢。

```text
扫描范围扩大
→ Queue Depth 上升
→ Worker 快速扩容
→ Graph 并发超过区域配额
→ 429 与重试增加
→ 有效吞吐下降
→ Queue Age 继续上升
```

**根因不是 AKS 计算不足，而是扩缩容没有受 Graph 配额约束，独立重试又放大了流量。**

应急处理分三步：先把 Deep Scan 最大并发降到 Graph 安全配额以内；暂停低优先级后台刷新，优先处理用户主动任务；将 429 Step 转成 `waiting_dependency`，按 `Retry-After` 加随机抖动重新调度，不占住 Worker。队列稳定后再逐步恢复后台扫描。

后续修复包括：

- Worker 扩容公式加入 Graph 全局并发令牌，而不是只看 Queue Depth；
- 将用户主动任务和后台刷新拆成不同优先级队列；
- 增加 Oldest Message Age、每个成功 Step 的 Attempt 数和重试率告警；
- 由 Step Owner 统一执行重试，取消各调用层独立重试；
- 容量测试同时模拟 Worker 总并发和真实 Graph 限流，而不是只压单 Pod。

**Worker 数量不等于系统吞吐。Agent 部署的容量上限通常取决于最慢且有配额的依赖，扩缩容、背压和重试必须一起设计。**

## 这部分能够回答的面试题

- [生产级 Agent 架构怎样设计？]({{ site.baseurl }}/docs/interview/ai-agent/system-production/#production-agent-architecture)
- [模型并发限制应该放在哪里？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-concurrency-limits)
- [模型 API 限流后怎样重试？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-rate-limit-retry)
- [消息积压时能否直接增加消费者？]({{ site.baseurl }}/docs/interview/backend/message-queue/#scale-consumers-for-backlog)
- [线程池、连接池和队列怎样共同限制吞吐？]({{ site.baseurl }}/docs/interview/backend/performance-production/#thread-connection-pool-queue)
- [指标、日志和 Trace 分别解决什么问题？]({{ site.baseurl }}/docs/interview/backend/performance-production/#metrics-logs-traces)
- [大模型应用应该记录哪些可观测数据？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-observability)
