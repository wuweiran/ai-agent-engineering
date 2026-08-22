---
layout: default
title: Context Builder 与 Token Budget
grand_parent: 工作经历
parent: My Outlook Agent 部署
nav_order: 2
permalink: /docs/career/my-outlook-agent/context-builder/
---

# Context Builder 与 Token Budget

这部分展开 [My Outlook 项目概览]({{ site.baseurl }}/docs/career/my-outlook-agent/#dynamic-context-assembly)中的动态 Context 设计，重点说明 Model Input 怎样从持久状态和外部 Artifact 重建，而不是依赖不断增长的聊天记录。

## Context Builder 在 Agent Loop 中的位置

My Outlook 自己运行 Agent Loop。每次需要调用模型时，POS Service 不会把 Conversation、Deep Scan Finding、Synthesis Artifact 和全部 Worker Result直接拼接，而是先根据当前 Step 生成 `ContextBuildRequest`，再由 Context Builder 构造这一次 Model Input。POS 随后把 Model Input Reference 作为参数写入 Model Step，由 Worker 调用模型。

```text
Agent Loop
→ 根据 Task State 确定当前 Step 和需要回答的问题
→ Context Builder 选择、替换、压缩 Context Blocks
→ Token Counter 检查预算
→ LLM Worker 调用模型
→ 模型结果与引用写入 Step Result
→ Agent Loop 决定下一步
```

代表性的输入和输出是：

```text
ContextBuildRequest
├─ task_id、run_id、step_id、step_type
├─ 当前用户输入和最近消息位置
├─ Task State 与完成条件
├─ 可用 Finding / Artifact / Worker Result Reference
├─ model_context_limit
└─ reserved_output_tokens

ContextBuildResult
├─ ordered_blocks[]
├─ input_token_count
├─ included_refs[] / excluded_refs[]
├─ summary_version
└─ diagnostics
```

**Context Builder 只生成本轮模型工作集，不修改或删除持久状态。**Conversation Store、Task Store 和 Artifact Store 维护各自的数据，所以某条内容本轮没有进入 Context，不代表被永久删除。

## 为什么需要分层存储

不同信息具有不同权威性和生命周期：

| 信息 | 保存位置 | 进入 Context 的方式 |
| --- | --- | --- |
| 用户和 Agent 消息 | Conversation Store | 最近几轮原文，较早部分使用摘要 |
| 目标、确认事实、当前 Step、待办 | Task Store | 固定保留的结构化 Task State |
| Deep Scan 候选发现 | Annotation Store | 按当前目标、时间和置信信息筛选 |
| Briefing、Recommendation、Draft | SDS Artifact Store | 按 Artifact 类型、版本和引用选择 |
| 邮件、日历和联系人原文 | Microsoft 365 权威系统 | 需要核实时由 Graph/API Worker 按 ID 读取 |
| Worker 执行结果 | Task Store 中的 Step Result | 只保留仍影响后续判断的最新结果 |

**结构化 Task State 不能依赖摘要，完整邮件也不复制进 Conversation。**摘要有损，邮件可能变化、体积大且受权限控制；Artifact 保存生成结果和 Citation，使模型不必每轮重新处理全部原始材料。

## Token Budget 怎样分配

**Token Budget 不是简单截断消息尾部，而是按信息权威性和当前步骤分配。**Context Builder 先从模型窗口扣除输出预留和工具 Schema，再给各类 Block 分配软预算：

```text
可用输入预算
= 模型 Context Window
- reserved_output_tokens
- System Instructions
- Tool Schema
- safety_margin
```

剩余预算按优先级分配：

1. 当前用户输入、任务目标、确认约束和完成条件；
2. 当前 Step 必须使用的证据和最新 Worker Result；
3. 最近几轮 Conversation 原文；
4. 与当前目标相关的 Finding 和 Artifact；
5. 较早历史摘要与低优先级候选。

每一类配置软上限而非固定比例。执行写操作时，资源 ID、版本和确认信息优先于历史说明；生成 Briefing 时，相关 Finding 与 Citation 可以获得更多预算。若高优先级内容本身超过窗口，Builder 返回 `context_overflow`，由 Agent Loop 缩小任务或拆分 Step，不能静默截掉完成任务必需的状态。

## 选择、替换和压缩

Context Builder 按三类规则处理信息。

### 选择

Finding 先按租户、用户、权限、时间范围和当前目标过滤，再按相关性、优先级和新鲜度排序。只把少量候选摘要放进 Context；需要原文时返回资源 ID，让 Graph/API Worker按需读取。这样降低 Token，也避免大量候选把关键证据挤到 Context 中间。

### 替换

同一对象只保留当前有效版本：

- 邮件详情替换对应的 Snippet；
- 新 Synthesis Artifact 替换旧 Artifact Version；
- 新 Worker Result 替换同一 Step 的旧 Attempt Result；
- 用户修改目标后，新 Task Constraint 覆盖旧约束，并让依赖旧约束的候选失效。

**替换依据稳定 Key 和 Version，不通过文本相似度猜测。**

### 压缩

超过预算时按固定顺序处理：去掉重复 Citation 和已消费结果；删除过期或低相关候选；把较早 Conversation 压缩为 `confirmed_facts`、`decisions`、`pending_items` 和 `unresolved_questions`；近期消息保留原文。

摘要保存来源消息范围和版本。新消息与摘要冲突时，以用户最新确认或权威系统为准，并重新生成摘要。模型假设不会自动进入 `confirmed_facts`。

## Worker Result 契约

Worker Result 如果返回整份 Graph Response 或模型内部诊断，会迅速撑大 Context。统一契约只保留 Agent Loop 需要的字段：

```json
{
  "stepId": "step-184",
  "status": "succeeded",
  "artifactRefs": ["sds://briefing/982?v=3"],
  "evidenceRefs": ["message:AAMk...", "event:AAQk..."],
  "summary": "Three commitments need follow-up this week.",
  "warnings": [],
  "retryable": false
}
```

大型提取结果和生成产物写入 SDS，Result 只携带 Reference 和短摘要。错误结果区分暂时失败、权限拒绝、结果未知和永久失败，让 Agent Loop 决定等待、换路或终止，而不是把堆栈交给模型解释。

## 怎样验证裁剪没有伤害任务

**Context 优化不能只看 Token 下降，必须同时验证任务完成率和 Citation。**离线测试检查：

- 早期确认的目标和限制是否仍被保留；
- 用户改口后旧约束是否失效；
- 当前步骤需要的 Finding、Artifact 和 Citation 是否进入 Context；
- 摘要是否把假设误写成事实；
- 裁剪前后任务完成率和 Citation 是否下降；
- 每个成功任务的总输入 Token 是否改善。

线上 Trace 不保存完整邮件正文，而是记录 Block 类型、Reference、Token 数、选择或丢弃原因、摘要版本和最终模型调用 ID。发生错误时，可以区分“证据没有被 Deep Scan 找到”“Context Builder 没选中”“选中了但模型没有使用”三类根因。

## 典型问题：候选过多挤掉当前任务状态

早期版本按 Finding 分数直接填满剩余窗口。用户连续对话后，Deep Scan 候选与 Briefing Artifact 数量增长，当前目标和最近一次修改虽然仍在输入中，却被大量候选分散，Agent 偶尔继续执行用户已经取消的旧目标。

**根因不是窗口不够，而是 Task State 没有不可挤占的预算。**修复后，当前目标、已确认约束、当前 Step 和完成条件成为固定 Block；候选只能使用剩余软预算，用户修改目标时通过 Constraint Version 使旧候选失效。回归集增加长对话、用户改口和大量 Finding 的组合用例，再比较任务完成和 Token，而不是只确认请求没有超过模型窗口。

## 这部分能够回答的面试题

- [Agent Context Engineering 有哪些主要技术？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-context-engineering-strategy)
- [一次 Agent 调用的 Context 怎样组装？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-context-assembly)
- [Context 超出限制时怎样处理？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#context-overflow-window-summary)
- [多轮对话的存储和记忆怎样设计？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#conversation-storage-memory)
- [Context 与任务状态有什么区别？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#context-vs-task-state)
- [大模型应用怎样控制 Token、延迟和成本？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-token-latency-cost)
