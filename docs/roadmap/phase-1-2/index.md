# Phase 1-2 索引：LLM 基础与 Prompt 工程

## 阶段目标

完成 Phase 1-2 后，你应该能够：

1. 独立搭建 Python 工程化环境，管理依赖和配置
2. 理解 LLM 的工作原理（Token、上下文窗口、生成参数）
3. 熟练调用主流 LLM API（OpenAI、Azure OpenAI、国内模型）
4. 编写高质量 Prompt，掌握 CoT、Few-shot 等技巧
5. 实现结构化输出（JSON Schema、Pydantic 验证）
6. 完成工具调用（Function Calling）的完整循环
7. 理解基本安全威胁（Prompt Injection）和防护措施
8. 搭建基础可观测性（日志、Token 计费、Trace）
9. 了解 Transformer 的核心机制（注意力机制）
10. 将 LLM 能力集成到 Java/Spring Boot 业务系统

## 知识点列表（14个）

| 编号 | 文件 | 主题 | 预计学习时间 |
|------|------|------|-------------|
| 1.1 | [python-engineering.md](python-engineering.md) | Python 工程化基础 | 4小时 |
| 1.2 | [llm-paradigm.md](llm-paradigm.md) | LLM 范式与原理 | 3小时 |
| 1.3 | [token-tokenization.md](token-tokenization.md) | Token 与分词 | 2小时 |
| 1.4 | [context-window.md](context-window.md) | 上下文窗口管理 | 3小时 |
| 1.5 | [generation-params.md](generation-params.md) | 生成参数调优 | 2小时 |
| 1.6 | [message-roles.md](message-roles.md) | 消息角色与会话结构 | 2小时 |
| 1.7 | [llm-api-integration.md](llm-api-integration.md) | LLM API 集成 | 4小时 |
| 1.8 | [prompt-engineering.md](prompt-engineering.md) | Prompt 工程 | 6小时 |
| 1.9 | [structured-output.md](structured-output.md) | 结构化输出 | 3小时 |
| 1.10 | [tool-calling.md](tool-calling.md) | 工具调用 | 4小时 |
| 1.11 | [basic-security.md](basic-security.md) | 基础安全防护 | 3小时 |
| 1.12 | [observability-intro.md](observability-intro.md) | 可观测性入门 | 2小时 |
| 1.13 | [transformer-intro.md](transformer-intro.md) | Transformer 原理 | 6小时 |
| 1.14 | [business-system-integration.md](business-system-integration.md) | 业务系统集成 | 4小时 |

**总计约 48小时（2周全职学习）**

## 学习路径建议

```
第1周：
  Day 1-2: 1.1 Python工程化 + 1.2 LLM范式
  Day 3:   1.3 Token分词 + 1.4 上下文窗口
  Day 4:   1.5 生成参数 + 1.6 消息角色
  Day 5-6: 1.7 API集成（重点，动手练习）
  Day 7:   复习 + 整合练习

第2周：
  Day 1-2: 1.8 Prompt工程（重点）
  Day 3:   1.9 结构化输出
  Day 4:   1.10 工具调用（重点）
  Day 5:   1.11 安全防护 + 1.12 可观测性
  Day 6:   1.13 Transformer原理（选读）
  Day 7:   1.14 业务集成 + 综合练习
```

## 阶段产出物

学习完成后，你应该有：

1. **一个可运行的 Python 项目**：包含环境配置、依赖管理、基础封装
2. **工具调用 Demo**：至少调用2个工具（如查询订单、查询账户）
3. **结构化输出 Demo**：LLM 返回 Pydantic 验证的 JSON
4. **安全测试报告**：记录你发现的 Prompt Injection 案例

## 前置依赖

```bash
pip install openai>=1.0.0
pip install langchain>=0.2.0
pip install langchain-openai
pip install pydantic>=2.0.0
pip install python-dotenv
pip install tiktoken
pip install loguru
```

## 相关资源

- [OpenAI API 文档](https://platform.openai.com/docs)
- [LangChain 文档](https://python.langchain.com/docs)
- [Pydantic 文档](https://docs.pydantic.dev)
- [Attention Is All You Need 论文](https://arxiv.org/abs/1706.03762)
