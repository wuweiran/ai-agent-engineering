---
layout: default
title: 发布与生产运行
grand_parent: 工作经历
parent: Outlook Copilot Agent 能力开发
nav_order: 6
permalink: /docs/career/copilot-capabilities/production/
---

# 发布与生产运行

## 部署边界

Outlook Copilot 使用 Microsoft 365 Copilot Declarative Agent 平台。项目不部署自己的 Agent Runtime，也不拥有附件提取、邮件读取和邮件写入 Extension 的服务实例。平台和既有 Extension 团队分别负责：

- 模型、会话、Tool Call 循环和流式输出；
- Planning、计划状态和用户确认；
- Extension 调度、认证和平台 Trace；
- 附件提取、邮件读写和 Insight 等工具后端；
- 底层容量、扩缩容与服务值班。

我的交付物是 Outlook Declarative Agent 的能力配置：

- Agent Instructions；
- Side Panel Context 接入；
- 场景 Prompt 和 Context Builder；
- 当前场景启用哪些既有 Extension；
- 工具 Description、参数使用方式和 Tool Result 处理规则；
- 安全边界和发布配置。

```text
Outlook Side Panel
→ Declarative Agent Definition
   ├─ Instructions
   ├─ 场景 Context
   └─ Extension 引用
→ Microsoft 365 Copilot 平台
→ 已有 Extension
→ Tool Result
→ 模型继续回答或行动
```

因此，这个项目的“部署”主要是把源码仓库中的 Agent Definition、Instructions、Capability 配置和 Extension 声明构建成带版本的 Agent 发布包，再由 Microsoft 365 Copilot 平台托管和加载，而不是部署一套新的在线服务。代码与运行位置的完整划分见[Declarative Agent 接入与执行]({{ site.baseurl }}/docs/career/copilot-capabilities/runtime/#agent-代码以什么形式存在)。

## 为什么不在项目侧做缓存

项目不缓存模型回答、邮件正文、Tool Result 或动作计划。邮件内容、权限和资源版本会变化，跨用户复用存在数据泄露风险；写操作依赖最新对象状态，命中旧缓存反而可能扩大错误。平台和既有 Extension 可以在各自边界内缓存模型前缀、身份令牌或后端数据，但这些实现不属于本项目。

项目侧的优化重点是减少无效 Context 和 Tool Call，而不是缓存动态结果：复用稳定 Instructions 的平台能力、移除被新结果替代的 Tool Result、通过 Side Panel Anchor 缩小读取范围，并在需要时重新读取 Outlook 对象。

## 版本控制

一次 Agent 行为由以下版本共同决定：

| 版本 | 内容 | 归属 |
| --- | --- | --- |
| Agent Definition Version | Agent 名称、Description、Instructions 和 Extension 引用 | 项目配置 |
| Scenario Version | 划词解释、附件总结和邮箱整理的场景 Prompt 与行为边界 | 项目配置 |
| Context Version | Side Panel Context 映射、字段选择和 Token 预算 | 项目配置 |
| Extension Version | 工具 Schema、Description、错误契约和能力边界 | 依赖的 Extension |
| Model Deployment | 模型与平台推理配置 | Microsoft 365 Copilot 平台 |

每次请求和 Trace 记录版本清单：

```json
{
  "agentDefinitionVersion": "outlook-agent-42",
  "scenarioVersion": "mailbox-organization-v3",
  "instructionVersion": "prompt-v27",
  "contextVersion": "side-panel-context-v12",
  "extensions": {
    "search_outlook_context": "5.4",
    "extract_attachment": "7.2",
    "move_messages": "3.1"
  },
  "modelDeployment": "m365-copilot-prod-2025-10"
}
```

项目不修改已有 Extension 的 Handler。Extension 升级时，我们根据新版 Schema 和错误契约更新 Agent Definition 与 Instructions。破坏性变更先在兼容期内同时支持旧版和新版，再切换 Agent 引用。

## 发布流程

```text
Agent Definition 校验
→ Instructions 与 Extension Schema 契约检查
→ Copilot Evaluation 质量门禁
→ 内部 Ring
→ 5% 用户
→ 25% 用户
→ 100% 发布
```

### 配置与契约检查

发布前验证：

- Agent Definition 能被 Declarative Agent 平台加载；
- Instructions 没有冲突或越过平台安全边界；
- Extension 引用存在，Tool Schema 与 Prompt 中的参数语义一致；
- Tool Result 的成功、部分成功和错误语义都有对应处理；
- 写工具仍由平台确认策略保护；
- Side Panel Context 字段符合数据边界。

Agent 行为的完整离线评测在 SEVAL 中执行；邮件 Grounding Data、Golden Set、LM Checklist Assertion、业务定制 Metric 和质量门禁由 [Copilot Evaluation 与 Golden Set]({{ site.baseurl }}/docs/career/copilot-evaluation/) 维护，本项目不重复展开。

### 分阶段发布

平台按用户稳定分 Ring，同一会话保持在同一 Agent Definition。内部 Ring 验证通过后逐步扩大用户范围，每个阶段同时观察任务结果、工具行为、安全和性能，再决定继续或停止。

Agent Definition、Instructions 和 Context 配置独立版本化。一次尽量只修改一层；必须同时修改 Prompt、工具白名单和 Description 时，将它们作为一个可回滚配置版本发布，便于出现线上变化时定位。

## 停止与回滚

以下情况停止放量：

- 权限泄露、未确认写入或 Prompt Injection 越权；
- 写操作与平台确认范围不一致；
- `lm_checklist`、Citation 或工具选择低于发布基线；
- 简单场景的无效 Tool Call、模型轮次、延迟或 Token 明显高于上一版本；
- `result_unknown`、`partial_success` 或用户重新确认持续偏离基线；
- 关键 Extension 持续不可用，导致对应场景无法完成。

绝对阈值由当次模型部署、功能基线和 Copilot Evaluation 质量门禁共同确定，不把一组固定数字套用到所有 Capability 和模型版本。

回滚通过平台版本指针将 Agent Definition、Instructions 或 Context 配置切回上一版本。Extension 自身发生故障时，项目可以从 Agent Definition 暂时移除该工具或在 Instructions 中禁用相关能力，并与 Extension 团队协同恢复。一次具体的工具误选和灰度回滚见[工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)。

## 流式进度

流式传输由 Microsoft 365 Copilot 平台提供，项目不实现 SSE 通道。Side Panel 需要区分文本增量、工具执行、等待确认、完成和失败等平台事件，不能把 Tool Call 等待期间的静默误显示为任务结束。项目配置场景输出和终止行为，并使用平台 Trace 排查事件顺序；网络重连、事件投递和流控属于平台职责。

## Trace

Microsoft 365 Copilot 平台提供会话、模型调用、Planning、确认和 Tool Call 的基础 Trace。项目关注如何用 Correlation ID 串联 Agent 行为与已有 Extension 的结果：

```text
Copilot Conversation / Run
→ Agent Definition 与 Instructions Version
→ Model Turn
→ Tool Call ID / Extension Version
→ Tool Result
→ Platform Plan / Confirmation
→ Final Answer
```

项目使用的平台 Trace 字段包括：

- Conversation、Run 和 Model Turn；
- Agent Definition、Instructions、Context 和 Model Deployment；
- 输入与输出 Token、首 Token、完整生成延迟；
- Tool Call、Tool Result、Extension Version 和 Stop Reason；
- Planning、Replanning、确认和终止原因。

对于依赖工具，只记录 Tool Call ID、参数摘要、Extension Version、延迟和稳定错误语义。附件与邮件全文、选区和用户 Token 不进入项目日志。

## 线上指标

### 用户结果

- 任务完成率和用户重新提问率；
- 划词解释、附件总结和邮箱整理的成功率；
- Citation 点击率和答案反馈；
- 用户确认、取消和超时；
- 邮箱整理成功、部分成功和失败比例。

### Agent 行为

- 每次请求的模型轮次和工具调用数；
- Agent Definition、Instructions、Context、Extension 和模型版本分布；
- 工具选择、参数错误和无效 Tool Call；
- Planning、Replanning 和用户确认次数；
- 达到轮次、Token、时间或费用预算的比例；
- `partial_success`、`version_conflict` 和 `result_unknown`。

### Prompt 与 Context

- 输入与输出 Token；
- Context 中 Instructions、历史、界面 Context、工具 Schema 和 Tool Result 的占比；
- `partial` 和 `truncated` 比例；
- 首 Token和完整生成 P50/P95/P99；
- 按场景、Agent Definition 和 Model Deployment 统计的费用。

### 工具

- 每个工具的调用量、成功率、P95/P99、限流和重试；
- Tool Result 的成功、部分成功和稳定错误分布；
- 工具选择后被用户取消或重新确认的比例；
- Insight、附件提取和 Outlook 写工具的可用性。

平台或 Extension 团队负责底层资源指标。项目侧用这些依赖指标判断 Agent 质量变化是否来自 Prompt/Context，还是来自工具本身的故障。

## 值班排查

排障从 Copilot Run ID 开始寻找第一次偏离：

```text
入口、Utterance 与 Side Panel Context
→ Agent Definition / Scenario / Context Version
→ 模型第一次决策
→ Tool Call 与参数
→ Extension Version 与 Tool Result
→ 平台 Plan / Confirmation
→ Final Answer
```

- Context 或 Instructions 缺少必要信息，由项目团队修复；
- 候选工具集合、Description 或参数语义导致误选，由项目团队调整 Agent 配置；
- Tool Call 正确但 Extension 返回错误，与依赖团队协同；
- 平台没有按风险配置形成预期确认，与 Microsoft 365 Copilot 平台团队协同；
- Tool Result 正确但最终回答遗漏或幻觉，检查 Capability Prompt 和 Context 排序。

修复后把失败模式加入 Golden Set，先验证失败 Query，再运行 Capability 切片和全量回归，最后由 Ring 逐步恢复发布。Agent 开发与 Evaluation 的闭环不在生产事故后临时建立，而是在 2025 年两项工作重叠阶段成为固定流程。
