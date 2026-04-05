# AI Agent 学习路线图 —— 总索引

## 概览

本路线图将 AI Agent 学习分为 **4个主要阶段**（Phase 1-8，共8周核心 + 可选专题），从 LLM 基础调用到生产级多智能体系统，逐步递进。

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Agent 学习路线                          │
├─────────────┬───────────────────────────────────────────────┤
│ Phase 1-2   │ LLM 基础 + Prompt 工程 + 安全入门             │
│ (第1-4周)   │ 14个知识点                                    │
├─────────────┼───────────────────────────────────────────────┤
│ Phase 3-4   │ RAG + 上下文工程 + 评估部署                   │
│ (第5-8周)   │ 10个知识点                                    │
├─────────────┼───────────────────────────────────────────────┤
│ Phase 5-8   │ 高级检索 + 多智能体 + 可靠性                  │
│ (第9-16周)  │ 14个知识点                                    │
├─────────────┼───────────────────────────────────────────────┤
│ 可选专题    │ 本地模型 + 微调 + 多模态 + 企业级              │
│ (按需选修)  │ 9个知识点                                     │
└─────────────┴───────────────────────────────────────────────┘
```

---

## Phase 1-2：LLM 基础与 Prompt 工程

> 目标：能独立集成 LLM API，实现工具调用和结构化输出，理解基本安全防护

| 编号 | 文件 | 主题 | 重要程度 |
|------|------|------|---------|
| 1.1 | [python-engineering.md](phase-1-2/python-engineering.md) | Python 工程化基础 | ⭐⭐⭐⭐⭐ |
| 1.2 | [llm-paradigm.md](phase-1-2/llm-paradigm.md) | LLM 范式与原理 | ⭐⭐⭐⭐⭐ |
| 1.3 | [token-tokenization.md](phase-1-2/token-tokenization.md) | Token 与分词 | ⭐⭐⭐⭐ |
| 1.4 | [context-window.md](phase-1-2/context-window.md) | 上下文窗口管理 | ⭐⭐⭐⭐⭐ |
| 1.5 | [generation-params.md](phase-1-2/generation-params.md) | 生成参数调优 | ⭐⭐⭐⭐ |
| 1.6 | [message-roles.md](phase-1-2/message-roles.md) | 消息角色与会话结构 | ⭐⭐⭐⭐⭐ |
| 1.7 | [llm-api-integration.md](phase-1-2/llm-api-integration.md) | LLM API 集成 | ⭐⭐⭐⭐⭐ |
| 1.8 | [prompt-engineering.md](phase-1-2/prompt-engineering.md) | Prompt 工程 | ⭐⭐⭐⭐⭐ |
| 1.9 | [structured-output.md](phase-1-2/structured-output.md) | 结构化输出 | ⭐⭐⭐⭐⭐ |
| 1.10 | [tool-calling.md](phase-1-2/tool-calling.md) | 工具调用 | ⭐⭐⭐⭐⭐ |
| 1.11 | [basic-security.md](phase-1-2/basic-security.md) | 基础安全防护 | ⭐⭐⭐⭐⭐ |
| 1.12 | [observability-intro.md](phase-1-2/observability-intro.md) | 可观测性入门 | ⭐⭐⭐⭐ |
| 1.13 | [transformer-intro.md](phase-1-2/transformer-intro.md) | Transformer 原理 | ⭐⭐⭐ |
| 1.14 | [business-system-integration.md](phase-1-2/business-system-integration.md) | 业务系统集成 | ⭐⭐⭐⭐⭐ |

---

## Phase 3-4：RAG 与上下文工程

> 目标：构建完整 RAG 流水线，实现引用溯源，部署可评估的 AI 服务

| 编号 | 文件 | 主题 | 重要程度 |
|------|------|------|---------|
| 2.1 | [context-engineering.md](phase-3-4/context-engineering.md) | 上下文工程 | ⭐⭐⭐⭐⭐ |
| 2.2 | [embedding-similarity.md](phase-3-4/embedding-similarity.md) | Embedding 与相似度 | ⭐⭐⭐⭐⭐ |
| 2.3 | [chunking-strategies.md](phase-3-4/chunking-strategies.md) | 分块策略 | ⭐⭐⭐⭐ |
| 2.4 | [rag-pipeline.md](phase-3-4/rag-pipeline.md) | RAG 流水线 | ⭐⭐⭐⭐⭐ |
| 2.5 | [vector-db.md](phase-3-4/vector-db.md) | 向量数据库 | ⭐⭐⭐⭐⭐ |
| 2.6 | [citation-traceability.md](phase-3-4/citation-traceability.md) | 引用溯源 | ⭐⭐⭐⭐ |
| 2.7 | [hallucination-rejection.md](phase-3-4/hallucination-rejection.md) | 幻觉检测与拒绝 | ⭐⭐⭐⭐⭐ |
| 2.8 | [business-rules.md](phase-3-4/business-rules.md) | 业务规则引擎 | ⭐⭐⭐⭐ |
| 2.9 | [evaluation-regression.md](phase-3-4/evaluation-regression.md) | 评估与回归测试 | ⭐⭐⭐⭐⭐ |
| 2.10 | [deployment-engineering.md](phase-3-4/deployment-engineering.md) | 部署工程化 | ⭐⭐⭐⭐⭐ |

---

## Phase 5-8：高级能力与生产级系统

> 目标：构建可靠、安全、可扩展的生产级 AI Agent 系统

| 编号 | 文件 | 主题 | 重要程度 |
|------|------|------|---------|
| 3.1 | [advanced-retrieval.md](phase-5-8/advanced-retrieval.md) | 高级检索 | ⭐⭐⭐⭐⭐ |
| 3.2 | [reranking.md](phase-5-8/reranking.md) | 重排序 | ⭐⭐⭐⭐ |
| 3.3 | [workflow-orchestration.md](phase-5-8/workflow-orchestration.md) | 工作流编排 | ⭐⭐⭐⭐⭐ |
| 3.4 | [memory-system.md](phase-5-8/memory-system.md) | 记忆系统 | ⭐⭐⭐⭐⭐ |
| 3.5 | [multi-agent-collaboration.md](phase-5-8/multi-agent-collaboration.md) | 多智能体协作 | ⭐⭐⭐⭐⭐ |
| 3.6 | [reliability-engineering.md](phase-5-8/reliability-engineering.md) | 可靠性工程 | ⭐⭐⭐⭐⭐ |
| 3.7 | [guardrails.md](phase-5-8/guardrails.md) | 护栏系统 | ⭐⭐⭐⭐⭐ |
| 3.8 | [tool-governance.md](phase-5-8/tool-governance.md) | 工具治理 | ⭐⭐⭐⭐ |
| 3.9 | [business-security-compliance.md](phase-5-8/business-security-compliance.md) | 业务安全合规 | ⭐⭐⭐⭐⭐ |
| 3.10 | [data-ingestion-engineering.md](phase-5-8/data-ingestion-engineering.md) | 数据摄取工程 | ⭐⭐⭐⭐ |
| 3.11 | [cost-optimization.md](phase-5-8/cost-optimization.md) | 成本优化 | ⭐⭐⭐⭐ |
| 3.12 | [advanced-observability.md](phase-5-8/advanced-observability.md) | 高级可观测性 | ⭐⭐⭐⭐ |
| 3.13 | [advanced-testing.md](phase-5-8/advanced-testing.md) | 高级测试 | ⭐⭐⭐⭐ |
| 3.14 | [release-strategy.md](phase-5-8/release-strategy.md) | 发布策略 | ⭐⭐⭐⭐ |

---

## 可选专题

> 根据个人需求选修，不影响主线学习

| 编号 | 文件 | 主题 |
|------|------|------|
| O.1 | [local-model-inference.md](optional/local-model-inference.md) | 本地模型推理 |
| O.2 | [fine-tuning.md](optional/fine-tuning.md) | 模型微调 |
| O.3 | [multimodal.md](optional/multimodal.md) | 多模态能力 |
| O.4 | [browser-automation.md](optional/browser-automation.md) | 浏览器自动化 |
| O.5 | [sql-data-analysis-agent.md](optional/sql-data-analysis-agent.md) | SQL 数据分析 Agent |
| O.6 | [code-agent.md](optional/code-agent.md) | 代码生成 Agent |
| O.7 | [enterprise-auth.md](optional/enterprise-auth.md) | 企业级认证授权 |
| O.8 | [sandbox-trusted-execution.md](optional/sandbox-trusted-execution.md) | 沙箱与可信执行 |
| O.9 | [product-metrics-experiments.md](optional/product-metrics-experiments.md) | 产品指标与实验 |

---

## Demo 项目

- **[demo-projects.md](demo-projects.md)**：包含两个完整的业务 Demo
  - 支付失败排查助手：工具调用 + 结构化分析
  - 退款工单助手：RAG + 规则引擎 + 多轮对话

---

## 学习进度追踪

建议在学习过程中使用 GitHub Projects 或简单的 checklist 追踪进度：

```markdown
## 我的学习进度

### Phase 1-2
- [ ] 1.1 Python 工程化基础
- [ ] 1.2 LLM 范式与原理
- [ ] 1.3 Token 与分词
...
```
