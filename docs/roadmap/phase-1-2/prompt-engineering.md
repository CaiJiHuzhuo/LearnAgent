# Prompt 工程

## 目标与重要性

Prompt 工程是提升 LLM 输出质量最直接的方法，不需要修改模型权重。掌握 Prompt 设计能让你在业务场景中获得更准确、更稳定的输出。

## 核心概念清单

- 基础技巧：明确指令、角色扮演、格式控制
- Chain of Thought (CoT)：分步推理
- Few-shot Learning：示例驱动
- Self-consistency：多次采样取最优
- ReAct：推理与行动结合
- Prompt 模板化与版本管理

## 学习路径

### 入门（第1层）

```python
from openai import OpenAI
client = OpenAI()

# 技巧1：明确角色和任务
def analyze_payment_failure_v1(order_info: dict) -> str:
    """版本1：简单直接的 prompt"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "system",
            "content": """你是一个专业的支付系统故障分析专家，有10年支付行业经验。
你的分析需要：准确、简洁、有可操作性。
始终用中文回复。"""
        }, {
            "role": "user",
            "content": f"""分析以下支付失败案例：

订单信息：{order_info}

请提供：
1. 失败根因（1-2句话）
2. 严重程度（低/中/高）
3. 建议措施（2-3条）"""
        }],
        temperature=0.1
    )
    return response.choices[0].message.content

# 技巧2：格式控制
def get_structured_analysis(order_info: dict) -> str:
    """要求 LLM 按照固定格式输出"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"""分析支付失败：{order_info}

请严格按以下格式回复：
---
根因：[简明原因]
类别：[账户问题/渠道问题/风控拦截/系统故障]
严重程度：[低/中/高]
建议1：[第一条建议]
建议2：[第二条建议]
---"""
        }],
        temperature=0.1
    )
    return response.choices[0].message.content
```

### 进阶（第2层）

```python
# Chain of Thought (CoT)：让模型展示推理过程
def analyze_with_cot(order_info: dict) -> str:
    """CoT：先推理再得出结论"""
    cot_prompt = f"""你是支付系统分析专家。请分析以下支付失败案例。

订单信息：
{order_info}

请按照以下步骤分析：

步骤1 - 识别直接失败原因：
（分析错误代码和失败消息的字面含义）

步骤2 - 分析根本原因：
（结合账户状态、渠道状态、时间等因素）

步骤3 - 评估影响范围：
（这是个案还是系统性问题？影响哪些用户？）

步骤4 - 给出建议：
（针对用户的建议 + 针对系统的建议）

最终结论：
（用2-3句话总结根因和最关键的建议）"""

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": cot_prompt}],
        temperature=0.1
    )
    return response.choices[0].message.content

# Few-shot：通过示例教模型
def translate_error_code_few_shot(error_code: str) -> str:
    """Few-shot：通过示例学习翻译格式"""
    few_shot_examples = [
        ("INSUFFICIENT_BALANCE", "账户余额不足，当前余额无法支付此订单。\n建议：请充值后重试，或选择其他支付方式。"),
        ("CARD_EXPIRED", "您的银行卡已过期，无法完成支付。\n建议：请更新银行卡信息或使用其他有效卡片。"),
        ("RISK_CONTROL_REJECTED", "交易被风控系统拦截，可能存在安全风险。\n建议：请联系客服核实身份，或稍后再试。"),
    ]

    examples_text = "\n\n".join([
        f"错误代码：{code}\n用户提示：{desc}"
        for code, desc in few_shot_examples
    ])

    prompt = f"""将支付错误代码转换为对用户友好的提示语。

以下是示例：

{examples_text}

现在请翻译：
错误代码：{error_code}
用户提示："""

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.1,
        stop=["\n\n错误代码："]  # 防止模型继续生成示例
    )
    return response.choices[0].message.content
```

### 深入（第3层）

```python
# Prompt 模板管理系统
from string import Template
from typing import Dict, Any
import json
import yaml

class PromptTemplate:
    def __init__(self, template: str, version: str = "1.0"):
        self.template = template
        self.version = version
        self._validate()

    def _validate(self):
        """验证模板变量"""
        import re
        self.variables = re.findall(r'\{(\w+)\}', self.template)

    def render(self, **kwargs) -> str:
        missing = set(self.variables) - set(kwargs.keys())
        if missing:
            raise ValueError(f"缺少模板变量: {missing}")
        return self.template.format(**kwargs)

class PromptRegistry:
    """Prompt 模板注册表"""

    def __init__(self):
        self._templates: Dict[str, PromptTemplate] = {}

    def register(self, name: str, template: str, version: str = "1.0"):
        self._templates[f"{name}:{version}"] = PromptTemplate(template, version)
        # 同时注册为 latest
        self._templates[f"{name}:latest"] = self._templates[f"{name}:{version}"]

    def get(self, name: str, version: str = "latest") -> PromptTemplate:
        key = f"{name}:{version}"
        if key not in self._templates:
            raise KeyError(f"Prompt 模板不存在: {key}")
        return self._templates[key]

# 使用示例
registry = PromptRegistry()

registry.register(
    "analyze_payment_failure",
    """你是支付系统分析专家。

分析以下支付失败订单：
- 订单ID：{order_id}
- 失败代码：{fail_code}
- 失败原因：{fail_reason}
- 账户余额：{balance}元
- 订单金额：{amount}元

请提供：
1. 根因分析
2. 建议措施
3. 是否需要人工介入（是/否）""",
    version="1.0"
)

# 渲染并使用
template = registry.get("analyze_payment_failure")
prompt = template.render(
    order_id="PAY001",
    fail_code="INSUFFICIENT_BALANCE",
    fail_reason="余额不足",
    balance=50,
    amount=299
)

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": prompt}],
    temperature=0.1
)
print(response.choices[0].message.content)
```

## 最小可执行练习

```python
# A/B 测试不同 prompt 的效果
import time

def ab_test_prompts(prompts: dict, test_cases: list, n_runs: int = 3) -> dict:
    """对比不同 prompt 的输出质量"""
    results = {name: [] for name in prompts}

    for test_case in test_cases:
        for prompt_name, prompt_template in prompts.items():
            outputs = []
            for _ in range(n_runs):
                response = client.chat.completions.create(
                    model="gpt-4o-mini",
                    messages=[{
                        "role": "user",
                        "content": prompt_template.format(**test_case)
                    }],
                    temperature=0.5  # 稍高温度体现差异
                )
                outputs.append(response.choices[0].message.content)

            results[prompt_name].append({
                "test_case": test_case,
                "outputs": outputs,
                "consistent": len(set(outputs)) == 1
            })

    return results

# 测试
prompts = {
    "简单版": "分析支付失败原因：{fail_code}",
    "详细版": "你是支付专家。分析失败代码 {fail_code} 的原因，给出3条建议。",
    "CoT版": "你是支付专家。首先分析 {fail_code} 的字面含义，然后推断根因，最后给建议。",
}

test_cases = [
    {"fail_code": "INSUFFICIENT_BALANCE"},
    {"fail_code": "NETWORK_ERROR"},
]

results = ab_test_prompts(prompts, test_cases)
print(json.dumps(results, ensure_ascii=False, indent=2))
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 输出格式不稳定 | 格式要求不够明确 | 使用结构化输出或提供明确的格式示例 |
| 模型忽略约束 | System prompt 太弱 | 在 system 和 user 都重复关键约束 |
| Few-shot 效果差 | 示例质量低或不够典型 | 精选高质量示例，覆盖边界情况 |
| CoT 推理出错 | 中间步骤有误导致最终错误 | 使用 Self-consistency，多次采样取最优 |

## Java/Spring Boot 对接要点

```java
// 使用 Spring AI 的 PromptTemplate
@Service
public class PaymentAnalysisService {

    private final ChatClient chatClient;

    private static final String ANALYSIS_PROMPT = """
        你是支付系统分析专家。
        
        分析失败订单：
        - 订单ID：{orderId}
        - 失败代码：{failCode}
        
        请提供根因分析和建议措施。
        """;

    public String analyzeFailure(String orderId, String failCode) {
        PromptTemplate template = new PromptTemplate(ANALYSIS_PROMPT);
        Prompt prompt = template.create(Map.of(
            "orderId", orderId,
            "failCode", failCode
        ));
        return chatClient.prompt(prompt).call().content();
    }
}
```

## 参考资料

- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [Chain-of-Thought Prompting 论文](https://arxiv.org/abs/2201.11903)
- [ReAct 论文](https://arxiv.org/abs/2210.03629)
- [大模型提示工程指南（中文）](https://www.promptingguide.ai/zh)
