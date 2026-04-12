# AI Agent 学习路线图 —— 8周系统入门指南

## 项目简介

本项目是一份面向工程师的 **AI Agent 系统学习路线图**，覆盖从基础 LLM 调用到生产级多智能体协作的完整知识体系。内容以实际业务场景（支付、退款、客服等）为驱动，强调工程实践与安全合规。

> 适合人群：有一定 Python/Java 基础，希望系统掌握 AI Agent 开发的后端工程师。

---

## 学习路线总览

| 阶段 | 周次 | 主题 | 关键产出 |
|------|------|------|---------|
| Phase 1 | 第1-2周 | LLM 基础与工程化 | 能独立调用 LLM API，实现结构化输出与工具调用 |
| Phase 2 | 第3-4周 | Prompt 工程与安全 | 掌握提示词设计、基本安全防护、可观测性入门 |
| Phase 3 | 第5-6周 | RAG 与上下文工程 | 构建完整 RAG 流水线，向量检索+引用溯源 |
| Phase 4 | 第7-8周 | 评估、部署与规则引擎 | 上线一个可评估、可回归的 RAG 服务 |
| Phase 5 | 第9-10周 | 高级检索与工作流编排 | 多步骤 Agent 工作流，Reranking |
| Phase 6 | 第11-12周 | 记忆系统与多智能体 | 多 Agent 协作，长期记忆管理 |
| Phase 7 | 第13-14周 | 可靠性与安全合规 | Guardrails、工具治理、合规审计 |
| Phase 8 | 第15-16周 | 成本优化与发布策略 | 灰度发布、成本控制、高级测试 |

---

## 前置条件

- **Python 3.10+**：熟悉类型注解、async/await、虚拟环境管理
- **Java 17+（可选）**：了解 Spring Boot，能读懂 Java 示例
- **Git**：基本版本控制操作
- **Docker**：能运行容器化服务（向量数据库等）
- **基础数学**：线性代数基础（向量点积、矩阵乘法）

---

## 目录结构

```
docs/
└── roadmap/
    ├── README.md                    # 总索引
    ├── demo-projects.md             # 两个完整 Demo 项目
    ├── phase-1-2/                   # Phase 1-2：14个知识点
    │   ├── index.md
    │   ├── python-engineering.md
    │   ├── llm-paradigm.md
    │   ├── token-tokenization.md
    │   ├── context-window.md
    │   ├── generation-params.md
    │   ├── message-roles.md
    │   ├── llm-api-integration.md
    │   ├── prompt-engineering.md
    │   ├── structured-output.md
    │   ├── tool-calling.md
    │   ├── basic-security.md
    │   ├── observability-intro.md
    │   ├── transformer-intro.md
    │   └── business-system-integration.md
    ├── phase-3-4/                   # Phase 3-4：10个知识点
    ├── phase-5-8/                   # Phase 5-8：14个知识点
    └── optional/                    # 可选专题：9个知识点
```

---

## 快速开始

### 环境搭建

```bash
# 克隆项目
git clone https://github.com/your-org/LearnAgent.git
cd LearnAgent

# 创建虚拟环境
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# 安装依赖
pip install openai langchain langchain-openai langchain-community
pip install chromadb sentence-transformers tiktoken
pip install fastapi uvicorn python-dotenv pydantic

# 配置 API Key
cp .env.example .env
# 编辑 .env 填入 OPENAI_API_KEY
```

### 第一个 Hello World

```python
from openai import OpenAI

client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "你是一个专业的支付系统客服助手。"},
        {"role": "user", "content": "我的支付失败了，订单号 PAY202401001，请帮我排查。"}
    ],
    temperature=0.1,
    max_tokens=500
)

print(response.choices[0].message.content)
print(f"消耗 Token：{response.usage.total_tokens}")
```

---

## 核心理念

### 1. 业务驱动学习
每个知识点都与实际业务场景绑定：
- 支付失败排查 → 工具调用、结构化输出
- 退款工单处理 → RAG、规则引擎
- 客服问答 → 多轮对话、记忆系统

### 2. 安全第一
- 所有写操作需人工审核
- 敏感数据脱敏后才能进入 LLM
- 完整的审计日志

### 3. 可观测性贯穿始终
- 每个 LLM 调用都要有 trace_id
- 记录 token 消耗和延迟
- 支持回放和调试

---

## 学习建议

1. **按顺序学习**：Phase 1-2 是基础，不建议跳过
2. **动手实践**：每个知识点都有最小可执行练习，务必完成
3. **结合业务**：将练习场景替换为你实际工作中的系统
4. **记录问题**：遇到的坑记录在 issues 里，方便后续查阅

---

## Demo 项目

- **[支付失败排查助手](docs/roadmap/demo-projects.md#项目一支付失败排查助手)**：调用支付系统 API，分析失败原因，生成排查报告
- **[退款工单助手](docs/roadmap/demo-projects.md#项目二退款工单助手)**：RAG + 规则引擎，自动处理退款申请

---

## 贡献指南

欢迎提交 PR 改进文档或补充代码示例。请确保：
- 代码示例经过实际测试
- 中文内容准确清晰
- 遵循现有文件结构

---

## License

MIT License - 详见 [LICENSE](LICENSE)

---

*本路线图持续更新，建议 Watch 本仓库获取最新内容。*
