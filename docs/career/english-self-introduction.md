---
layout: default
title: English Interview Introduction
parent: 工作经历
nav_order: 8
permalink: /docs/career/english-self-introduction/
---

# English Interview Introduction

## Self-introduction

Hi, I am a **backend engineer** with **end-to-end experience in AI Agent development**. Throughout my career, I have built backend systems for both enterprise customers and individual consumers, so I am familiar with the different business requirements and engineering challenges of both **To-B and To-C products**.

I began my career at **Meituan**, where I worked in Meituan F&B SaaS team and helped build core merchant-facing services. That experience gave me a strong foundation in **complex business modeling, system evolution, and performance optimization** for To-B systems. At **Alibaba**, I was responsible for the task scheduling system that supported order fulfillment across Taobao's virtual-goods transaction flow, and I also led a cross-department project. There, I gained experience in **large-scale To-C traffic, production reliability, and cross-team delivery**.

After joining **Microsoft**, I applied this backend experience to enterprise cloud products. In **Microsoft Purview**, I designed and built the **Workflow Expression** area from the ground up and continued to own its evolution. I was responsible for the complete lifecycle, including syntax design, parsing, validation, runtime evaluation, and integration with the workflow system. I also participated in introducing LLM capabilities into the product, which marked the beginning of my work in **LLM applications and AI Agents**.

After moving to **Outlook Copilot**, I started with a **search-based RAG service**, where I owned the email retrieval path. My work then expanded to **production Agent development and evaluation**. I built three core Outlook Copilot scenarios, took responsibility for the overall evaluation of Outlook Copilot, and built an automated **bake-off system** for comparing Outlook Copilot with Gmail Gemini. Separately, I participated in **My Outlook**, a self-hosted Agent product, and learned how a **production-grade Agent** is deployed and operated.

My strength is connecting **Agent design with production engineering**. I can start from a business problem, define the **Context and tool boundaries**, design **reliable task execution and recovery**, and then use **evaluation and production feedback** to improve business metrics such as task completion rate. My backend background also helps me handle permissions, state management, retries, idempotency, scalability, and production incidents in Agent systems.

## Meituan F&B SaaS

At Meituan, I worked in the F&B SaaS team and helped build core merchant-facing systems. I translated complex workflows, business rules, and user roles into domain models, evolved existing services as the product grew, and improved performance along real production request paths.

## Alibaba Taobao

At Alibaba, I worked in the Taobao technology team and was responsible for the task scheduling system that supports order fulfillment across the virtual-goods transaction flow. It manages task creation, scheduling, execution, retry, and final state convergence under large-scale To-C traffic. I also led a cross-team project that extended the existing capability from Tmall merchants to Taobao merchants. I coordinated stakeholders across multiple teams, managed dependencies and timelines, and drove the project through delivery.

## Microsoft Purview Workflow

Microsoft Purview is an enterprise data governance product. Its Workflow capability supports approvals, notifications, and automated processing for data governance tasks. Azure Logic Apps provides the underlying workflow scheduling, while our team owns the Purview domain layer, including Workflow Definitions, Actions, and product integration.

I designed and built the Workflow Expression capability from the ground up and served as its domain owner. I defined the expression syntax and implemented parsing, validation, and runtime evaluation. Expressions are used in Workflow Definitions, Actions, and Custom Actions. They can read workflow inputs, reference outputs from earlier steps, and call registered backend functions. This established an extensible framework where new domain functions and customization capabilities can be added quickly, reducing development time by about 70%.

Later, I also participated in adding LLM capabilities for workflow summarization and generation. Backend validation and user confirmation kept the generated output controlled before it could be saved or published.

## Outlook Copilot Insight Service

The Outlook Copilot Insight Service sits between Outlook business data and Copilot. It retrieves the email, conversation, calendar, and people evidence needed for the current request. Its search-based RAG architecture uses Outlook Search for retrieval.

My main responsibility was the email query pipeline. I implemented paginated retrieval, object-level permission filtering, message and conversation deduplication, multi-signal ranking, and Top-K Context selection. The pipeline also preserves citations so that Copilot can ground its answer in the source emails.

The main challenge was to retrieve enough relevant and authorized evidence from a continuously changing mailbox while keeping latency, memory usage, downstream calls, and Token consumption within strict limits. I used a bounded candidate pool, early stopping during pagination, timeout-based degradation, and overload protection to keep the request cost predictable.

## Outlook Copilot Agent Capabilities

In this project, I developed three Outlook Copilot scenarios from scratch: explaining selected text, summarizing attachments, and organizing a mailbox. Together, they covered three important Agent capabilities: understanding the current UI Context, retrieving additional evidence through tools, and executing actions in the user's mailbox.

The Microsoft 365 Copilot platform provides the Agent Runtime, conversation management, planning, and confirmation mechanisms. On top of that platform, I designed the Agent Definition, scenario-specific Instructions, Sydney Context configuration, and Extension integration. Instructions define how the model should make decisions, Context configuration controls the UI and business data available in each scenario, and Plugin Manifests define the tools and their parameter contracts. I also used on-demand tool calls to avoid putting unnecessary email or attachment content into every model request.

Mailbox organization was the most complex scenario. The Agent clarifies an ambiguous goal, searches for candidate emails, reads details when needed, produces an action plan, obtains user confirmation, and then executes write operations. The model decides how to investigate and organize the task, while permissions, confirmation, resource versions, and the final email state remain under deterministic control.

I designed the execution flow to handle version conflicts, partial success, and unknown results. When mailbox state changes after planning, the Agent refreshes the affected item before execution. When only part of a batch succeeds, it retries the remaining items. When a request times out with an unknown result, it queries the original operation to avoid a duplicate write. These controls make the task confirmable and recoverable.

We defined task completion using the final email state. I used the online task funnel to find losses before and after user confirmation, then improved goal clarification, plan confirmation, and execution recovery. The online task completion rate increased from 62% to 78%, and the offline scenario pass rate increased from 68% to 84%, while serious safety violations remained at zero.

## My Outlook Self-hosted Agent Deployment

My Outlook is a personal Agent that uses signals from email, calendars, and organizational data to generate briefings, catch-up items, drafts, and recommendations.

My Outlook has a self-hosted Agent backend. The online Agent service hosts the Agent loop and task orchestration, while separate AKS Workers handle Deep Scan, Synthesis, LLM calls, and Graph or API operations.

I participated in the deployment and runtime engineering of this system. My work focused on the Service-to-Worker task contract, queue-based scheduling, Context assembly and trimming, task recovery, AKS scaling, and production monitoring. The architecture separates low-latency Agent decisions from long-running or resource-intensive work so that each part can scale and recover independently.

Long-running work is persisted as tasks and dispatched to Workers through queues. A Task, Run, Step, and Attempt represent different levels of execution state. Leases control which Worker can advance a task, while stable business identities and idempotency keys prevent repeated delivery from creating duplicate side effects. After a Worker restart, execution resumes from the persisted task state.

Before each model call, the Context Builder reconstructs the model input from conversation history, task state, findings, artifacts, and Worker results. It always preserves the current goal and confirmed state, selects other information according to the current task stage, removes results that have already been consumed, and compresses older history when the input exceeds the Token budget.

My responsibility focused on the deployment and execution path. The Deep Scan and Synthesis business algorithms were owned by other teams.

## Copilot Evaluation and Golden Set

I am responsible for the overall evaluation of Outlook Copilot. The internal SEVAL platform provides job execution and result management, while the Outlook team owns the email-specific evaluation data, success criteria, regression analysis, and release decisions.

I maintain the Golden Set Queries, Grounding Data, and Assertions. Each Query combines the current email context with a user utterance. Grounding Data contains the controlled emails and attachments used as evidence, while Assertions define the facts that a response must cover and the content or behavior that is not allowed.

I also incorporate privacy-protected, eyes-off user utterances to represent real user behavior in the evaluation set. I prioritize reusable emails that can support multiple Queries and retain low-frequency, high-risk cases. This improves both the representativeness and maintainability of the evaluation assets.

For each Agent, model, Prompt, Context, or tool change, I compare the Baseline and Candidate on the same evaluation assets. We evaluate the final result, Tool Calls, citations, safety, latency, and cost, and use both absolute thresholds and relative regression gates to prevent quality loss. Severe safety or incorrect-action failures can block a release even when the average score improves.

When a case fails, I follow its Trace to identify the first point where execution diverges from the expected path. The root cause may be the Context, model decision, Tool Call, Tool Result, or evaluator itself. After the owning layer is fixed, I rerun the failed cases, related scenarios, and the full regression set. Production failures are also converted into reusable evaluation cases, forming a continuous quality improvement loop.

## Copilot Bake-off

Copilot Bake-off is a competitive evaluation system within the Outlook Copilot evaluation work. It compares the end-to-end product behavior of Outlook Copilot and Gmail Gemini using the same supported email tasks, Grounding Data, and Assertions.

I built this system from scratch. To compare the two products end to end, the system uses Playwright to operate the Gmail Gemini interface and collect its responses. It selects tasks supported by both products, imports the corresponding Outlook test data into Google Workspace, maintains cross-system mappings, runs both products, pairs valid responses, and submits them for the same checklist-based scoring.

The comparison identified cases where Outlook failed while Gemini passed. High-value gaps were added to the regular Outlook evaluation set, assigned to the relevant owner, fixed, and verified through regression. A later Bake-off run then confirmed whether the product gap had actually been closed.