# 消息角色与会话结构

## 目标与重要性

消息角色（system/user/assistant/tool）是 LLM 对话的基础结构。正确使用角色能让 LLM 准确理解自己的职责、用户的意图和工具的返回值，是构建稳定 Agent 的基础。

## 核心概念清单

- `system`：系统指令，定义 LLM 的角色和行为约束
- `user`：用户输入
- `assistant`：LLM 的回复（包含工具调用）
- `tool`：工具调用的返回结果
- 多轮对话的消息结构
- Few-shot 示例的放置位置

## 学习路径

### 入门（第1层）

```python
from openai import OpenAI
client = OpenAI()

# 基础消息结构
messages = [
    # system: 定义 AI 的身份和行为规则
    {
        "role": "system",
        "content": """你是一个专业的支付系统客服助手。

你的职责：
1. 帮助用户理解支付失败原因
2. 提供可操作的解决建议
3. 必要时引导用户联系人工客服

限制：
- 不能泄露系统内部信息
- 不能承诺具体的退款时间
- 涉及金额超过1000元的问题，建议转人工"""
    },
    # user: 用户的第一条消息
    {
        "role": "user",
        "content": "我的订单 PAY202401001 支付失败了，怎么回事？"
    },
    # assistant: AI 的回复
    {
        "role": "assistant",
        "content": "您好！我来帮您查看一下。请问您用的是哪种支付方式？是银行卡还是余额支付？"
    },
    # user: 用户的第二条消息
    {
        "role": "user",
        "content": "余额支付，但余额明明够的"
    }
]

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages,
    temperature=0.1
)
print(response.choices[0].message.content)
```

### 进阶（第2层）

```python
# 工具调用的完整消息结构
import json

# 第一步：LLM 决定调用工具
messages_with_tool_call = [
    {"role": "system", "content": "你是支付分析助手，可以调用工具查询订单信息。"},
    {"role": "user", "content": "请查询订单 PAY001 的状态"},
    # assistant 消息包含 tool_calls
    {
        "role": "assistant",
        "content": None,  # 调用工具时 content 为 None
        "tool_calls": [
            {
                "id": "call_abc123",
                "type": "function",
                "function": {
                    "name": "get_order_info",
                    "arguments": json.dumps({"order_id": "PAY001"})
                }
            }
        ]
    },
    # tool 消息：工具的返回结果
    {
        "role": "tool",
        "tool_call_id": "call_abc123",  # 必须与上面的 id 对应
        "content": json.dumps({
            "order_id": "PAY001",
            "status": "FAILED",
            "fail_reason": "余额不足"
        })
    }
]

# 第二步：LLM 基于工具结果生成最终回答
final_response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages_with_tool_call,
    temperature=0.1
)
print(final_response.choices[0].message.content)

# Few-shot 示例的最佳放置位置
def create_few_shot_messages(examples: list, user_query: str) -> list:
    """在 system 中放少量示例，在 history 中放更多示例"""
    messages = [
        {
            "role": "system",
            "content": """你是支付错误代码翻译助手。

示例：
用户：INSUFFICIENT_BALANCE
助手：账户余额不足。建议：1)充值后重试 2)更换支付方式"""
        }
    ]

    # 将示例放入 user/assistant 轮次
    for example in examples:
        messages.append({"role": "user", "content": example["input"]})
        messages.append({"role": "assistant", "content": example["output"]})

    # 最后放实际查询
    messages.append({"role": "user", "content": user_query})
    return messages
```

### 深入（第3层）

```python
# 多 Assistant 角色的注意事项
# OpenAI 不支持在 messages 中直接模拟多个 assistant
# 解决方案：使用 name 字段区分

def create_multi_perspective_messages(question: str) -> list:
    """模拟多视角分析（使用 name 区分）"""
    return [
        {"role": "system", "content": "你是一个支付系统分析专家团队。"},
        {"role": "user", "content": question},
        # 通过 name 字段区分不同"专家"的视角
        {
            "role": "assistant",
            "name": "technical_expert",
            "content": "从技术角度看，这个错误代码通常是由..."
        },
        {
            "role": "assistant",
            "name": "business_expert",
            "content": "从业务角度看，这种情况通常发生在..."
        },
        {"role": "user", "content": "综合两位专家的意见，给出最终建议"},
    ]

# 处理 assistant 消息中的 tool_calls 序列化
def serialize_message(msg) -> dict:
    """安全地序列化包含 tool_calls 的消息"""
    if hasattr(msg, 'tool_calls') and msg.tool_calls:
        return {
            "role": msg.role,
            "content": msg.content,
            "tool_calls": [
                {
                    "id": tc.id,
                    "type": tc.type,
                    "function": {
                        "name": tc.function.name,
                        "arguments": tc.function.arguments
                    }
                }
                for tc in msg.tool_calls
            ]
        }
    return {"role": msg.role, "content": msg.content or ""}
```

## 最小可执行练习

```python
# 构建一个完整的多轮对话追踪器
class ConversationTracker:
    def __init__(self, system_prompt: str):
        self.messages = [{"role": "system", "content": system_prompt}]
        self.turn_count = 0

    def user_message(self, content: str) -> None:
        self.messages.append({"role": "user", "content": content})

    def assistant_message(self, content: str, tool_calls=None) -> None:
        msg = {"role": "assistant", "content": content}
        if tool_calls:
            msg["tool_calls"] = tool_calls
        self.messages.append(msg)
        self.turn_count += 1

    def tool_result(self, tool_call_id: str, result: str) -> None:
        self.messages.append({
            "role": "tool",
            "tool_call_id": tool_call_id,
            "content": result
        })

    def get_messages(self) -> list:
        return self.messages

    def export(self) -> str:
        import json
        return json.dumps(self.messages, ensure_ascii=False, indent=2)

# 测试
tracker = ConversationTracker("你是支付客服助手。")
tracker.user_message("我的支付失败了")
tracker.assistant_message("请提供订单号")
tracker.user_message("订单号是 PAY001")
print(tracker.export())
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `tool_call_id` 不匹配错误 | tool 消息的 id 与 assistant 中的不对应 | 确保严格使用 `tool_call.id` |
| system prompt 被忽略 | prompt 太长或与 user 消息冲突 | 精简 system prompt，避免矛盾指令 |
| 多轮对话失去上下文 | 没有保存历史消息 | 维护完整的 messages 列表 |
| assistant content=None 序列化失败 | 工具调用时 content 为 None | 序列化时检查 None |

## Java/Spring Boot 对接要点

```java
// Spring AI 中的消息构建
List<Message> messages = new ArrayList<>();
messages.add(new SystemMessage("你是支付系统客服助手"));
messages.add(new UserMessage("我的订单支付失败了"));
messages.add(new AssistantMessage("请提供订单号"));
messages.add(new UserMessage("PAY202401001"));

ChatResponse response = chatModel.call(new Prompt(messages));
```

## 参考资料

- [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat)
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)

---

## 补充：消息角色高级应用

### 角色扮演与角色切换

```python
from openai import OpenAI
import json

client = OpenAI()

# 高级系统 prompt：定义多个角色规则
MULTI_ROLE_SYSTEM = """你是支付系统客服中心的 AI 助手。

当处理不同类型的问题时，你扮演不同的角色：
- 账单问题 → 扮演"账单专员"
- 退款问题 → 扮演"退款审核员"  
- 技术故障 → 扮演"技术支持工程师"
- 投诉 → 扮演"客户关系经理"

每次回复开头用 [角色] 标注当前身份，例如：[技术支持工程师]"""

def route_to_role(user_query: str) -> str:
    messages = [
        {"role": "system", "content": MULTI_ROLE_SYSTEM},
        {"role": "user", "content": user_query}
    ]
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        temperature=0.1
    )
    return response.choices[0].message.content

# 测试不同问题的角色路由
test_queries = [
    "我的账单上有一笔我不认识的费用",
    "我要申请退款，商品有质量问题",
    "支付一直转圈圈，是系统问题吗",
    "你们的服务太差了！"
]

for query in test_queries:
    result = route_to_role(query)
    print(f"问题: {query}")
    print(f"回复: {result[:100]}...\n")
```

### 工具调用消息的完整生命周期

```python
# 完整的工具调用消息序列管理
class ToolCallConversation:
    """管理包含工具调用的完整对话序列"""

    def __init__(self, system_prompt: str, tools: list):
        self.messages = [{"role": "system", "content": system_prompt}]
        self.tools = tools
        self.client = OpenAI()

    def add_user(self, content: str) -> None:
        self.messages.append({"role": "user", "content": content})

    def run_turn(self, tool_registry: dict) -> str:
        """运行一轮对话，处理工具调用"""
        while True:
            response = self.client.chat.completions.create(
                model="gpt-4o",
                messages=self.messages,
                tools=self.tools,
                tool_choice="auto"
            )

            msg = response.choices[0].message
            finish_reason = response.choices[0].finish_reason

            # 构建 assistant 消息（注意 tool_calls 字段）
            assistant_msg = {
                "role": "assistant",
                "content": msg.content or ""
            }
            if msg.tool_calls:
                assistant_msg["tool_calls"] = [
                    {
                        "id": tc.id,
                        "type": "function",
                        "function": {
                            "name": tc.function.name,
                            "arguments": tc.function.arguments
                        }
                    }
                    for tc in msg.tool_calls
                ]
            self.messages.append(assistant_msg)

            if finish_reason == "stop":
                return msg.content or ""

            if finish_reason == "tool_calls":
                for tc in msg.tool_calls:
                    args = json.loads(tc.function.arguments)
                    if tc.function.name in tool_registry:
                        result = tool_registry[tc.function.name](**args)
                    else:
                        result = {"error": f"未知工具: {tc.function.name}"}

                    self.messages.append({
                        "role": "tool",
                        "tool_call_id": tc.id,
                        "content": json.dumps(result, ensure_ascii=False)
                    })

    def export_conversation(self) -> list:
        """导出完整对话历史（用于调试/存储）"""
        return [
            {
                "role": msg["role"],
                "content_preview": str(msg.get("content", ""))[:100],
                "has_tool_calls": "tool_calls" in msg
            }
            for msg in self.messages
        ]

# 使用示例
TOOLS = [{
    "type": "function",
    "function": {
        "name": "get_order",
        "description": "查询订单",
        "parameters": {
            "type": "object",
            "properties": {"order_id": {"type": "string"}},
            "required": ["order_id"]
        }
    }
}]

conv = ToolCallConversation("你是支付助手。", TOOLS)
conv.add_user("查询订单 PAY001")

result = conv.run_turn({
    "get_order": lambda order_id: {"status": "FAILED", "reason": "余额不足"}
})
print("回答:", result)
print("\n对话历史:")
for msg in conv.export_conversation():
    print(f"  [{msg['role']}] {msg['content_preview']}")
```
