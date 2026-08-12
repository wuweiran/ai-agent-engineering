---
layout: default
title: 自我介绍与经历重点
parent: 工作经历
nav_order: 0
permalink: /docs/career/self-introduction/
---

# 面试自我介绍

我一直从事后端开发工作。在美团餐饮 SaaS 技术部，我参与商家侧核心系统建设，积累了复杂 B 端业务建模、系统演进和性能治理经验。在阿里巴巴淘系技术部，我负责核心订单任务调度系统，并主导过跨团队项目，积累了大规模 C 端系统建设和协作交付经验。加入微软后，我将这些经验用于 Azure 企业级产品，主导 Purview Workflow Expression 等重要领域建设。转向 AI Agent 后，我先后负责搜索型 RAG、生产级 Agent、Agent 评测体系和竞品自动化评测系统，具备从业务场景设计、Context 与工具编排、任务执行与容错到评测上线的端到端工程经验，能够独立承担复杂 Agent 项目，推动方案从业务需求走到生产落地，并通过评测和线上反馈持续提升任务完成率等业务指标。

## 各阶段关注重点

### 美团餐饮 SaaS

- 理解商家经营流程，将复杂规则和角色关系落到清晰的领域模型；
- 通过系统重构拆分职责和依赖，使核心能力能够持续扩展；
- 结合真实业务链路进行性能治理，在交付新需求的同时控制系统复杂度。

### 阿里巴巴淘系

- 建设订单任务调度系统，处理任务创建、调度、执行、重试和状态收敛；
- 面向大规模 C 端流量关注容量、性能、故障恢复和生产稳定性；
- 主导跨团队项目，明确上下游接口、状态边界和交付节奏，推动完整链路上线。

### Purview Workflow

- 作为 Expression 领域 Owner，从业务定制需求中抽象可复用的表达式能力；
- 负责领域设计、前后端解析、验证、运行时求值及后续演进；
- 将 Expression 接入 Workflow Definition、Action 和 Custom Action，缩短定制需求的交付周期；
- 在已有工作流中接入 LLM，同时用后端校验和用户确认守住确定性边界。

### Outlook 搜索型 RAG

- 将自然语言 Query 转换为受权限控制的邮件检索链路；
- 处理分页召回、对象级权限过滤、去重、排序和 Top-K Context 选择；
- 在召回质量、Citation、Token、延迟和下游调用量之间做取舍；
- 通过有界资源、超时降级、限流和生产监控保障服务稳定运行。

### Outlook Copilot Agent

- 从用户任务出发设计划词解释、附件总结和邮箱整理三个场景；
- 通过 Agent Definition、Instructions、Context Policy 和 Plugin Schema 组织模型行为与工具边界；
- 动态装配和裁剪 Context，管理多轮状态、计划确认和 Token Budget；
- 处理版本冲突、部分成功和结果未知，使写操作可确认、可恢复且不会重复执行；
- 通过线上任务漏斗定位损失并持续提升邮箱整理任务完成率。

### Agent 评测与竞品对比

- 维护 Golden Set、Grounding Data 和 Assertion，定义 Agent 的任务成功与安全标准；
- 评估最终结果、Tool Call、Citation、安全、延迟和成本，并建立质量门禁；
- 通过 Baseline/Candidate 配对和 Trace 找到执行路径中的第一次偏离；
- 建设竞品自动化评测链路，在相同任务和证据下比较 Outlook Copilot 与 Gmail Gemini；
- 将线上失败和竞品差距加入长期回归，推动功能改进并验证问题是否真正收敛。
