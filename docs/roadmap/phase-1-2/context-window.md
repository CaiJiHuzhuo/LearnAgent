# 上下文窗口管理

## 目标与重要性

上下文窗口（Context Window）是 LLM 能"看到"的信息总量。有效管理上下文是构建多轮对话、长文档处理和 Agent 系统的关键技能。上下文溢出会导致早期对话信息丢失，影响 Agent 的决策质量。

## 核心概念清单

- 上下文窗口大小（不同模型差异）
- Token 分布：system/user/assistant/tool
- 滑动窗口策略
- 上下文压缩技术
- 记忆层次结构
- 长文档处理策略

## 学习路径

### 入门（第1层）

```python
# 主流模型的上下文窗口大小（2024年）
MODEL_CONTEXT_WINDOWS = {
    "gpt-4o": 128_000,       # 128K tokens
    "gpt-4o-mini": 128_000,
    "gpt-4-turbo": 128_000,
    "claude-3-5-sonnet": 200_000,  # 200K tokens
    "gemini-1.5-pro": 1_000_000,   # 1M tokens
}

# 实际可用空间需要减去输出预留
def get_available_context(model: str, output_reserved: int = 2000) -> int:
    return MODEL_CONTEXT_WINDOWS.get(model, 8192) - output_reserved

# 监控上下文使用率
class ContextMonitor:
    def __init__(self, model: str = "gpt-4o-mini"):
        self.model = model
        self.max_tokens = MODEL_CONTEXT_WINDOWS[model]

    def check_usage(self, messages: list) -> dict:
        import tiktoken
        enc = tiktoken.encoding_for_model(self.model)
        used_tokens = sum(
            len(enc.encode(str(m.get("content", ""))))
            for m in messages
        )
        return {
            "used_tokens": used_tokens,
            "max_tokens": self.max_tokens,
            "usage_rate": used_tokens / self.max_tokens,
            "remaining_tokens": self.max_tokens - used_tokens,
            "warning": used_tokens > self.max_tokens * 0.8
        }
```

### 进阶（第2层）

```python
from collections import deque
from typing import Optional
import tiktoken

class SlidingWindowConversation:
    """滑动窗口对话管理"""

    def __init__(
        self,
        model: str = "gpt-4o-mini",
        max_tokens: int = 4000,
        system_prompt: str = ""
    ):
        self.model = model
        self.max_tokens = max_tokens
        self.system_prompt = system_prompt
        self.history: deque = deque()
        self.enc = tiktoken.encoding_for_model(model)

    def _count_tokens(self, text: str) -> int:
        return len(self.enc.encode(text))

    def _get_history_tokens(self) -> int:
        return sum(
            self._count_tokens(msg["content"])
            for msg in self.history
        )

    def add_message(self, role: str, content: str) -> None:
        self.history.append({"role": role, "content": content})

        # 当历史超出限制时，移除最旧的对话轮次（保留system）
        while self._get_history_tokens() > self.max_tokens and len(self.history) > 2:
            # 移除最旧的一对 user+assistant
            self.history.popleft()
            if self.history and self.history[0]["role"] == "assistant":
                self.history.popleft()

    def get_messages(self) -> list:
        messages = []
        if self.system_prompt:
            messages.append({"role": "system", "content": self.system_prompt})
        messages.extend(list(self.history))
        return messages

    def get_stats(self) -> dict:
        return {
            "turns": len(self.history) // 2,
            "tokens_used": self._get_history_tokens(),
            "max_tokens": self.max_tokens
        }

# 使用示例
from openai import OpenAI
client = OpenAI()

conv = SlidingWindowConversation(
    system_prompt="你是支付系统客服，帮助用户解决支付问题。",
    max_tokens=3000
)

questions = [
    "我的订单 PAY001 支付失败了",
    "失败原因是什么？",
    "我该怎么办？",
    "如果余额不足，我在哪里充值？",
]

for question in questions:
    conv.add_message("user", question)
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=conv.get_messages(),
        temperature=0.1
    )
    answer = response.choices[0].message.content
    conv.add_message("assistant", answer)
    print(f"Q: {question}")
    print(f"A: {answer[:100]}...")
    print(f"Stats: {conv.get_stats()}")
    print()
```

### 深入（第3层）

```python
# 上下文压缩：摘要长对话
async def compress_conversation(
    messages: list,
    keep_last_n: int = 4
) -> list:
    """将旧对话压缩为摘要，保留最近 N 轮"""
    if len(messages) <= keep_last_n + 1:  # +1 for system
        return messages

    system_msg = messages[0] if messages[0]["role"] == "system" else None
    recent_messages = messages[-keep_last_n:]
    old_messages = messages[1:-keep_last_n] if system_msg else messages[:-keep_last_n]

    # 使用 LLM 生成摘要
    summary_prompt = f"""请将以下对话历史压缩为一段简洁的摘要，保留所有重要信息：

{chr(10).join(f"{m['role']}: {m['content']}" for m in old_messages)}

要求：
1. 保留用户的问题要点
2. 保留助手提供的关键信息
3. 不超过200字"""

    summary_response = await openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": summary_prompt}],
        max_tokens=300
    )
    summary = summary_response.choices[0].message.content

    # 构建压缩后的消息列表
    compressed = []
    if system_msg:
        compressed.append(system_msg)
    compressed.append({
        "role": "system",
        "content": f"[对话历史摘要]: {summary}"
    })
    compressed.extend(recent_messages)

    return compressed
```

## 最小可执行练习

实现一个能处理长文档的问答系统：

```python
def chunk_document(text: str, chunk_size: int = 1000, overlap: int = 100) -> list:
    """将长文档切分为重叠的块"""
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        chunks.append(chunk)
    return chunks

def answer_from_long_document(question: str, document: str) -> str:
    """从长文档中回答问题（简化版 RAG）"""
    chunks = chunk_document(document)

    # 简单关键词匹配（实际应用用向量检索）
    relevant_chunks = [
        chunk for chunk in chunks
        if any(kw in chunk for kw in question.split())
    ][:3]

    context = "\n\n".join(relevant_chunks)
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"基于以下内容回答：\n{context}\n\n问题：{question}"
        }],
        max_tokens=500
    )
    return response.choices[0].message.content
```

## 常见坑与排查

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| 上下文溢出 | `context_length_exceeded` 错误 | 发送前检查 token 数，实现滑动窗口 |
| 早期信息丢失 | Agent 忘记之前的信息 | 实现对话摘要或外部记忆 |
| 系统 prompt 占比过大 | 留给对话的空间不足 | 精简系统 prompt，提取公共内容 |
| 工具结果太长 | 工具输出占满上下文 | 截断或摘要工具输出 |

## Java/Spring Boot 对接要点

```java
@Service
public class ConversationContextService {

    private final Map<String, Deque<ChatMessage>> sessions = new ConcurrentHashMap<>();

    public List<ChatMessage> getContext(String sessionId, String systemPrompt) {
        sessions.putIfAbsent(sessionId, new ArrayDeque<>());
        Deque<ChatMessage> history = sessions.get(sessionId);

        List<ChatMessage> messages = new ArrayList<>();
        messages.add(new SystemMessage(systemPrompt));

        // 限制历史条数（实际应基于 token 数）
        List<ChatMessage> recentHistory = history.stream()
            .skip(Math.max(0, history.size() - 20))
            .collect(Collectors.toList());
        messages.addAll(recentHistory);

        return messages;
    }

    public void addToContext(String sessionId, String role, String content) {
        sessions.get(sessionId).offer(new ChatMessage(role, content));
    }
}
```

## 参考资料

- [OpenAI Context Window 文档](https://platform.openai.com/docs/models)
- [LostInTheMiddle：长上下文的挑战](https://arxiv.org/abs/2307.03172)

---

## 补充：高级上下文管理技术

### KV Cache 与推理优化

```python
# 理解 KV Cache 对工程的意义
# KV Cache：推理时缓存历史 token 的 Key/Value，避免重复计算

# 工程实践：减少 system prompt 的变化
# 固定的 system prompt 可以被 KV Cache 缓存

# 不好的做法：每次都在 system prompt 中包含动态内容
def bad_practice(order_id: str) -> list:
    return [{
        "role": "system",
        "content": f"你是支付助手。当前时间：{__import__('datetime').datetime.now()}。订单：{order_id}"
        # 动态内容在 system 中 → 每次都无法缓存
    }]

# 好的做法：动态内容放在 user 消息中
def good_practice(order_id: str) -> list:
    return [
        {
            "role": "system",
            "content": "你是支付系统客服助手。"  # 固定 → 可以被缓存
        },
        {
            "role": "user",
            "content": f"分析订单 {order_id}（时间：{__import__('datetime').datetime.now().isoformat()}）"
            # 动态内容在 user 中
        }
    ]

# Prompt Caching（OpenAI 特性）
# 对于超过 1024 tokens 的 prompt，OpenAI 会自动缓存
# 缓存命中可节省 50% 的输入 token 成本

def create_cacheable_system_prompt() -> str:
    """创建可被缓存的 system prompt（内容固定、足够长）"""
    return """你是一个专业的支付系统客服助手，拥有以下能力和限制：

## 能力范围
1. 查询订单状态和支付记录
2. 分析支付失败原因
3. 指导用户完成退款申请
4. 解释支付政策和规则
5. 处理常见的账户问题

## 限制
1. 不能直接操作用户账户
2. 不能承诺具体的退款时间
3. 涉及金额超过1万元的问题，必须转人工
4. 不能泄露系统内部实现细节

## 回复规范
- 使用礼貌、专业的语气
- 回复简洁清晰，不超过200字
- 对敏感操作，引导用户通过官方渠道完成
- 遇到不确定的问题，诚实告知并转人工

## 常见支付错误代码
- INSUFFICIENT_BALANCE：余额不足
- CARD_EXPIRED：银行卡过期
- RISK_CONTROL_REJECTED：风控拦截
- CHANNEL_ERROR：支付渠道故障
- DAILY_LIMIT_EXCEEDED：超过日限额

## 退款政策
1. 自购买之日起7天内可申请全额退款
2. 7-30天内，需提供有效理由
3. 超过30天，原则上不退款，特殊情况人工处理
4. 大额退款（>1000元）需48小时人工审核"""

# 使用固定的长 system prompt（会被缓存）
FIXED_SYSTEM_PROMPT = create_cacheable_system_prompt()
```

### 多轮对话的记忆策略

```python
from typing import Optional
from enum import Enum

class MemoryStrategy(Enum):
    FULL_HISTORY = "full"          # 保留完整历史（短对话）
    SLIDING_WINDOW = "sliding"     # 滑动窗口（中等对话）
    SUMMARY_COMPRESSION = "summary" # 摘要压缩（长对话）
    HYBRID = "hybrid"              # 混合策略（生产推荐）

class HybridMemoryManager:
    """混合记忆策略：结合滑动窗口和摘要压缩"""

    def __init__(
        self,
        max_tokens: int = 4000,
        recent_turns_to_keep: int = 6,
        summary_interval: int = 10
    ):
        self.max_tokens = max_tokens
        self.recent_turns = recent_turns_to_keep
        self.summary_interval = summary_interval
        self.full_history = []
        self.summary = ""
        self.turn_count = 0

    def add_turn(self, role: str, content: str):
        self.full_history.append({"role": role, "content": content})
        if role == "assistant":
            self.turn_count += 1

        # 定期生成摘要
        if self.turn_count % self.summary_interval == 0 and self.turn_count > 0:
            self._generate_summary()

    def _generate_summary(self):
        """将旧对话压缩为摘要"""
        from openai import OpenAI
        client = OpenAI()

        old_turns = self.full_history[:-self.recent_turns * 2]
        if not old_turns:
            return

        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"请将以下对话历史压缩为200字以内的摘要：\n{str(old_turns)}"
            }],
            max_tokens=300,
            temperature=0.0
        )
        self.summary = response.choices[0].message.content

    def get_context_messages(self, system_prompt: str) -> list:
        """获取用于 API 调用的消息列表"""
        messages = [{"role": "system", "content": system_prompt}]

        if self.summary:
            messages.append({
                "role": "system",
                "content": f"[历史对话摘要]: {self.summary}"
            })

        # 最近 N 轮对话
        recent = self.full_history[-self.recent_turns * 2:]
        messages.extend(recent)

        return messages

# 使用示例
from openai import OpenAI
client = OpenAI()

memory = HybridMemoryManager(max_tokens=4000, recent_turns_to_keep=4)
system = FIXED_SYSTEM_PROMPT

# 模拟多轮对话
conversation = [
    ("user", "我的订单 PAY001 支付失败了"),
    ("user", "失败原因是什么"),
    ("user", "我该怎么处理"),
    ("user", "如果我的余额不足，可以用其他方式支付吗"),
]

for role, content in conversation:
    memory.add_turn(role, content)
    if role == "user":
        messages = memory.get_context_messages(system)
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            temperature=0.1,
            max_tokens=200
        )
        answer = response.choices[0].message.content
        memory.add_turn("assistant", answer)
        print(f"Q: {content[:50]}")
        print(f"A: {answer[:100]}...\n")
```
