---
layout: default
title: 工作流系统设计
parent: Microsoft Purview 工作流
grand_parent: 工作经历
nav_order: 2
permalink: /docs/career/purview-workflow/system-design/
---

# 工作流系统设计

## 系统边界

Purview 团队并不维护底层工作流引擎。工作流的实例执行、节点调度和依赖推进由 **Azure Logic Apps** 提供，我们负责其上的 Purview 产品层与集成层：对外提供工作流 API，定义 Purview 使用的 Workflow Definition 和 Action，连接 Purview Catalog，并对 Logic Apps 的运行实例做一层简单包装。

后端采用 **Scala + ZIO**。Scala 服务承接 Purview Workflow API、Definition 处理、Action 实现和上下游调用；ZIO 用于组织这些异步和网络操作。

```text
Purview Catalog
→ 调用 Purview Workflow API 触发流程
→ Purview 服务创建或启动 Logic Apps 工作流
→ Azure Logic Apps 按 Workflow Definition 调度 HTTP Action
→ HTTP Action 调用 Purview Action 接口
→ Purview 执行任务并回调 Logic Apps
→ Logic Apps 根据结果继续调度
→ 固定的完成 Action 回调 Purview Catalog
→ Catalog 继续或完成原业务流程
```

## Action 执行协议

大部分 Purview Action 在 Logic Apps 中实际配置为 **HTTP Action**。Logic Apps 调度到节点后，调用我们配置的 Purview 接口，请求中说明 Action 类型和输入参数。Scala 服务根据 Action 类型找到对应实现，对支持 Expression 的 Parameter 求值，再执行具体业务逻辑。

Action 采用异步回调方式报告结果。Purview 完成任务后调用 Logic Apps 提供的回调接口，由 Logic Apps 将节点更新为成功或失败，并根据 `runAfter` 继续调度后续节点。

```text
Logic Apps HTTP Action
→ Purview Action API（Action 类型、Parameter）
→ Action Handler
→ Expression 求值与业务执行
→ Logic Apps Callback API（结果或错误）
→ Logic Apps 更新节点状态
```

这条协议中存在两个独立的网络边界：Logic Apps 调用 Purview Action API，以及 Purview 回调 Logic Apps。任一方向超时，都不能只根据调用方没有收到响应判断对方没有处理；可靠性设计需要关联工作流实例、Action 和本次执行请求。

## Workflow Definition

Purview Workflow Definition 参考了 Logic Apps 自身的定义方式，以分层 JSON 描述 Action，并通过 `runAfter` 表达节点之间的依赖关系。后端不需要自己扫描节点和计算下一步，而是把定义交给 Logic Apps，由 Logic Apps 根据 `runAfter` 和 Action 结果推进流程。

```json
{
  "actions": {
    "firstAction": {
      "type": "...",
      "parameters": {}
    },
    "secondAction": {
      "type": "...",
      "runAfter": {
        "firstAction": ["Succeeded"]
      },
      "parameters": {}
    }
  }
}
```

这只是结构示意。Purview 会在这套定义上增加自己的 Action 和 Parameter 约束，其中指定的 Parameter 可以保存 [Expression]({{ site.baseurl }}/docs/career/purview-workflow/expression/) 字符串，在 Action 执行时计算实际参数值。

## 实例包装

面向 Purview 用户暴露的工作流实例，不是团队自行实现的一套执行状态机，而是对 **Logic Apps 运行实例的简单包装**。Purview 层负责关联业务请求与底层运行实例，并把需要的状态和结果提供给产品界面及上游服务；真正的节点状态、依赖判断和调度仍由 Logic Apps 维护。

这层包装的意义是隔离产品和基础设施边界。Purview Catalog 不需要理解 Logic Apps 的全部接口和运行细节，Logic Apps 也不需要理解 Catalog 中触发工作流的业务对象。

## 与 Purview Catalog 的闭环

Purview Catalog 是重要上游。Catalog 中的业务事件需要启动工作流时，会调用 Purview Workflow API。工作流在 Logic Apps 中执行，中间 Action 完成审批、通知或其他任务。

流程末尾使用一个固定 Action 调用 Catalog 接口，通知上游任务已经完成。这样形成完整闭环：

```text
Catalog 触发业务流程
→ Workflow API 接受请求
→ Logic Apps 执行工作流
→ 固定 Action 回调 Catalog
→ Catalog 更新并继续原有业务流程
```

固定完成 Action 让回调位置和契约保持一致，也避免每份 Workflow Definition 分别实现一套完成逻辑。

## 团队负责什么

团队主要负责：

- Purview Workflow API 与 Catalog 集成；
- 参考 Logic Apps 结构定义 Purview Workflow Definition；
- Purview Action、Parameter 和 Expression 能力；
- Logic Apps HTTP Action 调用的 Purview 接口，以及执行完成后的 Logic Apps 回调；
- Logic Apps 运行实例在 Purview 产品层的包装；
- 固定完成 Action 与 Catalog 接口；
- Scala + ZIO 服务的测试、发布、监控和日常维护。

团队不负责 Azure Logic Apps 引擎本身，也不自行实现节点调度和完整的工作流实例状态机。这条边界很重要：这段经历的重点是**在托管工作流引擎之上建设领域能力和业务集成**，而不是从零开发工作流引擎。

底层能力由 Logic Apps 提供后，Purview 团队仍需从端到端系统视角理解节点状态、`runAfter` 调度、重复投递、执行权和故障恢复，并处理自身 Action 与上下游接口的超时、重试、幂等和结果未知。相关实践见 [工作流运行与可靠性]({{ site.baseurl }}/docs/career/purview-workflow/reliability/)。具体部署、存储、容量和线上事故见 [部署与生产运行]({{ site.baseurl }}/docs/career/purview-workflow/production/)。
