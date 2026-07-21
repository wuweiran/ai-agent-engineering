---
layout: default
title: 发布与生产运行
grand_parent: 工作经历
parent: Outlook Copilot Agent 能力开发
nav_order: 6
permalink: /docs/career/copilot-capabilities/production/
---

# 发布与生产运行

## 发布的是什么

项目不部署自己的 Agent Runtime 或 Extension 服务。实际发布的是带版本的 Agent Definition，包括 Instructions、三项 Capability 配置、Side Panel Context 映射和 Extension 引用。发布包由 Microsoft 365 Copilot 平台托管和加载。

已有 Extension 的容量、扩缩容和服务值班由对应团队负责。项目值班主要处理 Agent 配置、Context、工具选择和最终回答的问题。

## 发布前检查

一次发布按下面的顺序进行：

```text
Agent Definition 和 Extension 引用校验
→ 失败 Query 验证
→ Capability 切片回归
→ 全量 Golden Set
→ 内部 Ring
→ 逐步扩大范围
```

配置检查只关注实际会影响行为的项目：Agent Definition 能否加载，场景是否引用正确工具，Prompt 参数是否与工具 Schema 一致，写工具是否仍受平台确认保护。

完整离线评测在 SEVAL 中完成。当前发布时主要看五个数值：

| Metric | 当前基线 |
| --- | ---: |
| `lm_checklist` | 87.6% |
| `citation` | 96.2% |
| `tool_call` | 93.8% |
| `safety` | 0.0% |
| `latency_p95` | 7.4 秒 |

严重安全违规必须保持为零。其他指标除了比较整体基线，还会检查同一 Query 是否从通过变成失败，避免平均值掩盖单个能力回归。

## 版本和回滚

Agent Definition、场景 Prompt 和 Context 配置跟随同一个发布版本，Trace 同时记录当前 Agent、Extension 和模型版本。这样线上出现变化时，可以先判断是本次 Agent 配置、依赖 Extension，还是平台模型发生了变化。

Ring 中出现未确认写入、权限泄露或明显的工具误选时停止放量，并把 Agent Definition 切回上一版本。Extension 故障时，项目可以暂时从场景中移除对应工具，但不能修改依赖团队的 Handler。

没有采用固定的 5%、25%、100% 作为所有版本的发布比例。实际放量范围由功能风险、当次回归结果和平台发布安排决定。

## 线上观测

项目日常关注少量直接指标：

- 三项 Capability 的请求量、成功和用户重新提问；
- 每次请求的模型轮次、Tool Call 数量和输入输出 Token；
- 工具选择错误、`version_conflict`、`partial_success` 和 `result_unknown`；
- 首 Token 和完整生成延迟；
- 用户确认、取消和计划修改。

底层 CPU、内存、连接池和工具服务可用性属于平台或 Extension 团队。项目只使用这些依赖指标判断问题是否来自工具后端。

## 排查方法

线上问题从 Copilot Run ID 开始，依次检查入口和 Side Panel Context、Agent 版本、模型第一次决策、Tool Call、Tool Result、平台确认和最终回答。

- 首轮就选错工具，检查 Capability Prompt、工具列表和 Description；
- Tool Call 正确但返回错误，与对应 Extension 团队协同；
- Tool Result 正确但回答遗漏，检查 Prompt 和进入本轮的证据；
- 平台确认行为异常，与 Microsoft 365 Copilot 平台团队协同。

Trace 只记录参数摘要、资源 ID、版本、延迟和错误状态，不记录完整邮件、附件、选区或用户 Token。修复后先验证原失败 Query，再跑 Capability 切片和全量 Golden Set，最后重新进入 Ring。
