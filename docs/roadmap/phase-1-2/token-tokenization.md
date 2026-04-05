# Token 与分词

## 目标与重要性

Token 是 LLM 的基本计算单位，直接决定了 API 成本、上下文窗口利用率和输出质量。理解 Token 机制能帮助你：
- 准确预估 API 成本
- 优化 prompt 长度
- 理解为什么中文更"贵"
- 设计合理的分块策略

## 核心概念清单

- Tokenization 原理（BPE 算法）
- 不同语言的 token 效率
- token 计数方法（tiktoken）
- token 与成本的关系
- context window 中的 token 分布

## 学习路径

### 入门（第1层）

```python
import tiktoken

# GPT-4 使用 cl100k_base 编码
enc = tiktoken.get_encoding("cl100k_base")

# 比较中英文 token 效率
texts = {
    "英文": "The payment failed due to insufficient balance.",
    "中文": "支付失败，原因是账户余额不足。",
    "数字": "订单号 PAY202401001，金额 299.00 元",
    "代码": "def get_order(order_id: str) -> dict:",
}

for label, text in texts.items():
    tokens = enc.encode(text)
    ratio = len(text) / len(tokens)
    print(f"{label}: {len(text)} 字符 → {len(tokens)} tokens (比率: {ratio:.1f})")

# 典型输出：
# 英文: 47 字符 → 9 tokens (比率: 5.2)   ← 英文效率高
# 中文: 15 字符 → 14 tokens (比率: 1.1)  ← 中文每字约1个token
# 数字: 24 字符 → 16 tokens (比率: 1.5)
# 代码: 37 字符 → 12 tokens (比率: 3.1)
```

### 进阶（第2层）

```python
# 精确计算 API 调用的 token 成本
def estimate_cost(
    prompt: str,
    expected_output_tokens: int,
    model: str = "gpt-4o"
) -> dict:
    """估算 API 调用成本"""
    PRICES = {
        "gpt-4o": {"input": 0.005, "output": 0.015},      # $/1K tokens
        "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
        "gpt-4-turbo": {"input": 0.01, "output": 0.03},
    }

    enc = tiktoken.encoding_for_model(model)
    input_tokens = len(enc.encode(prompt))

    if model not in PRICES:
        raise ValueError(f"Unknown model: {model}")

    price = PRICES[model]
    input_cost = input_tokens * price["input"] / 1000
    output_cost = expected_output_tokens * price["output"] / 1000
    total_cost = input_cost + output_cost

    return {
        "model": model,
        "input_tokens": input_tokens,
        "output_tokens": expected_output_tokens,
        "total_tokens": input_tokens + expected_output_tokens,
        "input_cost_usd": input_cost,
        "output_cost_usd": output_cost,
        "total_cost_usd": total_cost,
        "total_cost_cny": total_cost * 7.2  # 汇率估算
    }

# 实际使用
prompt = """你是支付系统分析助手。
分析以下订单的支付失败原因：
订单ID: PAY202401001
失败代码: INSUFFICIENT_BALANCE
账户余额: 50元
订单金额: 299元"""

cost = estimate_cost(prompt, expected_output_tokens=200, model="gpt-4o")
print(f"预计成本：¥{cost['total_cost_cny']:.4f}")
print(f"输入 Token：{cost['input_tokens']}")
```

### 深入（第3层）

```python
# BPE (Byte Pair Encoding) 算法简介
# tiktoken 使用 BPE 将文本切分为 token

# 理解 token 边界对提示词的影响
def visualize_tokens(text: str, model: str = "gpt-4o") -> None:
    """可视化 token 边界"""
    enc = tiktoken.encoding_for_model(model)
    tokens = enc.encode(text)

    print(f"原始文本：{text}")
    print(f"Token 数量：{len(tokens)}")
    print("Token 分解：")
    for token in tokens:
        decoded = enc.decode([token])
        print(f"  [{token}] → '{decoded}'")

# 特殊 token
SPECIAL_TOKENS = {
    "<|endoftext|>": 100257,  # 文本结束
    "<|fim_prefix|>": 100258, # 代码补全前缀
    "<|im_start|>": 100264,   # 消息开始
    "<|im_end|>": 100265,     # 消息结束
}

# 这些特殊 token 在计算上下文时会占用位置
def count_message_tokens(messages: list, model: str = "gpt-4o-mini") -> int:
    """精确计算消息列表的 token 数"""
    enc = tiktoken.encoding_for_model(model)
    total = 0

    for message in messages:
        total += 4  # 每条消息的固定开销：<|im_start|> role \n content <|im_end|>
        for key, value in message.items():
            total += len(enc.encode(str(value)))

    total += 2  # 回复前缀的固定开销

    return total

# 测试
messages = [
    {"role": "system", "content": "你是支付系统分析助手。"},
    {"role": "user", "content": "请分析订单 PAY202401001 的失败原因"}
]
print(f"消息 Token 数：{count_message_tokens(messages)}")
```

## 最小可执行练习

```python
# 练习：监控每次 LLM 调用的 token 消耗
from openai import OpenAI
import time

client = OpenAI()

class TokenTracker:
    def __init__(self):
        self.calls = []
        self.total_input_tokens = 0
        self.total_output_tokens = 0

    def track(self, response) -> None:
        usage = response.usage
        self.calls.append({
            "timestamp": time.time(),
            "input_tokens": usage.prompt_tokens,
            "output_tokens": usage.completion_tokens,
            "total_tokens": usage.total_tokens
        })
        self.total_input_tokens += usage.prompt_tokens
        self.total_output_tokens += usage.completion_tokens

    def report(self) -> dict:
        total_calls = len(self.calls)
        return {
            "total_calls": total_calls,
            "total_input_tokens": self.total_input_tokens,
            "total_output_tokens": self.total_output_tokens,
            "avg_input_per_call": self.total_input_tokens / max(total_calls, 1),
            "estimated_cost_cny": (
                self.total_input_tokens * 0.00015 / 1000 +
                self.total_output_tokens * 0.0006 / 1000
            ) * 7.2
        }

tracker = TokenTracker()

for i in range(3):
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": f"解释支付失败代码 ERR_{i:03d}"}],
        max_tokens=100
    )
    tracker.track(response)

import json
print(json.dumps(tracker.report(), indent=2, ensure_ascii=False))
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 实际 token 数与预估不符 | 不同模型使用不同编码 | 使用 `tiktoken.encoding_for_model()` |
| 中文处理效率低 | 中文每字约1 token，远低于英文 | 考虑压缩中文提示词 |
| 超出上下文限制 | 未预估 token 数就发送 | 发送前检查 token 数 |
| Token 计数包含特殊 token | 忽略了消息格式的固定开销 | 使用上面的 `count_message_tokens` |

## Java/Spring Boot 对接要点

```java
// Java 中没有 tiktoken，可以用近似估算
// 或调用 tiktoken 微服务
@Service
public class TokenEstimatorService {

    // 粗略估算：英文约 0.75 token/字符，中文约 1 token/字符
    public int estimateTokens(String text) {
        int chineseCount = 0;
        int otherCount = 0;
        for (char c : text.toCharArray()) {
            if (c >= '\u4e00' && c <= '\u9fff') {
                chineseCount++;
            } else {
                otherCount++;
            }
        }
        return chineseCount + (int)(otherCount * 0.35);
    }

    // 精确计数：调用 Python tiktoken 微服务
    public int countTokensExact(String text, String model) {
        return tokenCounterClient.count(text, model);
    }
}
```

## 参考资料

- [tiktoken GitHub](https://github.com/openai/tiktoken)
- [OpenAI Token 计价说明](https://openai.com/pricing)
- [BPE 算法原论文](https://arxiv.org/abs/1508.07909)

---

## 补充：Token 工程实践

### Token 预算管理

```python
import tiktoken
from openai import OpenAI
from typing import Optional

client = OpenAI()

class TokenBudgetManager:
    """Token 预算管理器：确保不超出上下文限制"""

    MODEL_LIMITS = {
        "gpt-4o": 128_000,
        "gpt-4o-mini": 128_000,
        "gpt-4-turbo": 128_000,
    }

    def __init__(self, model: str = "gpt-4o-mini", output_reserve: int = 1000):
        self.model = model
        self.output_reserve = output_reserve
        self.enc = tiktoken.encoding_for_model(model)
        self.max_tokens = self.MODEL_LIMITS.get(model, 8192)
        self.available_tokens = self.max_tokens - output_reserve

    def count(self, text: str) -> int:
        return len(self.enc.encode(text))

    def count_messages(self, messages: list) -> int:
        total = 0
        for msg in messages:
            total += 4  # 消息格式开销
            total += self.count(str(msg.get("content", "")))
            if "tool_calls" in msg:
                total += self.count(str(msg["tool_calls"]))
        total += 2  # 回复前缀
        return total

    def fit_messages(self, messages: list, budget: Optional[int] = None) -> list:
        """截断消息列表以适应 Token 预算"""
        budget = budget or self.available_tokens
        system_msgs = [m for m in messages if m["role"] == "system"]
        other_msgs = [m for m in messages if m["role"] != "system"]

        # 始终保留 system 消息
        system_tokens = self.count_messages(system_msgs)
        remaining = budget - system_tokens

        # 从最新消息开始添加
        fitted = []
        for msg in reversed(other_msgs):
            msg_tokens = self.count_messages([msg])
            if remaining - msg_tokens >= 0:
                fitted.insert(0, msg)
                remaining -= msg_tokens
            else:
                break

        return system_msgs + fitted

    def get_stats(self, messages: list) -> dict:
        used = self.count_messages(messages)
        return {
            "used_tokens": used,
            "available_tokens": self.available_tokens,
            "usage_pct": used / self.available_tokens * 100,
            "remaining_tokens": self.available_tokens - used,
            "is_safe": used < self.available_tokens * 0.9
        }

# 使用示例
manager = TokenBudgetManager("gpt-4o-mini", output_reserve=1000)

# 构建测试消息
messages = [
    {"role": "system", "content": "你是支付客服助手。"},
    {"role": "user", "content": "我的订单支付失败了"},
    {"role": "assistant", "content": "请提供订单号"},
    {"role": "user", "content": "PAY202401001"},
]

stats = manager.get_stats(messages)
print(f"Token 使用: {stats['used_tokens']} / {stats['available_tokens']}")
print(f"使用率: {stats['usage_pct']:.1f}%")
print(f"安全: {stats['is_safe']}")

# 批量处理时的 Token 成本估算
def estimate_batch_cost(
    items: list,
    prompt_template: str,
    model: str = "gpt-4o-mini",
    output_tokens_per_item: int = 200
) -> dict:
    """估算批量处理的 Token 成本"""
    PRICES = {
        "gpt-4o": (0.005, 0.015),
        "gpt-4o-mini": (0.00015, 0.0006),
    }

    enc = tiktoken.encoding_for_model(model)
    total_input_tokens = sum(
        len(enc.encode(prompt_template.format(item=item)))
        for item in items
    )
    total_output_tokens = len(items) * output_tokens_per_item

    input_price, output_price = PRICES.get(model, (0.005, 0.015))
    total_cost_usd = (
        total_input_tokens * input_price / 1000 +
        total_output_tokens * output_price / 1000
    )

    return {
        "items_count": len(items),
        "total_input_tokens": total_input_tokens,
        "total_output_tokens": total_output_tokens,
        "total_cost_usd": total_cost_usd,
        "total_cost_cny": total_cost_usd * 7.2,
        "cost_per_item_cny": total_cost_usd * 7.2 / len(items)
    }

# 估算处理1000条支付记录的成本
cost = estimate_batch_cost(
    items=[f"PAY{i:06d}" for i in range(1000)],
    prompt_template="分析支付订单 {item} 的风险",
    model="gpt-4o-mini"
)
print(f"\n批量处理1000条:")
print(f"  总成本: ¥{cost['total_cost_cny']:.2f}")
print(f"  单条成本: ¥{cost['cost_per_item_cny']:.4f}")
```

### 不同格式的 Token 效率对比

```python
import json

def compare_format_efficiency(data: dict) -> dict:
    """对比不同序列化格式的 Token 效率"""
    enc = tiktoken.encoding_for_model("gpt-4o-mini")

    formats = {
        "JSON（缩进）": json.dumps(data, ensure_ascii=False, indent=2),
        "JSON（紧凑）": json.dumps(data, ensure_ascii=False),
        "纯文本": "\n".join(f"{k}: {v}" for k, v in data.items()),
        "Markdown表格": "| 字段 | 值 |\n|------|----|\n" +
                        "\n".join(f"| {k} | {v} |" for k, v in data.items()),
    }

    results = {}
    for fmt_name, formatted_text in formats.items():
        token_count = len(enc.encode(formatted_text))
        results[fmt_name] = {
            "chars": len(formatted_text),
            "tokens": token_count,
            "efficiency": len(formatted_text) / token_count
        }

    return results

# 测试
order_data = {
    "订单号": "PAY202401001",
    "状态": "FAILED",
    "失败原因": "余额不足",
    "金额": "299.00",
    "账户ID": "ACC001"
}

comparison = compare_format_efficiency(order_data)
print("格式 Token 效率对比:")
for fmt, metrics in comparison.items():
    print(f"  {fmt}: {metrics['tokens']} tokens ({metrics['chars']} 字符)")

# 结论：紧凑 JSON 通常最省 Token
```
