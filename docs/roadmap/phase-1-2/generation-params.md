# 生成参数调优

## 目标与重要性

生成参数决定了 LLM 输出的质量和风格。错误的参数配置是导致输出不稳定、格式混乱的常见原因。掌握这些参数能让你在不同业务场景下得到最佳输出。

## 核心概念清单

- `temperature`：输出随机性
- `top_p`（nucleus sampling）：累积概率截断
- `top_k`：候选 token 数量限制
- `max_tokens`：输出长度上限
- `frequency_penalty`：频率惩罚（减少重复）
- `presence_penalty`：存在惩罚（鼓励新话题）
- `stop`：停止序列
- `seed`：随机种子（可复现）

## 学习路径

### 入门（第1层）

最重要的参数：`temperature`

```python
from openai import OpenAI
client = OpenAI()

def test_temperature(prompt: str, temperatures: list) -> None:
    for temp in temperatures:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=temp,
            max_tokens=100
        )
        print(f"temperature={temp}: {response.choices[0].message.content}")
        print()

# 测试：同一个 prompt，不同 temperature
test_temperature(
    "支付失败的可能原因是什么？列出3个。",
    temperatures=[0.0, 0.5, 1.0, 1.5]
)
# temperature=0.0 → 每次输出相同，最确定的答案
# temperature=1.5 → 每次输出不同，更有创意但可能不准确
```

### 进阶（第2层）

```python
# 不同场景的参数配置建议
PARAMETER_PRESETS = {
    # 数据分析、代码生成、事实问答 → 需要确定性
    "factual": {
        "temperature": 0.0,
        "top_p": 1.0,
        "frequency_penalty": 0.0,
        "presence_penalty": 0.0,
    },

    # 客服问答 → 稳定但不机械
    "customer_service": {
        "temperature": 0.1,
        "top_p": 0.95,
        "frequency_penalty": 0.1,
        "presence_penalty": 0.0,
    },

    # 创意写作、营销文案 → 需要变化
    "creative": {
        "temperature": 0.9,
        "top_p": 0.95,
        "frequency_penalty": 0.3,
        "presence_penalty": 0.3,
    },

    # 代码生成 → 确定性高
    "code_generation": {
        "temperature": 0.0,
        "top_p": 1.0,
        "frequency_penalty": 0.0,
        "presence_penalty": 0.0,
        "stop": ["```\n\n", "# End"],
    },

    # 结构化输出（JSON）→ 最高确定性
    "structured_output": {
        "temperature": 0.0,
        "top_p": 1.0,
    },
}

def create_completion(prompt: str, preset: str = "customer_service", **kwargs):
    params = {**PARAMETER_PRESETS[preset], **kwargs}
    return client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=kwargs.get("max_tokens", 500),
        **{k: v for k, v in params.items() if k != "max_tokens"}
    )
```

### 深入（第3层）

```python
# 使用 seed 实现可复现输出（用于测试）
def reproducible_test(prompt: str, seed: int = 42) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0,  # 与 seed 配合使用
        seed=seed
    )
    # 注意：seed 不保证100%可复现，但大大降低随机性
    fingerprint = response.system_fingerprint
    return response.choices[0].message.content, fingerprint

# 使用 stop 序列控制输出格式
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{
        "role": "user",
        "content": "列出支付失败的原因，每行一个，只需要原因不需要解释："
    }],
    stop=["\n\n"],  # 遇到空行就停止
    max_tokens=200
)

# 动态调整 max_tokens
def smart_completion(prompt: str, min_tokens: int = 50, max_tokens: int = 1000):
    """根据 prompt 复杂度动态调整 max_tokens"""
    import tiktoken
    enc = tiktoken.encoding_for_model("gpt-4o-mini")
    prompt_tokens = len(enc.encode(prompt))

    # 简单问题少给 tokens，复杂问题多给
    estimated_output = min(max_tokens, max(min_tokens, prompt_tokens // 2))

    return client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.1,
        max_tokens=estimated_output
    )
```

## 最小可执行练习

```python
# 参数对比实验
import json

def parameter_comparison_experiment():
    prompt = "分析订单 PAY001 支付失败的可能原因，给出3个最常见的原因。"

    experiments = [
        ("低温度（精确模式）", {"temperature": 0.0}),
        ("中温度（平衡模式）", {"temperature": 0.5}),
        ("高温度（创意模式）", {"temperature": 1.2}),
        ("频率惩罚（减少重复）", {"temperature": 0.7, "frequency_penalty": 0.8}),
    ]

    results = {}
    for name, params in experiments:
        outputs = []
        for _ in range(3):  # 每个配置运行3次
            resp = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[{"role": "user", "content": prompt}],
                max_tokens=150,
                **params
            )
            outputs.append(resp.choices[0].message.content)

        # 检查3次输出是否一致
        consistent = len(set(outputs)) == 1
        results[name] = {
            "params": params,
            "consistent": consistent,
            "sample_output": outputs[0][:100]
        }

    print(json.dumps(results, ensure_ascii=False, indent=2))

parameter_comparison_experiment()
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 输出每次不同 | temperature > 0 | 设置 temperature=0 或使用 seed |
| 输出被截断 | max_tokens 太小 | 增大 max_tokens 或检查是否真的需要更长输出 |
| 输出重复词语 | 没有频率惩罚 | 设置 frequency_penalty=0.3~0.5 |
| JSON 格式混乱 | temperature 太高 | 结构化输出时设置 temperature=0 |

## Java/Spring Boot 对接要点

```java
// Spring AI 中的参数配置
@Configuration
public class OpenAiConfig {

    @Bean
    public ChatClient factualChatClient(OpenAiChatModel model) {
        return ChatClient.builder(model)
            .defaultOptions(OpenAiChatOptions.builder()
                .withTemperature(0.0f)
                .withTopP(1.0f)
                .withMaxTokens(1000)
                .build())
            .build();
    }

    @Bean
    public ChatClient creativeChatClient(OpenAiChatModel model) {
        return ChatClient.builder(model)
            .defaultOptions(OpenAiChatOptions.builder()
                .withTemperature(0.9f)
                .withFrequencyPenalty(0.3f)
                .withPresencePenalty(0.3f)
                .build())
            .build();
    }
}
```

## 参考资料

- [OpenAI API 参数文档](https://platform.openai.com/docs/api-reference/chat)
- [Temperature 与采样策略](https://huggingface.co/blog/how-to-generate)

---

## 补充：高级参数应用场景

### 生产环境参数配置模板

```python
from openai import OpenAI
from pydantic import BaseModel, Field
from typing import Literal, Optional

client = OpenAI()

class GenerationConfig(BaseModel):
    """生成参数配置模型"""
    temperature: float = Field(0.1, ge=0.0, le=2.0)
    top_p: float = Field(0.95, ge=0.0, le=1.0)
    max_tokens: int = Field(500, ge=1, le=4096)
    frequency_penalty: float = Field(0.0, ge=-2.0, le=2.0)
    presence_penalty: float = Field(0.0, ge=-2.0, le=2.0)
    stop: Optional[list] = None
    seed: Optional[int] = None

# 业务场景参数配置库
CONFIGS = {
    "payment_analysis": GenerationConfig(temperature=0.0, seed=42),
    "customer_service": GenerationConfig(temperature=0.2, max_tokens=300),
    "report_generation": GenerationConfig(temperature=0.1, max_tokens=2000),
    "creative_writing": GenerationConfig(temperature=1.0, frequency_penalty=0.5),
}

def call_with_config(prompt: str, config_name: str) -> str:
    config = CONFIGS[config_name]
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=config.temperature,
        top_p=config.top_p,
        max_tokens=config.max_tokens,
        frequency_penalty=config.frequency_penalty,
        presence_penalty=config.presence_penalty,
        stop=config.stop,
        seed=config.seed
    )
    return response.choices[0].message.content

# 参数调优实验框架
class ParamTuningExperiment:
    def __init__(self, model: str = "gpt-4o-mini"):
        self.client = OpenAI()
        self.model = model
        self.results = []

    def run_experiment(
        self,
        prompt: str,
        param_grid: dict,
        n_samples: int = 3
    ) -> list:
        """网格搜索最优参数"""
        import itertools

        # 生成所有参数组合
        keys = param_grid.keys()
        values = param_grid.values()
        combinations = list(itertools.product(*values))

        for combo in combinations:
            params = dict(zip(keys, combo))
            outputs = []

            for _ in range(n_samples):
                response = self.client.chat.completions.create(
                    model=self.model,
                    messages=[{"role": "user", "content": prompt}],
                    max_tokens=200,
                    **params
                )
                outputs.append(response.choices[0].message.content)

            self.results.append({
                "params": params,
                "outputs": outputs,
                "consistency": len(set(outputs)) == 1,
                "avg_length": sum(len(o) for o in outputs) / len(outputs)
            })

        return self.results

# 使用示例
experiment = ParamTuningExperiment()
results = experiment.run_experiment(
    prompt="分析支付失败原因：INSUFFICIENT_BALANCE",
    param_grid={
        "temperature": [0.0, 0.5, 1.0],
        "top_p": [1.0, 0.9],
    },
    n_samples=2
)

for r in results:
    print(f"参数: {r['params']} | 一致性: {r['consistency']} | 平均长度: {r['avg_length']:.0f}")
```

### 停止序列的高级用法

```python
# 使用停止序列精确控制输出格式
def generate_structured_list(topic: str) -> list:
    """使用停止序列生成格式化列表"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"列出{topic}的5个要点，每行以'-'开头，每点不超过20字："
        }],
        stop=["6.", "\n\n"],  # 生成第6点或空行时停止
        max_tokens=300,
        temperature=0.1
    )
    content = response.choices[0].message.content

    # 解析列表
    items = [
        line.lstrip("- ").strip()
        for line in content.split("\n")
        if line.strip().startswith("-")
    ]
    return items

items = generate_structured_list("支付失败的常见原因")
for i, item in enumerate(items, 1):
    print(f"{i}. {item}")
```

### 参数对成本和延迟的影响

```python
import time
import statistics

def benchmark_params(prompts: list, params_list: list) -> dict:
    """基准测试不同参数配置的性能"""
    benchmark_results = {}

    for params in params_list:
        label = str(params)
        latencies = []
        token_counts = []

        for prompt in prompts:
            start = time.time()
            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[{"role": "user", "content": prompt}],
                **params
            )
            latency = (time.time() - start) * 1000
            latencies.append(latency)
            token_counts.append(response.usage.total_tokens)

        benchmark_results[label] = {
            "avg_latency_ms": statistics.mean(latencies),
            "p95_latency_ms": sorted(latencies)[int(len(latencies) * 0.95)],
            "avg_tokens": statistics.mean(token_counts),
        }

    return benchmark_results

test_prompts = ["支付失败原因？"] * 5
param_configs = [
    {"temperature": 0.0, "max_tokens": 100},
    {"temperature": 0.5, "max_tokens": 200},
    {"temperature": 1.0, "max_tokens": 500},
]

results = benchmark_params(test_prompts, param_configs)
for config, metrics in results.items():
    print(f"{config}: latency={metrics['avg_latency_ms']:.0f}ms, tokens={metrics['avg_tokens']:.0f}")
```
