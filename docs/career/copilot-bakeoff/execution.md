---
layout: default
title: Scraping 与跨系统执行
grand_parent: 工作经历
parent: Copilot Bake-off
nav_order: 1
permalink: /docs/career/copilot-bakeoff/execution/
---

# Scraping 与跨系统执行

## 执行边界

Outlook Copilot 已经接入内部 Evaluation 链路，可以直接从 Golden Set 运行 Query。Gmail Gemini 没有供项目调用的对等评测接口，所以 Bake-off 的核心是使用 Playwright 操作真实 Gmail Gemini 页面，取得最终 Response。这个过程内部称为 scraping。

Scraping 不是爬取公开网页，也不是绕过用户权限。它使用隔离的 Google Workspace 测试账号和已导入的测试邮件，模拟测试用户在产品页面中的正常操作。

## 运行方式与系统形态

Bake-off 不是 SEVAL 提供的能力。Golden Set 裁剪、Grounding Data ingestion、跨系统 Mapping、Outlook 执行、Playwright scraping、批量调度、失败重试和两侧 Response 汇总，全部由 Outlook Team 搭建和维护。只有在两侧 Response 已经准备完成后，才把评分输入提交到 SEVAL 运行 `lm_checklist`。

```text
Outlook Team Bake-off System
├─ 选择 Golden Set Version 与 Bake-off 标签
├─ Grounding Data ingestion / Mapping
├─ Outlook Evaluation Runner
├─ Playwright Scraping Workers
├─ Query 调度、重试与状态管理
└─ 两侧 Response 配对与导出
                ↓
SEVAL Job
└─ 使用同一 Assertion 运行 lm_checklist
                ↓
Outlook Team 汇总并分析比较结果
```

正式对比由项目成员在 Outlook Team 的 Bake-off 系统中手动创建 Run，通常在以下情况运行：

- Outlook Copilot 有重要模型、Prompt、Context 或功能版本需要做竞品对比；
- Gmail Gemini 出现明确的产品或模型版本变化；
- Golden Set、Grounding Data 或跨系统 Mapping 完成一轮更新；
- 阶段性产品评审需要生成新的对比报告。

不在每次发布后自动跑全量，原因是 Gmail Gemini 没有可订阅的统一发布事件，Playwright scraping 和 Google Workspace ingestion 成本也明显高于普通离线评测。Outlook 自身的发布质量门禁继续使用 Copilot Evaluation 的 Golden Set 回归，不依赖 Bake-off。

Outlook Team 的 scraping 基础设施另有一组定时运行的小型 smoke suite。它只使用固定账号和少量 Query，检查登录状态、Gmail 邮件定位、Gemini 输入、生成完成判定和 Response 提取，目的是尽早发现页面变化，不生成正式 Bake-off 结论。

因此自动化分成两类：

- **自动 smoke**：由 Outlook Team 系统验证 scraping 链路是否还能工作；
- **手动 full run**：在 Outlook Team 系统中固定本次对比范围和版本，自动执行完整 Bake-off，最后调用 SEVAL 评分。

手动的是“何时用哪一组版本启动一次对比”，不是手动逐条操作浏览器。Run 启动后，Query 调度、浏览器执行、重试、状态收集和 Response 配对由 Outlook Team 系统自动完成；SEVAL 只负责最后的 LM Checklist Job。

## Golden Set 裁剪

Bake-off 不直接运行完整 Outlook Golden Set。每条 Query 带有功能和 Bake-off 适用标签，执行前只选择 Outlook Copilot 与 Gmail Gemini 都有对应功能的 Query：

```text
完整 Outlook Golden Set
→ 读取功能标签和 Bake-off 标签
→ 排除任一侧没有对应功能的 Query
→ 得到 Bake-off Golden Set 子集
→ 解析子集关联的 Grounding Data 与 Assertion
```

功能支持情况在数据准备阶段确定，不根据某次产品回答动态判断。Gmail Gemini 没有对应功能的 Query 不创建 scraping 任务、不进入最终配对评分集合，也不记为 Gemini 失败。

这个裁剪保证比较范围公平，但 Bake-off 结果只代表双方能力交集，不能据此评价某一产品独有功能，也不能把裁剪后集合的通过率当成完整 Outlook Golden Set 的通过率。报告记录完整 Golden Set Version、筛选标签和裁剪后 Query 数量，使每次对比的能力范围可以复现。

## Grounding Data ingestion

Bake-off 复用裁剪后 Query 关联的 Outlook Grounding Data，但 Google Workspace 不能使用 Outlook 的 User ID、Email ID 和邮件存储。只同步 Bake-off 子集实际引用的邮件，避免把完整评测邮箱无差别复制到 Google Workspace。执行前需要把邮件转换并导入测试 Workspace：

```text
Bake-off 子集关联的 Outlook Grounding Data
→ 读取邮件、参与者和关联关系
→ 映射测试用户
→ 转换邮件与线程数据
→ 导入 Google Workspace
→ 验证 Gmail 中的用户、邮件和可见性
→ 保存跨系统 Mapping
```

导入后必须保持影响任务语义的字段：

- Sender 与 Recipients 的角色关系；
- Subject、Body 和时间；
- 邮件之间的回复或线程关系；
- Query 需要使用的附件；
- 测试用户能看到哪些邮件。

Outlook 与 Gmail 的底层格式和线程规则不同，因此目标是语义等价，不是复制内部对象结构。

## 三类 Mapping

### User Mapping

```text
Outlook Test User ID
↔ Google Workspace User
```

User Mapping 保持“我”“发件人”“收件人”和组织角色的含义。Utterance 中如果依赖当前用户身份，Gemini 侧必须登录映射后的 Workspace 用户。

### Email Mapping

```text
Outlook Email ID
↔ Gmail Message / Thread
```

CIQ 中的 Outlook Current Email ID 不能直接交给 Gmail。Scraping 执行前根据 Email Mapping 找到 Gmail 中对应的邮件或线程，再通过 UI 打开它。

### Golden Set Mapping

```text
Outlook Query
├─ CIQ / Current Email ID
├─ Utterance
└─ Assertion
        ↓
Gemini Query
├─ 映射后的 Gmail 邮件入口
├─ 语义相同的 Utterance
└─ 同一组 Assertion
```

Golden Set Mapping 记录一条 Query 在两边的执行入口。Utterance 和成功标准保持一致，只有产品所需的 Context 入口不同。

## Playwright scraping

一个 Gemini Query 的执行步骤是：

```text
创建隔离 Browser Context
→ 登录映射后的 Workspace 测试用户
→ 打开 Gmail
→ 根据 Email Mapping 定位并打开邮件
→ 校验当前邮件
→ 打开 Gemini 面板
→ 记录提交前页面状态
→ 输入并提交 Utterance
→ 等待新的 Response 开始生成
→ 判断生成结束
→ 提取文本结果
→ 保存执行状态、Screenshot 和耗时
```

稳定性不依赖固定时间的 `sleep`，而是把页面操作建模为状态机。每一步只有在前置状态满足后才能继续：

```text
Gmail Ready
→ Target Email Opened
→ Gemini Ready
→ Prompt Submitted
→ Response Started
→ Response Streaming
→ Response Completed / Product Error / Scraping Error
```

### 页面就绪与邮件校验

Gmail 是长连接应用，不能使用 `networkidle` 判断页面完成，因为后台请求可能一直存在。执行器只等待当前步骤需要的业务元素：Gmail 主界面、目标邮件容器、Gemini 入口和输入框。

Email Mapping 保存 Gemini 侧对应的 Gmail Message 或 Thread。打开后再检查 Subject、Sender 和测试数据标识，确认页面确实是当前 Query 对应的邮件。邮件不存在、当前用户不可见或打开了错误线程时停止执行，分别记录 `mail_not_visible` 或 `email_mismatch`，不能继续向 Gemini 提问。

### 元素定位

定位器集中封装在 Page Object 中，按以下优先级选择：

1. 可访问性 Role 与稳定的 Accessible Name；
2. `aria-label` 等语义属性；
3. 页面中稳定的文本和结构锚点；
4. 最后才使用局部 CSS Selector。

不依赖动态 CSS 类名、完整 DOM 层级或列表序号。Gemini 页面变化时只更新 Page Object，不把 Selector 散落在每条测试中。关键 Locator 在执行前同时检查可见、可交互和唯一性；匹配多个元素也按 `selector_ambiguous` 失败，避免点击错误按钮。

### Gmail Gemini 界面更新

Gmail 和 Gemini 由外部产品团队持续更新，Playwright 无法保证 Selector 永久有效。稳定性的目标是尽早发现变化、把修改限制在适配层，并在误操作前停止 Run。

页面适配集中在版本化的 Page Object 中：

```text
Bake-off Scenario
→ GmailPage
→ GeminiPanel
→ ResponseView
→ Locator Set / UI Variant
```

业务执行器只调用 `openMappedEmail`、`openGemini`、`submitUtterance` 和 `readCompletedResponse` 等语义方法，不直接操作 Selector。每次 Page Object 变更生成新的 Scraping Version，正式报告记录使用的版本。

每个关键组件可以维护少量按优先级排列的 Locator：

```text
Gemini 输入框
1. Role + Accessible Name
2. 稳定 aria-label
3. 组件内部语义锚点
```

Fallback 只在前一个 Locator **匹配零个元素**时使用。任何 Locator 匹配多个元素，或者多个候选同时出现，都立即返回 `selector_ambiguous`，不自动点击“最像”的元素。Fallback 成功也记录命中的是哪一组 Locator，便于发现主 Locator 已经失效。

Google 可能按账号或 Region 分批发布不同 UI。Scraping 启动时先运行 Variant Detection，根据 Gemini 入口、输入区域和 Response 容器的稳定特征识别已支持的 UI Variant：

```text
检测 UI Variant
├─ gmail-gemini-v1 → Page Object v1
├─ gmail-gemini-v2 → Page Object v2
└─ unknown         → 阻断该账号
```

同一个正式 Run 固定允许的 UI Variant 集合。检测到未知 Variant 时不边跑边切换 Selector，而是停止受影响账号，避免一部分 Query 使用旧界面、一部分使用新界面，造成不可解释的对比结果。

界面更新通过两层检查发现：

1. **定时 smoke**：固定账号和固定 Query 检查登录、邮件打开、Gemini 输入、生成完成和 Response 提取；
2. **full run preflight**：正式 Run 前，每个测试账号先执行一条只验证链路的冒烟 Query，全部通过后才开始批量执行。

如果同一 Scraping Version 在连续 Query 中出现相同阶段的失败，例如 `selector_not_found` 或 `response_extract_failed`，Runner 达到阈值后对该 UI Variant 熔断，不继续消耗剩余 Query。已完成结果保留，但本次 Run 标记为基础设施失败，不进入产品对比。

修复过程是：

```text
Smoke 告警或 Variant 熔断
→ 查看 Screenshot、DOM Snapshot 和 Locator 诊断
→ 在 Page Object 中增加或更新 Variant
→ 用固定 smoke suite 验证
→ 发布新的 Scraping Version
→ 重新运行 preflight
→ 重启正式 Run
```

失败证据只来自隔离测试账号。DOM Snapshot 会裁剪到相关组件并去除测试账号信息和邮件正文，避免为了修 Selector 保存不必要的 Grounding Data。

### 识别新的 Response

提交前记录当前 Gemini 对话中的 Response 数量和最后一条 Response 的文本摘要。输入 Utterance 后，通过提交按钮发起请求，并等待以下任一信号确认提交成功：

- 输入框被清空；
- Utterance 出现在当前对话中；
- Response 容器数量增加；
- 页面进入生成中状态。

后续只监控新出现的 Response 容器，不能直接读取页面中最后一个已有节点，否则可能把上一条 Query 或上一轮回答当成本次结果。

### 判断生成结束

Gemini Response 是流式更新的，看到非空文本只能说明生成已经开始。执行器每 **500 毫秒**读取一次新 Response 的规范化 `innerText`，同时观察生成状态。只有以下条件全部满足才标记为完成：

1. 新 Response 已出现且文本非空；
2. 生成中标记或停止生成按钮已经消失；
3. 输入框重新可用，Response 操作区已经出现；
4. 规范化文本连续 **4 次**保持不变，即稳定至少 **2 秒**；
5. 页面没有出现限流、重试、登录失效或产品错误提示。

规范化只移除流式渲染产生的首尾空白，不改写正文内容。最终提取前再读取一次文本，并确认与稳定窗口中的最后版本一致。

单条 Query 的生成超时为 **90 秒**。达到上限时：

- 仍显示生成中，记录 `generation_timeout`；
- 生成状态已经结束但 Response 为空，记录 `response_empty`；
- Response DOM 存在但无法提取文本，记录 `response_extract_failed`；
- 页面明确显示 Gemini 无法回答或产品错误，保留为产品结果，而不是 scraping 失败。

超时取得的半段文本不进入 LM Checklist，避免将截断回答当成完整产品结果。

### 重试边界

导航失败、元素尚未加载等错误，只能在 Utterance 提交前刷新页面并重试一次。提交后如果结果未知，不直接再次发送 Utterance，而是先检查当前 Gemini 对话中是否已经出现对应请求和 Response；仍无法确定时记录 scraping 失败。

这个边界可以避免一次 Query 被 Gemini 实际执行两次，也避免自动重试选择“最好的一次”结果，破坏 Bake-off 的公平性。

### 会话隔离与并发

登录状态按 Google Workspace 测试用户保存，但不同 Query 使用独立 Browser Context，或在执行前创建新的 Gemini Conversation，防止上一条 Query 的历史污染结果。多轮 Scenario 只在内部复用会话，Scenario 结束后立即重置。

同一个测试用户的 Query 串行执行，不让两个 Browser Page 同时修改 Gmail 和 Gemini 状态；不同测试用户之间可以并行。每次执行记录 Query ID、测试用户、Mapping Version、Scraping Version 和页面状态阶段。

### 失败证据

Scraping 失败时保存失败阶段、错误码、UI Variant、Scraping Version、当前 URL、Screenshot、裁剪并脱敏的 DOM Snapshot、Locator 诊断和页面状态摘要。排查首先确定失败发生在邮件打开、Gemini 初始化、Prompt 提交、流式生成还是 Response 提取，不能只看最终的 Playwright Timeout。测试账号中只包含评测 Grounding Data，产物不包含真实用户凭据。

## 失败分类

执行结果分为三层：

### 数据与映射失败

- `user_mapping_missing`；
- `email_mapping_missing`；
- `ingestion_failed`；
- `mail_not_visible`。

### scraping 失败

- `login_expired`；
- `selector_not_found`；
- `selector_ambiguous`；
- `unsupported_ui_variant`；
- `email_mismatch`；
- `page_timeout`；
- `generation_timeout`；
- `response_empty`；
- `response_extract_failed`。

### 产品有效结果

只有成功提取到 Outlook 与 Gemini 两侧 Response 的 Query 才进入配对质量评分。页面显示 Gemini 自身无法完成任务或产品错误时，要保留为产品结果；Playwright 没能操作页面则属于 scraping 失败。两者必须分开，否则自动化稳定性会被误当成模型质量。

## Response 配对与 SEVAL 评分

Outlook Team 的 Bake-off 系统先完成两侧执行、过滤无效 Run，并生成统一的 Response 记录：

```json
{
  "queryId": "query-1042",
  "outlook": {
    "status": "success",
    "response": "..."
  },
  "gemini": {
    "status": "success",
    "response": "..."
  },
  "assertionVersion": "assertion-v12",
  "mappingVersion": "workspace-map-v7"
}
```

Bake-off 系统将两侧 Response、对应 Grounding Data 和同一组 Assertion 组装成 SEVAL 能接受的评分输入，创建最后一步的 SEVAL Job。SEVAL 只运行 `lm_checklist` 并返回每条 Response 的 Assertion 结果，不参与 scraping、两侧执行或产品比较逻辑。

Outlook Team 取得评分结果后，再按 Query 生成配对结果：

```text
Outlook pass / Gemini fail
Outlook fail / Gemini pass
Both pass
Both fail
```

不能把两侧分别跑出的平均分直接对比，因为有效 Query 集合可能不同。Bake-off 系统先取两侧都成功执行的 Query 交集，再提交评分并做配对统计；数据和 scraping 失败率单独报告。

## 可复现性

一次 Bake-off 结果需要固定：

- Outlook Golden Set Version、Bake-off 筛选标签和裁剪后 Query 集合；
- 裁剪后 Grounding Data 和 Assertion Version；
- Google Workspace ingestion Version；
- User、Email 和 Golden Set Mapping Version；
- Playwright scraping Version；
- 两侧产品和模型版本；
- Outlook Team Bake-off Runner Version；
- 最终评分使用的 SEVAL Job 与 `lm_checklist` Version。

如果 Gmail 页面结构变化导致 scraping Version 更新，先用固定冒烟 Query 验证采集结果，再重新运行对比。不能把自动化脚本变化和产品版本变化放在同一次实验中解释。
