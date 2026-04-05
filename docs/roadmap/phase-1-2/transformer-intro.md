# Transformer 原理深度解析

## 目标与重要性

Transformer 是现代 LLM 的核心架构。理解其工作原理能帮助你：
- 理解为什么某些 prompt 技巧有效
- 理解上下文窗口的本质限制
- 在需要时做出正确的模型选择
- 在团队中与 ML 工程师有效沟通

## 核心概念清单

- 自注意力机制（Self-Attention）
- Q/K/V 矩阵
- 多头注意力（Multi-Head Attention）
- 位置编码（Positional Encoding）
- 前馈神经网络（FFN）
- 编码器-解码器架构
- 层归一化（Layer Normalization）

## 数学基础

### 注意力机制的核心公式

注意力机制的核心公式：

```
Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V
```

其中：
- **Q (Query)**：当前 token 在"提问"什么
- **K (Key)**：每个 token 的"标签"，用于被查询
- **V (Value)**：每个 token 实际携带的信息
- **d_k**：Key 的维度（用于缩放，防止点积过大）

### 直觉理解

```
想象支付系统查询：

Q（查询）：我想了解"账户余额"
K（索引）：["订单金额", "账户余额", "渠道状态", "失败代码"]
V（内容）：[299元, 50元, 正常, INSUFFICIENT_BALANCE]

注意力分数：
- "账户余额" 与 Q 最匹配 → 高分 → 对应 V 权重大
- "订单金额" 与 Q 部分匹配 → 中等分 → 中等权重
- "渠道状态" 与 Q 不匹配 → 低分 → 低权重

最终输出 = 加权组合所有 V = 主要包含"50元"的信息
```

## 学习路径

### 入门（第1层）：自注意力机制

```python
import numpy as np
import torch
import torch.nn as nn
import torch.nn.functional as F

def simple_attention(Q: np.ndarray, K: np.ndarray, V: np.ndarray) -> np.ndarray:
    """最简单的注意力机制实现（用于理解原理）"""
    d_k = K.shape[-1]

    # 步骤1：计算注意力分数 (Q · K^T)
    scores = np.matmul(Q, K.T)  # shape: [seq_len, seq_len]

    # 步骤2：缩放（防止梯度消失/爆炸）
    scores = scores / np.sqrt(d_k)

    # 步骤3：Softmax → 注意力权重（和为1）
    # softmax(x_i) = exp(x_i) / sum(exp(x_j))
    attention_weights = np.exp(scores) / np.exp(scores).sum(axis=-1, keepdims=True)

    # 步骤4：加权求和 V
    output = np.matmul(attention_weights, V)  # shape: [seq_len, d_v]

    return output, attention_weights

# 演示：4个token的注意力
np.random.seed(42)
seq_len = 4
d_k = d_v = 8

# 假设4个token的Q/K/V向量
Q = np.random.randn(seq_len, d_k)
K = np.random.randn(seq_len, d_k)
V = np.random.randn(seq_len, d_v)

output, weights = simple_attention(Q, K, V)

print("注意力权重矩阵（每行和为1）:")
print(np.round(weights, 3))
print()
print("每个token主要关注哪个位置（argmax）:")
for i, row in enumerate(weights):
    print(f"  token{i} 最关注 token{np.argmax(row)}")
```

### 进阶（第2层）：多头注意力

```python
class MultiHeadAttention(nn.Module):
    """多头注意力机制"""

    def __init__(self, d_model: int, num_heads: int):
        super().__init__()
        assert d_model % num_heads == 0, "d_model 必须能被 num_heads 整除"

        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads  # 每个头的维度

        # 线性投影矩阵（每个头有独立的投影）
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)  # 输出投影

    def split_heads(self, x: torch.Tensor) -> torch.Tensor:
        """将 [batch, seq, d_model] 分为 [batch, heads, seq, d_k]"""
        batch_size, seq_len, _ = x.size()
        x = x.view(batch_size, seq_len, self.num_heads, self.d_k)
        return x.transpose(1, 2)  # [batch, heads, seq, d_k]

    def forward(
        self,
        query: torch.Tensor,
        key: torch.Tensor,
        value: torch.Tensor,
        mask: torch.Tensor = None
    ) -> tuple[torch.Tensor, torch.Tensor]:
        batch_size = query.size(0)

        # 1. 线性投影
        Q = self.split_heads(self.W_q(query))  # [batch, heads, seq, d_k]
        K = self.split_heads(self.W_k(key))
        V = self.split_heads(self.W_v(value))

        # 2. 缩放点积注意力（每个头独立计算）
        scores = torch.matmul(Q, K.transpose(-2, -1)) / (self.d_k ** 0.5)

        # 3. 可选：应用掩码（因果掩码/填充掩码）
        if mask is not None:
            scores = scores.masked_fill(mask == 0, -1e9)

        attention_weights = F.softmax(scores, dim=-1)

        # 4. 加权 V
        context = torch.matmul(attention_weights, V)  # [batch, heads, seq, d_k]

        # 5. 拼接所有头
        context = context.transpose(1, 2).contiguous()
        context = context.view(batch_size, -1, self.d_model)

        # 6. 最终线性变换
        output = self.W_o(context)

        return output, attention_weights

# 验证多头注意力
batch_size, seq_len, d_model = 2, 10, 512
num_heads = 8

mha = MultiHeadAttention(d_model=d_model, num_heads=num_heads)
x = torch.randn(batch_size, seq_len, d_model)
output, attn_weights = mha(x, x, x)

print(f"输入形状: {x.shape}")
print(f"输出形状: {output.shape}")   # 应与输入相同
print(f"注意力权重形状: {attn_weights.shape}")  # [batch, heads, seq, seq]
print(f"每个头的维度: {d_model // num_heads}")
```

### 深入（第3层）：完整 Transformer Block

```python
class TransformerBlock(nn.Module):
    """单个 Transformer Block（Decoder-only，GPT风格）"""

    def __init__(self, d_model: int, num_heads: int, d_ff: int, dropout: float = 0.1):
        super().__init__()

        # 多头自注意力
        self.attention = MultiHeadAttention(d_model, num_heads)

        # 前馈神经网络（FFN）：两层线性变换 + ReLU/GELU
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),    # 第一层：扩展到 d_ff（通常是 4*d_model）
            nn.GELU(),                    # 激活函数（GPT使用GELU，比ReLU更平滑）
            nn.Dropout(dropout),
            nn.Linear(d_ff, d_model),    # 第二层：压缩回 d_model
        )

        # 层归一化（在每个子层之前应用）
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)

        self.dropout = nn.Dropout(dropout)

    def forward(
        self,
        x: torch.Tensor,
        causal_mask: torch.Tensor = None
    ) -> torch.Tensor:
        """
        Pre-norm 架构（现代 GPT 使用）：
        x = x + Attention(LayerNorm(x))
        x = x + FFN(LayerNorm(x))
        """
        # 自注意力子层（带残差连接）
        normed_x = self.norm1(x)
        attn_output, _ = self.attention(normed_x, normed_x, normed_x, causal_mask)
        x = x + self.dropout(attn_output)  # 残差连接

        # FFN 子层（带残差连接）
        x = x + self.dropout(self.ffn(self.norm2(x)))  # 残差连接

        return x

class MiniGPT(nn.Module):
    """极简 GPT 实现（用于理解原理）"""

    def __init__(
        self,
        vocab_size: int,
        d_model: int = 256,
        num_heads: int = 4,
        num_layers: int = 4,
        d_ff: int = 1024,
        max_seq_len: int = 512,
    ):
        super().__init__()
        self.d_model = d_model

        # Token Embedding：将 token ID 映射为向量
        self.token_embedding = nn.Embedding(vocab_size, d_model)

        # 位置编码：给每个位置一个唯一向量
        # 可学习的位置编码（GPT风格）
        self.position_embedding = nn.Embedding(max_seq_len, d_model)

        # Transformer Blocks（堆叠多层）
        self.blocks = nn.ModuleList([
            TransformerBlock(d_model, num_heads, d_ff)
            for _ in range(num_layers)
        ])

        # 最终层归一化
        self.final_norm = nn.LayerNorm(d_model)

        # 语言模型头：将向量映射回词表
        self.lm_head = nn.Linear(d_model, vocab_size, bias=False)

    def make_causal_mask(self, seq_len: int) -> torch.Tensor:
        """因果掩码：防止当前 token 看到未来 token"""
        # 下三角矩阵
        mask = torch.tril(torch.ones(seq_len, seq_len))
        return mask  # 1=可见, 0=不可见

    def forward(self, input_ids: torch.Tensor) -> torch.Tensor:
        batch_size, seq_len = input_ids.shape

        # 创建位置索引
        positions = torch.arange(seq_len, device=input_ids.device).unsqueeze(0)

        # Token + 位置嵌入
        x = self.token_embedding(input_ids) + self.position_embedding(positions)

        # 因果掩码（确保 token 只能关注之前的 token）
        causal_mask = self.make_causal_mask(seq_len).to(input_ids.device)

        # 逐层 Transformer
        for block in self.blocks:
            x = block(x, causal_mask)

        # 最终归一化
        x = self.final_norm(x)

        # 预测下一个 token 的概率
        logits = self.lm_head(x)  # [batch, seq, vocab_size]

        return logits

# 创建一个极小的 GPT
vocab_size = 1000  # 词表大小
model = MiniGPT(vocab_size=vocab_size, d_model=64, num_heads=4, num_layers=2)

# 计算参数量
total_params = sum(p.numel() for p in model.parameters())
print(f"模型参数量: {total_params:,}")
print(f"（GPT-3有1750亿参数，GPT-4估计约1万亿参数）")

# 前向传播
input_ids = torch.randint(0, vocab_size, (2, 10))  # batch=2, seq_len=10
logits = model(input_ids)
print(f"\n输入形状: {input_ids.shape}")
print(f"输出 logits 形状: {logits.shape}")  # [2, 10, 1000]
```

## 位置编码详解

```python
import matplotlib.pyplot as plt
import math

def sinusoidal_positional_encoding(max_seq_len: int, d_model: int) -> np.ndarray:
    """
    原始 Transformer 论文的正弦位置编码
    PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
    PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
    """
    PE = np.zeros((max_seq_len, d_model))

    for pos in range(max_seq_len):
        for i in range(0, d_model, 2):
            PE[pos, i] = math.sin(pos / (10000 ** (2*i / d_model)))
            if i + 1 < d_model:
                PE[pos, i+1] = math.cos(pos / (10000 ** (2*i / d_model)))

    return PE

# 可视化位置编码
PE = sinusoidal_positional_encoding(50, 64)
print("位置编码矩阵形状:", PE.shape)
print("位置0的前10维:", np.round(PE[0, :10], 3))
print("位置1的前10维:", np.round(PE[1, :10], 3))
print()

# 两个相邻位置的余弦相似度（应该很高）
cos_sim_adjacent = np.dot(PE[0], PE[1]) / (np.linalg.norm(PE[0]) * np.linalg.norm(PE[1]))
# 两个较远位置的余弦相似度（应该较低）
cos_sim_far = np.dot(PE[0], PE[49]) / (np.linalg.norm(PE[0]) * np.linalg.norm(PE[49]))
print(f"相邻位置(0,1)的相似度: {cos_sim_adjacent:.3f}")
print(f"较远位置(0,49)的相似度: {cos_sim_far:.3f}")
```

## 关键概念对照表

| 概念 | 直觉理解 | 工程意义 |
|------|---------|---------|
| Q/K/V 矩阵 | 查询/索引/内容 | 通过学习的线性变换提取不同语义 |
| 多头注意力 | 多个"专家"独立分析 | 不同头关注不同类型的关系（语法/语义/位置）|
| 残差连接 | 保留原始信息 | 解决深层网络的梯度消失问题 |
| 层归一化 | 稳定训练 | 加速收敛，允许更深的网络 |
| 因果掩码 | 只看历史，不看未来 | 确保语言模型的自回归特性 |
| 上下文窗口 | 注意力矩阵的大小 | 越大越贵：O(n²) 内存和计算 |
| KV Cache | 缓存已计算的 K/V | 推理加速：避免重复计算历史 token |

## 为什么长上下文"更贵"

```python
# 注意力机制的计算复杂度分析
def attention_complexity(seq_len: int, d_model: int) -> dict:
    """
    注意力机制的计算量：
    - QK^T 矩阵乘法: O(seq_len^2 * d_model)
    - 内存占用: O(seq_len^2)（存储注意力矩阵）
    """
    attention_ops = seq_len ** 2 * d_model
    attention_memory = seq_len ** 2

    return {
        "seq_len": seq_len,
        "attention_ops": attention_ops,
        "attention_memory_tokens": attention_memory,
        "relative_cost": attention_ops / attention_complexity(1000, d_model)["attention_ops"]
    }

# 对比不同上下文长度的成本
for seq_len in [1000, 4000, 16000, 64000, 128000]:
    info = attention_complexity(seq_len, d_model=4096)
    print(f"seq_len={seq_len:7d}: 相对计算量 = {info['relative_cost']:.0f}x")

# 输出：
# seq_len=  1000: 相对计算量 = 1x
# seq_len=  4000: 相对计算量 = 16x
# seq_len= 16000: 相对计算量 = 256x
# seq_len= 64000: 相对计算量 = 4096x
# seq_len=128000: 相对计算量 = 16384x
```

## Transformer 架构演进

```
2017: 原始 Transformer (Vaswani et al.)
      编码器-解码器 → 适合翻译任务

2018: GPT-1 (OpenAI)
      Decoder-only → 适合生成任务
      1.17亿参数

2018: BERT (Google)
      Encoder-only → 适合理解任务（分类、问答）
      3.4亿参数

2019: GPT-2
      15亿参数，首次展示涌现能力

2020: GPT-3
      1750亿参数，Few-shot 学习能力突破

2022: ChatGPT = GPT-3.5 + RLHF
      指令跟随能力大幅提升

2023: GPT-4
      多模态，能力大幅提升

2024: GPT-4o
      更快、更便宜、原生多模态

关键趋势：
- Scale Laws：参数量增加 → 能力提升（可预测的幂律关系）
- Emergent Abilities：某些能力在参数量达到阈值后突然涌现
- RLHF：让模型行为与人类偏好对齐
```

## 最小可执行练习

```python
# 用 PyTorch 实现并验证注意力机制
import torch
import torch.nn.functional as F

def scaled_dot_product_attention(
    Q: torch.Tensor,
    K: torch.Tensor,
    V: torch.Tensor,
    mask: torch.Tensor = None
) -> tuple[torch.Tensor, torch.Tensor]:
    d_k = Q.size(-1)
    scores = torch.matmul(Q, K.transpose(-2, -1)) / (d_k ** 0.5)

    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))

    weights = F.softmax(scores, dim=-1)
    output = torch.matmul(weights, V)
    return output, weights

# 测试
seq_len, d_k = 5, 8
Q = torch.randn(seq_len, d_k)
K = torch.randn(seq_len, d_k)
V = torch.randn(seq_len, d_k)

# 无掩码（encoder style）
output_enc, weights_enc = scaled_dot_product_attention(Q, K, V)
print("编码器注意力权重（每行和为1）:")
print(weights_enc.round(decimals=3))

# 因果掩码（decoder/GPT style）
causal_mask = torch.tril(torch.ones(seq_len, seq_len))
output_dec, weights_dec = scaled_dot_product_attention(Q, K, V, causal_mask)
print("\n解码器注意力权重（下三角）:")
print(weights_dec.round(decimals=3))
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 注意力权重全相同 | Q、K 初始化不当 | 使用 Xavier 初始化 |
| 梯度消失 | 没有残差连接 | 确保每个子层有残差连接 |
| 训练不稳定 | 没有层归一化 | 在每个子层前/后加 LayerNorm |
| 长序列 OOM | 注意力矩阵 O(n²) | 使用 Flash Attention 或降低 seq_len |

## Java/Spring Boot 对接要点

理解 Transformer 的意义对 Java 工程师：

```java
// 1. 理解为什么 LLM 调用需要等待
//    → 自回归生成：每次只生成一个 token，需要多次前向传播

// 2. 理解流式输出的意义
//    → 每生成一个 token 就可以立即返回，减少感知延迟
@GetMapping(value = "/analyze/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> analyzeStream(@RequestParam String orderId) {
    return chatModel.stream(new Prompt("分析订单 " + orderId))
        .map(response -> response.getResult().getOutput().getContent());
}

// 3. 理解 KV Cache 的意义
//    → 推理优化：缓存历史 token 的 Key/Value，避免重复计算
//    → 实践：减少 system prompt 长度，避免每次都重新计算
```

## 参考资料

- [Attention Is All You Need 原论文](https://arxiv.org/abs/1706.03762)
- [The Illustrated Transformer (Jay Alammar)](https://jalammar.github.io/illustrated-transformer/)
- [GPT-3 论文](https://arxiv.org/abs/2005.14165)
- [Understanding LLMs - 综合综述](https://arxiv.org/abs/2307.06435)
- [Flash Attention 论文](https://arxiv.org/abs/2205.14135)
