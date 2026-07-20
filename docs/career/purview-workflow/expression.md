---
layout: default
title: Workflow Expression
parent: Microsoft Purview 工作流
grand_parent: 工作经历
nav_order: 1
permalink: /docs/career/purview-workflow/expression/
---

# Workflow Expression

## 为什么需要 Expression

Purview 工作流需要服务不同企业的数据治理流程。客户会根据申请人、资产类型、分类级别和前序审批结果决定后续步骤，这些条件很难全部预先固化在产品代码中。

在 Expression 出现之前，每项客户定制需求都需要实现为一个新的 Workflow Action。一次交付要经过产品方案设计、前端配置界面开发、后端实现、测试和发布，周期长，而且每增加一种需求，产品中就会多出一项专用能力。Expression 的目标是把其中可以抽象的**动态数据处理和判断逻辑**变成 Action Parameter 中的字符串，由 Purview Action 执行层求值。客户可以组合已有函数满足不同需求，不必每次都开发新的 Action。

## 为什么自定义 Expression

设计前调研的方案主要分为三类：

- **通用脚本语言**：表达能力强，但权限、资源消耗和副作用难以约束，也不适合让工作流配置直接执行任意代码；
- **JSONPath、JMESPath 一类数据查询语言**：适合从结构化数据中选取字段，但不足以覆盖函数组合、条件计算以及 `getUserName`、`getManager` 这类领域能力；
- **现有工作流 Expression**：Azure Logic Apps 和 Power Automate 的函数式表达式是重要参考，Purview Workflow Definition 本身也参考了 Logic Apps 的 JSON 层级和 `runAfter`。但 Purview 仍需要定义哪些 Action Parameter 支持动态值，并接入 `getUserName`、`getManager` 等领域函数，因此没有直接照搬完整的函数集合和求值语义。

最终选择定义一套范围较小的领域语言，由 Purview Action 执行层负责解释。它可以直接识别 `runInput()`、`outputs('action')` 等工作流概念，也允许后端注册 Purview 需要的函数。

这里的“自定义”主要是**定义自己的语言契约和运行时语义**，不是从零编写所有基础设施。语法由 EBNF 描述，前端使用 ANTLR，Scala 后端使用 scala-parser-combinators。这样既能控制语言边界，也复用了成熟的解析工具。

## Expression 的形式

Expression 本质上是一个符合指定语法规则的单行字符串。它不是 Workflow Definition 中独立的数据结构，而是作为某个 Action 的 Parameter 值保存在定义中；由我们在 Action 的能力定义中指定哪些 Parameter 支持 Expression。字符串语法由 EBNF 统一定义，明确函数调用、字面量、数据引用、括号和属性访问怎样组合。一个实际表达式例如：

```text
and(equals(runInput()['requestor'],"1234"),contains(outputs('action')['level'],"Confidential"))
```

这段表达式同时读取两类运行时数据：

- `runInput()` 取得当前工作流实例的触发输入；
- `outputs('action')` 取得指定前序节点的输出；
- `equals`、`contains` 和 `and` 对数据进行比较和组合。

它不是允许客户执行任意代码的通用脚本。表达式只能使用系统注册的函数，但这些函数不局限于本地计算：开发者可以在后端自定义并实现 `getUserName`、`getManager` 等函数，由函数封装对用户和组织信息服务的网络调用。客户只能使用已经开放的函数，不能在 Expression 中自行指定地址或编写网络请求。

## Expression 在系统中的位置

Expression 位于 Workflow Definition 中的 **Action Parameter**。例如，一个 Action 可以有多个 Parameter，其中只有我们预先定义为支持 Expression 的 Parameter 才会把字符串交给 Expression Engine；普通 Parameter 仍按原值处理。

```text
Logic Apps 根据 runAfter 调度 HTTP Action
→ 调用 Purview Action API 并传入 Parameter
→ Evaluator 计算支持 Expression 的 Parameter
→ Purview 使用实际参数值执行 Action
→ 回调 Logic Apps 报告结果
```

定义保存或发布时，Purview 后端解析并验证支持 Expression 的 Parameter。Logic Apps 执行到相应 HTTP Action 后，调用 Purview Action API 并传入 Action 类型和 Parameter；Purview 执行层调用 Evaluator，使用求值结果完成任务，随后通过 Logic Apps 的回调接口报告 Action 结果。

## 核心实现

### 统一语法与前端编辑

我使用 **EBNF** 定义 Expression 字符串的唯一语法，规定函数调用、参数、字面量、数据引用和嵌套结构的合法形式；同时定义哪些 Action Parameter 支持这种字符串。前端根据这份语法使用 **ANTLR** 生成 Parser，并与 **Monaco Editor** 集成，在编辑相应 Parameter 时提供语法高亮和即时错误提示。

前端解析用于改善编辑体验，后端仍要独立解析和验证。两端虽然使用不同的 Parser 实现，但遵守同一份 EBNF；语法升级还要保证已经保存的旧 Expression 继续有效。

### Scala 后端解析与验证

我基于 **scala-parser-combinators** 实现后端 Parser，将单行 Expression 转换成 AST。Validator 再基于 AST 检查函数、参数和数据引用。后端解析是保存和执行链路中的可信边界，不能直接采信前端的解析结果。

部分数据只在工作流运行时出现，设计阶段只能验证引用形式，实际取值问题由运行时返回明确的求值错误。

### 运行时求值

Evaluator 遍历 AST，并从当前工作流实例取得运行输入和前序节点输出。`equals`、`and` 等函数在本地计算，`getUserName`、`getManager` 等函数可以调用后端服务取得用户和组织关系数据。

求值需要区分正常结果、类型错误、引用不存在、网络超时和下游失败。网络调用由系统注册的函数封装，包括服务地址、身份认证、权限、超时和返回值转换；Expression 只能传入参数并使用结果，不能自行发起任意网络请求。求值结果交给当前 Purview Action 使用；Action 完成后，Purview 通过回调接口把结果报告给 Logic Apps，再由 Logic Apps 按 Workflow Definition 中的 `runAfter` 推进。

## 我负责的边界

作为 Workflow Expression 领域的 Owner，我负责的不只是语法解析代码，还包括整项能力的产品和运行时边界：

- 使用 EBNF 定义唯一的 Expression 字符串语法，并定义哪些 Action Parameter 支持 Expression；
- 维护前后端解析的一致性与兼容性，配合前端使用 ANTLR 和 Monaco Editor 实现语法高亮与错误提示；
- 使用 scala-parser-combinators 实现后端 Parser 和 AST；
- 设计函数、数据引用和返回类型，并完成验证与后端求值；
- 设计自定义函数的注册与扩展方式，并支持函数封装后端网络调用；
- 将 Expression 接入工作流定义的保存、发布和运行链路；
- 将 Expression 作为运行时动态参数能力接入后来的 Custom Action；
- 处理兼容性、错误语义、下游失败、测试和后续扩展。

这项工作的关键取舍是**可配置性与可控制性**。Expression 要足以覆盖高频客户需求，包括通过受控函数取得外部数据；同时不能演变成客户可以任意编写网络请求、难以验证和难以排查的脚本平台。

## 复用方式与交付周期

Expression 把每个 Action 都可能需要的动态能力抽成了统一层：支持 Expression 的 Parameter 使用相同的语法、Parser、Validator 和 Evaluator；`getManager` 等领域函数注册一次后，可以被不同 Action 和不同客户的工作流组合使用。新增需求的实现单位因此从“开发一个完整 Action”变成了“组合已有函数”，能力不足时也通常只需增加一个受控函数。

Expression 后来直接支撑了 **Custom Action**。用户可以自定义 Action，其中需要在运行时变化的 Parameter 值由 Expression 根据工作流输入、前序节点输出或外部数据计算。Custom Action 不需要重新实现一套动态参数机制。

客户需求形成了三种不同的处理方式：

| 需求类型 | 主要工作 | 典型交付周期 |
| --- | --- | --- |
| 新建专用 Action | 产品设计、前端、后端、端到端测试和服务发布 | 约 4～8 周 |
| 使用已有 Expression 函数组合 | 用户自行组合函数并配置到 Action Parameter | 无需工程交付 |
| Expression 需要新增领域函数 | 定义函数契约，实现后端调用、验证、测试和发布 | 约 1 周 |

表中的工程交付周期指从需求范围确认到能力在目标环境可用的 **Lead Time**，不是 Expression 的运行延迟。已有函数足以表达需求时，用户可以自行完成配置，不产生新的工程交付周期；需要新增函数时，再通过需求工单、代码变更和发布记录统计，通常约一周。与开发完整 Action 相比，它不再需要为每项需求重复完成产品和前端设计，改动范围明显更小。
