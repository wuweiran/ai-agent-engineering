---
layout: default
title: English Interview Introduction
parent: 工作经历
nav_order: 8
permalink: /docs/career/english-self-introduction/
---

# English Interview Introduction

## Self-introduction

Hi, I am a **backend engineer** with **end-to-end experience** in AI Agent development. **Throughout** my career, I have built backend systems for both enterprise customers and individual consumers, so I am familiar with the different business requirements and engineering challenges of both **To-B and To-C products**.

I **began my career** at Meituan, where I worked in Meituan F&B SaaS team and helped build core **merchant-facing** services. That experience gave me a **strong foundation** in complex business **modeling**, system **evolution**, and **performance** optimization for To-B systems. At Alibaba, I was **responsible for** the task scheduling system that supports order fulfillment in Taobao's virtual-items transaction flow, and I also led a big cross-department project. There, I gained experience in large-scale To-C **traffic**, production **reliability**, and **cross-team** delivery.

After **joining** Microsoft, I **applied** my backend experience to enterprise cloud products. In Microsoft Purview, I built the Workflow Expression domain **from the ground up**, owned it end to end, and drove its continued evolution. I also **participated** in integrating LLM capabilities into the product, which **marked the beginning** of my work in LLM applications and AI Agents.

After **moving** to Outlook Copilot, I **started with** a search-based RAG service, where I owned the email retrieval path. My work then **expanded to** Agent capability development and evaluation. I built three core Outlook Copilot **scenarios**, owned the overall **evaluation** of Outlook Copilot, and built an automated **bake-off test** system for comparing Outlook Copilot with competitors like Gmail Gemini. **Separately**, I participated in My Outlook, a self-hosted Agent product, and learned how an **Agent runtime** is deployed and operated.

My **strength** is **connecting** Agent design with production engineering. I can start from a business problem, define the **Context and tool boundaries**, design **reliable task execution and recovery**, and then use **evaluation and production feedback** to improve business metrics. My backend background also helps me handle permissions, state management, retries, idempotency, scalability, and production incidents in Agent systems.

## Meituan F&B SaaS

At Meituan, I worked in the F&B SaaS team and helped build core merchant-facing systems. I translated complex workflows, business rules, and user roles into domain models, evolved existing services as the product grew, and improved performance along real production request paths.

## Alibaba Taobao

At Alibaba, I worked in the Taobao technology team and was responsible for the task scheduling system that supports order fulfillment across the virtual-goods transaction flow. It manages task creation, scheduling, execution, retry, and final state convergence under large-scale To-C traffic. I also led a cross-team project that extended the existing capability from Tmall merchants to Taobao merchants. I coordinated stakeholders across multiple teams, managed dependencies and timelines, and drove the project through delivery.

## Microsoft Purview Workflow

Microsoft Purview is an enterprise data governance product. Its Workflow capability supports approvals, notifications, and automated processing for data governance tasks. Azure Logic Apps provides the underlying workflow scheduling, while our team owns the Purview domain layer, including Workflow Definitions, Actions, and product integration.

I designed and built the Workflow Expression capability from the ground up and served as its domain owner. I defined the expression syntax and implemented parsing, validation, and runtime evaluation. Expressions are used in Workflow Definitions, Actions, and Custom Actions. They can read workflow inputs, reference outputs from earlier steps, and call registered backend functions. This established an extensible framework where new domain functions and customization capabilities can be added quickly, reducing development time by about 70%.

Later, I also participated in adding LLM capabilities for workflow summarization and generation. Backend validation and user confirmation kept the generated output controlled before it could be saved or published.

## Outlook Copilot Insight Service

Copilot needs real-time Outlook evidence to answer email questions, but raw Outlook Search results cannot be placed directly into the model Context. The Insight Service therefore acts as a search-based RAG layer between Outlook Search and Copilot.

My main responsibility was the email query pipeline. A continuously changing mailbox creates several challenges: results may be duplicated across messages and conversations, every candidate must remain within the user's permission boundary, and retrieval must stay within strict latency, memory, and Token budgets.

I used Outlook Search for paginated retrieval, then applied object-level permission filtering, message and conversation deduplication, and deterministic multi-signal ranking. A bounded Top-12 candidate pool, early stopping, and timeout-based degradation kept downstream calls and resource usage predictable. The final results included lightweight snippets and citations for Copilot. The pipeline achieved 93.2% Recall@12, zero permission leaks, 100% Citation Validity, and 1.1-second P95 latency.

## Outlook Copilot Agent Capabilities

In this project, I developed three Outlook Copilot scenarios from scratch: explaining selected text, summarizing attachments, and organizing a mailbox. Mailbox organization was the most challenging because the Agent had to turn an ambiguous request into a confirmed plan and then complete real mailbox changes safely.

The scenarios ran on the managed Microsoft 365 Copilot Agent Runtime, which provided the model loop, conversation state, planning, and confirmation. I built the Agent Definition, scenario instructions, Context configuration, and tool integration on top of this Runtime.

The initial online task completion rate was 62%. Funnel analysis showed two main problems. Some tasks never reached confirmation because the Agent asked too many questions or failed to narrow the email scope. Other tasks failed after confirmation because mailbox state had changed, only part of a batch succeeded, or the result of a write request was unknown.

I split the flow into scope clarification and confirmed execution. The Agent asks only for information that changes the result, searches for candidate emails, reads details when needed, and presents a concrete plan for confirmation. During execution, it refreshes changed items, retries only unfinished operations after partial success, and checks the original operation after an unknown result.

These changes increased the online task completion rate from 62% to 78% and the offline scenario pass rate from 68% to 84%, while serious safety violations remained at zero.

## My Outlook Self-hosted Agent Deployment

My Outlook is a self-hosted personal Agent that uses email, calendar, and organizational signals to generate briefings, catch-up items, drafts, and recommendations. The POS Service owns the Agent loop and task orchestration, while separate AKS Workers execute Deep Scan, Synthesis, LLM, and Graph or API steps.

Because these tasks can run across multiple services and outlive the original HTTP request, each action is persisted as a Task, Run, Step, and Attempt before being dispatched through a queue. The Task Store is the source of truth; leases control which Worker can advance a Step, and idempotency keys protect external side effects.

I helped deploy and maintain this runtime and used end-to-end traces to diagnose real recovery failures. In one case, a Synthesis Worker wrote an Artifact to SDS but restarted before saving the Step Result, so recovery generated the Artifact again. We introduced a stable Artifact ID and idempotency key, checked SDS before retrying, and then repaired the missing Step Result. I also fixed stale Attempts overwriting newer results by validating both the Attempt ID and Lease Version, and kept old Tasks running across rolling releases by pinning their Agent, Context, and Worker capability versions.

Before each model call, the Context Builder reconstructs the Model Input from conversation history, task state, findings, artifacts, and Worker results. It preserves the current goal and confirmed state, selects evidence for the current Step, replaces stale versions, and compresses older history when the Token budget is exceeded. This keeps Context management separate from durable task recovery.

My responsibility focused on maintaining the deployment and execution path. The Deep Scan and Synthesis business algorithms were owned by other teams.

## Copilot Evaluation and Golden Set

I am responsible for the overall evaluation of Outlook Copilot. Evaluation is used throughout Feature development, not only as a release gate. The internal SEVAL platform provides job execution and result management, while the Outlook team owns the email-specific evaluation data, success criteria, and regression analysis.

Initially, each Feature maintained its own Queries and test emails. As coverage grew, the data became duplicated and fragmented, and an email change could break tests without a clear impact scope. We separated and centralized Queries, Grounding Data, and Assertions into a shared Golden Set. Reusable high-value emails support multiple Queries, while stable mappings make affected tests traceable.

Developer-written Utterances also tended to be cleaner and more explicit than real user input. I therefore incorporated privacy-protected, eyes-off user Utterances to represent the real distribution, while retaining low-frequency, high-risk boundary cases.

For each Agent, model, Prompt, Context, or tool change, I run a Baseline and compare the Candidate on the same assets. We evaluate the final result, Tool Calls, citations, safety, and latency. During development, Feature-level slices show whether the change works; after that, the full Golden Set checks cross-feature regressions and supports the release decision.

When a case fails, I follow its Trace to find the first divergence in the Context, model decision, Tool Call, Tool Result, or evaluator. After the owning layer is fixed, I rerun the failed cases, related scenarios, and the full regression set. Production failures are also converted into reusable evaluation cases.

## Copilot Bake-off

Copilot Bake-off compares the end-to-end product behavior of Outlook Copilot and Gmail Gemini rather than comparing their underlying models. The main challenge was fairness: the two products have different capability ranges, user systems, mail stores, and execution interfaces.

I built the system from scratch. It first selects Queries supported by both products, imports the corresponding Outlook test emails into Google Workspace, and maintains User, Email, and Golden Set mappings so that both sides receive equivalent tasks and data. Outlook responses are collected through SEVAL scraping. Gmail has no equivalent evaluation API, so Playwright operates the real Gemini interface and waits for the streamed response to complete.

Only Queries with valid responses from both products are paired and scored against the same Assertions; ingestion, mapping, and scraping failures are reported separately from product failures. The comparison produces Query-level outcomes such as `Outlook fail / Gemini pass`. High-value gaps are added to the regular Outlook Golden Set, fixed through the normal regression process, and checked again in a later Bake-off run.