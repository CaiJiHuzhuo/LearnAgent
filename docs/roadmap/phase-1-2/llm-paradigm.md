# LLM 范式与原理

## 目标与重要性

理解 LLM 的工作范式是有效使用和调试 AI Agent 的前提。很多"玄学"问题（为什么有时候输出正确有时候不对、为什么 temperature 影响结果）都能从原理层面得到解释。

## 核心概念清单

- 语言模型的核心任务：下一个 Token 预测
- 预训练（Pre-training）vs 微调（Fine-tuning）vs RLHF
- 涌现能力（Emergent Abilities）
- 指令跟随（Instruction Following）
- 上下文学习（In-context Learning）
- 幻觉（Hallucination）的来源
- 模型能力边界

## 学习路径

### 入门（第1层）

LLM 是一个**条件概率模型**，给定前缀文本，预测下一个 token 的概率分布：

```
P(token_n | token_1, token_2, ..., token_{n-1})
```

生成文本时，模型不断采样下一个 token，直到生成终止符。

```python
# 理解 token 预测的直觉
# LLM 看到: "支付失败原因是"
# 它会预测下一个 token 的概率：
# "余额" -> 0.25
# "网络" -> 0.20
# "风控" -> 0.15
# "系统" -> 0.12
# ...

# temperature=1.0: 按原始概率采样
# temperature=0.1: 更确定，高概率 token 更可能被选中
# temperature=2.0: 更随机，分布更平

from openai import OpenAI
client = OpenAI()

# 同样的 prompt，不同 temperature
for temp in [0.0, 0.5, 1.0, 1.5]:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "用一句话描述春天"}],
        temperature=temp,
        max_tokens=50
    )
    print(f"temperature={temp}: {response.choices[0].message.content}")
```

### 进阶（第2层）

理解模型训练阶段：

```
阶段1：预训练 (Pre-training)
  数据：海量互联网文本（数万亿 token）
  目标：学习语言知识、世界知识
  结果：Base Model（会续写文本，但不懂指令）

阶段2：指令微调 (SFT - Supervised Fine-tuning)
  数据：人工标注的问答对（指令+期望输出）
  目标：学会理解和执行指令
  结果：Instruct Model（能回答问题，但可能不安全）

阶段3：RLHF (Reinforcement Learning from Human Feedback)
  数据：人类对多个输出的偏好排序
  目标：让输出更符合人类偏好（有帮助、无害、诚实）
  结果：Chat Model（GPT-4、Claude等）
```

```python
# 演示上下文学习（In-context Learning）
# 不需要微调，通过示例教模型做新任务

few_shot_prompt = """将以下支付错误代码翻译为用户友好的提示语。

错误代码: INSUFFICIENT_BALANCE
用户提示: 您的账户余额不足，请充值后重试。

错误代码: CARD_EXPIRED  
用户提示: 您的银行卡已过期，请更换有效的支付方式。

错误代码: RISK_CONTROL_REJECTED
用户提示: """

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": few_shot_prompt}],
    temperature=0.1
)
# 模型会根据示例的模式，生成对应的用户友好提示
print(response.choices[0].message.content)
```

### 深入（第3层）

理解幻觉的根源和缓解方法：

```python
# 幻觉的3个主要来源：
# 1. 训练数据中的错误信息
# 2. 模型过度自信（即使知识不足也会生成文本）
# 3. 提示词导致的偏向性输出

# 缓解幻觉的方法：
def query_with_uncertainty(question: str, context: str) -> str:
    """要求模型表达不确定性"""
    prompt = f"""基于以下上下文回答问题。
如果上下文中没有相关信息，请明确说"根据现有信息无法确定"，
不要猜测或编造答案。

上下文：
{context}

问题：{question}

请用以下格式回答：
- 确定信息：[基于上下文可以确定的内容]
- 不确定信息：[需要额外验证的内容]
- 无法回答：[完全没有信息的部分]"""

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.1
    )
    return response.choices[0].message.content

# 使用 RAG 提供事实依据
def query_with_rag(question: str, retrieved_docs: list) -> str:
    context = "\n\n".join([
        f"[来源{i+1}] {doc['content']}"
        for i, doc in enumerate(retrieved_docs)
    ])
    return query_with_uncertainty(question, context)
```

## 最小可执行练习

```python
# 实验：不同 prompt 对幻觉的影响
import json

def compare_prompts(question: str):
    """对比不同 prompt 策略下的输出质量"""

    # 策略1：直接问（容易产生幻觉）
    response1 = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": question}],
        temperature=0
    )

    # 策略2：要求引用来源
    response2 = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"{question}\n\n请注明你的信息来源，如果不确定请说明。"
        }],
        temperature=0
    )

    # 策略3：限制范围（Chain of Thought）
    response3 = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"{question}\n\n请一步步分析，先列出你确定知道的事实，再给出结论。"
        }],
        temperature=0
    )

    return {
        "direct": response1.choices[0].message.content,
        "with_source": response2.choices[0].message.content,
        "cot": response3.choices[0].message.content
    }

# 测试问题
result = compare_prompts("支付宝2024年的日活用户是多少？")
print(json.dumps(result, ensure_ascii=False, indent=2))
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 模型不执行指令 | 指令不够清晰或 system prompt 太弱 | 使用更明确的指令，加强 system prompt |
| 结果不一致 | temperature 过高 | 降低 temperature 至 0.1 以下 |
| 模型编造数字/日期 | 训练数据截止，缺乏实时信息 | 通过工具调用获取实时数据 |
| 输出格式混乱 | 没有指定格式要求 | 使用结构化输出或在 prompt 中明确格式 |
| 上下文丢失 | 对话轮数太多 | 实现上下文压缩或滑动窗口 |

## Java/Spring Boot 对接要点

```java
// 理解 LLM 的异步特性，在 Spring 中正确处理
@Service
public class LlmService {

    private final OpenAiApi openAiApi;

    // 流式输出，适用于实时展示场景
    public Flux<String> streamCompletion(String prompt) {
        var request = ChatCompletionRequest.builder()
            .model("gpt-4o-mini")
            .messages(List.of(new ChatMessage("user", prompt)))
            .temperature(0.1)
            .stream(true)
            .build();

        return openAiApi.streamChatCompletion(request)
            .map(response -> response.getChoices().get(0).getDelta().getContent())
            .filter(content -> content != null);
    }

    // 非流式，适用于需要完整结果的场景
    public String completion(String prompt) {
        var request = ChatCompletionRequest.builder()
            .model("gpt-4o-mini")
            .messages(List.of(new ChatMessage("user", prompt)))
            .temperature(0.1)
            .build();

        return openAiApi.chatCompletion(request)
            .getChoices().get(0).getMessage().getContent();
    }
}
```

## 参考资料

- [Language Models are Few-Shot Learners (GPT-3)](https://arxiv.org/abs/2005.14165)
- [InstructGPT: Training language models to follow instructions](https://arxiv.org/abs/2203.02155)
- [Survey on Hallucination in LLMs](https://arxiv.org/abs/2309.01219)

---

## 补充：LLM 能力边界与工程启示

### 1. 训练数据截止的影响

```python
# 处理知识截止问题的最佳实践
from openai import OpenAI
from datetime import datetime

client = OpenAI()

def query_with_date_awareness(question: str) -> str:
    """让 LLM 意识到自己的知识截止"""
    current_date = datetime.now().strftime("%Y年%m月%d日")

    system_prompt = f"""你的训练数据截止到2024年初。
当前日期：{current_date}

对于需要实时数据的问题（价格、汇率、股价、新闻等），
请明确说明你的信息可能已过时，建议用户查阅最新来源。"""

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": question}
        ],
        temperature=0.1
    )
    return response.choices[0].message.content

# 对需要实时信息的问题，使用工具调用获取最新数据
def query_with_realtime_data(question: str, realtime_context: dict) -> str:
    """结合实时数据回答问题"""
    augmented_question = f"""
当前实时数据：
{realtime_context}

基于以上实时数据，回答：{question}"""

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": augmented_question}],
        temperature=0.0
    )
    return response.choices[0].message.content

# 示例：支付成功率查询
realtime_data = {
    "alipay_success_rate_1h": 0.998,
    "wechat_success_rate_1h": 0.995,
    "query_time": datetime.now().isoformat()
}
result = query_with_realtime_data(
    "哪个支付渠道目前更稳定？",
    realtime_data
)
print(result)
```

### 2. LLM 的推理局限

```python
# LLM 不擅长的任务类型
LLMS_LIMITATIONS = {
    "精确计算": "LLM 做算数不可靠，应使用 Python 计算工具",
    "实时数据": "LLM 没有实时信息，需要工具调用",
    "长期记忆": "每次对话独立，需要外部存储",
    "确定性执行": "输出有随机性，关键操作需要人工确认",
    "复杂推理链": "多步推理容易出错，使用 CoT 或分步处理",
}

# 处理计算任务：使用工具而非 LLM 直接计算
def calculate_refund_amount(order_amount: float, refund_rate: float) -> float:
    """精确计算退款金额（不依赖 LLM）"""
    return round(order_amount * refund_rate, 2)

def analyze_refund_with_tools(
    order_amount: float,
    refund_rate: float,
    reason: str
) -> str:
    """用工具做计算，用 LLM 做分析"""
    # 精确计算（不用 LLM）
    refund_amount = calculate_refund_amount(order_amount, refund_rate)

    # LLM 做分析（语言理解，不涉及精确计算）
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"""分析以下退款情况：
订单金额：{order_amount}元
退款比例：{refund_rate:.0%}
退款金额：{refund_amount}元（已精确计算）
退款原因：{reason}

请给出退款处理建议（不要重新计算金额）。"""
        }],
        temperature=0.1
    )
    return f"退款金额：{refund_amount}元\n\n{response.choices[0].message.content}"

result = analyze_refund_with_tools(299.00, 1.0, "商品质量问题")
print(result)
```

### 3. 模型选择决策树

```
需要处理的任务
    │
    ├─ 简单问答/分类 → gpt-4o-mini（快速、便宜）
    │
    ├─ 复杂推理/代码生成 → gpt-4o（准确、全面）
    │
    ├─ 长文档处理（>8K tokens）→ gpt-4o（128K 上下文）
    │
    ├─ 实时/低延迟（<1秒）→ gpt-4o-mini
    │
    ├─ 数据安全/不能出境 → 本地模型（Llama、Qwen）
    │
    └─ 中文专业领域 → 通义千问、文心一言
```

```python
# 自动模型选择
from pydantic import BaseModel
from typing import Literal

class ModelSelector:
    def select(
        self,
        task_complexity: Literal["simple", "medium", "complex"],
        requires_real_time: bool = False,
        data_sensitive: bool = False,
        max_cost_per_call_cny: float = 0.5,
        context_tokens: int = 0
    ) -> str:
        if data_sensitive:
            return "local-qwen"  # 数据不出境

        if context_tokens > 50000:
            return "gpt-4o"  # 长上下文

        if task_complexity == "simple" and max_cost_per_call_cny < 0.1:
            return "gpt-4o-mini"

        if task_complexity == "complex":
            return "gpt-4o"

        return "gpt-4o-mini"

selector = ModelSelector()
print(selector.select("complex", context_tokens=1000))  # gpt-4o
print(selector.select("simple", max_cost_per_call_cny=0.05))  # gpt-4o-mini
print(selector.select("medium", data_sensitive=True))  # local-qwen
```
