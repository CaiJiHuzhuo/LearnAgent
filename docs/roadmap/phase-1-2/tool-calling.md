# 工具调用（Function Calling）

## 目标与重要性

工具调用是 Agent 与外部世界交互的核心能力。通过工具调用，Agent 可以查询数据库、调用 API、执行代码，从而完成复杂的业务任务。掌握工具调用是构建实用 Agent 的关键。

## 核心概念清单

- 工具定义（JSON Schema）
- 工具调用循环（ReAct Loop）
- 并行工具调用
- 工具调用的安全性
- 工具结果处理

## 学习路径

### 入门（第1层）

```python
from openai import OpenAI
import json

client = OpenAI()

# 定义工具
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_order_status",
            "description": "查询支付订单的当前状态和详细信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "订单号，格式：PAY后跟数字，例如 PAY202401001"
                    }
                },
                "required": ["order_id"],
                "additionalProperties": False
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_account_balance",
            "description": "查询用户账户余额",
            "parameters": {
                "type": "object",
                "properties": {
                    "user_id": {
                        "type": "string",
                        "description": "用户ID"
                    }
                },
                "required": ["user_id"]
            }
        }
    }
]

# 工具实现
def get_order_status(order_id: str) -> dict:
    # 实际生产中调用真实 API
    return {
        "order_id": order_id,
        "status": "FAILED",
        "fail_code": "INSUFFICIENT_BALANCE",
        "amount": 299.00,
        "user_id": "USER001"
    }

def get_account_balance(user_id: str) -> dict:
    return {
        "user_id": user_id,
        "balance": 50.00,
        "currency": "CNY"
    }

TOOL_REGISTRY = {
    "get_order_status": get_order_status,
    "get_account_balance": get_account_balance,
}

# 完整的工具调用循环
def run_agent(user_query: str) -> str:
    messages = [
        {"role": "system", "content": "你是支付客服助手，可以调用工具查询订单和账户信息。"},
        {"role": "user", "content": user_query}
    ]

    while True:
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,
            tool_choice="auto"
        )

        msg = response.choices[0].message
        finish_reason = response.choices[0].finish_reason

        # 添加 assistant 消息
        messages.append({
            "role": "assistant",
            "content": msg.content,
            "tool_calls": [
                {
                    "id": tc.id,
                    "type": tc.type,
                    "function": {"name": tc.function.name, "arguments": tc.function.arguments}
                }
                for tc in (msg.tool_calls or [])
            ] or None
        })

        if finish_reason == "stop":
            return msg.content

        if finish_reason == "tool_calls":
            for tool_call in msg.tool_calls:
                func_name = tool_call.function.name
                func_args = json.loads(tool_call.function.arguments)

                # 执行工具
                if func_name in TOOL_REGISTRY:
                    result = TOOL_REGISTRY[func_name](**func_args)
                else:
                    result = {"error": f"工具 {func_name} 不存在"}

                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result, ensure_ascii=False)
                })

result = run_agent("我的订单 PAY202401001 支付失败了，能查一下原因吗？")
print(result)
```

### 进阶（第2层）

```python
# 并行工具调用（多个工具同时执行）
import asyncio

async def run_agent_async(user_query: str) -> str:
    """异步 Agent，支持并行工具调用"""
    messages = [
        {"role": "system", "content": "你是支付分析助手。"},
        {"role": "user", "content": user_query}
    ]

    while True:
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,
            tool_choice="auto",
            parallel_tool_calls=True  # 允许并行调用
        )

        msg = response.choices[0].message
        finish_reason = response.choices[0].finish_reason

        messages.append({
            "role": "assistant",
            "content": msg.content,
            "tool_calls": [
                {
                    "id": tc.id,
                    "type": tc.type,
                    "function": {"name": tc.function.name, "arguments": tc.function.arguments}
                }
                for tc in (msg.tool_calls or [])
            ] or None
        })

        if finish_reason == "stop":
            return msg.content

        if finish_reason == "tool_calls":
            # 并行执行所有工具调用
            async def execute_tool(tool_call):
                func_name = tool_call.function.name
                func_args = json.loads(tool_call.function.arguments)
                if func_name in TOOL_REGISTRY:
                    # 如果是 IO 密集型，使用线程池
                    loop = asyncio.get_event_loop()
                    result = await loop.run_in_executor(
                        None,
                        lambda: TOOL_REGISTRY[func_name](**func_args)
                    )
                else:
                    result = {"error": f"工具不存在: {func_name}"}

                return tool_call.id, result

            results = await asyncio.gather(*[
                execute_tool(tc) for tc in msg.tool_calls
            ])

            for tool_call_id, result in results:
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call_id,
                    "content": json.dumps(result, ensure_ascii=False)
                })

# 工具调用安全校验
class SafeToolExecutor:
    def __init__(self, allowed_tools: set, read_only_mode: bool = True):
        self.allowed_tools = allowed_tools
        self.read_only_mode = read_only_mode
        self.call_log = []

        # 写操作工具集合
        self.write_tools = {"create_refund", "update_order", "send_notification"}

    def execute(self, tool_name: str, args: dict) -> dict:
        # 检查工具是否被允许
        if tool_name not in self.allowed_tools:
            return {"error": f"工具 {tool_name} 未被授权"}

        # 只读模式下禁止写操作
        if self.read_only_mode and tool_name in self.write_tools:
            return {"error": "当前处于只读模式，禁止执行写操作"}

        # 记录调用
        self.call_log.append({
            "tool": tool_name,
            "args": args,
            "timestamp": __import__("time").time()
        })

        # 执行工具
        if tool_name in TOOL_REGISTRY:
            return TOOL_REGISTRY[tool_name](**args)
        return {"error": "工具未实现"}
```

### 深入（第3层）

```python
# 工具调用追踪和调试
from dataclasses import dataclass, field
from typing import Any

@dataclass
class ToolCallTrace:
    tool_name: str
    args: dict
    result: Any
    duration_ms: float
    success: bool
    error: str | None = None

class TrackedAgent:
    def __init__(self):
        self.client = OpenAI()
        self.traces: list[ToolCallTrace] = []

    def run(self, query: str, tools: list, tool_registry: dict) -> str:
        messages = [
            {"role": "system", "content": "你是支付分析助手。"},
            {"role": "user", "content": query}
        ]

        iteration = 0
        max_iterations = 10

        while iteration < max_iterations:
            iteration += 1
            response = self.client.chat.completions.create(
                model="gpt-4o",
                messages=messages,
                tools=tools,
                tool_choice="auto"
            )

            msg = response.choices[0].message
            finish_reason = response.choices[0].finish_reason

            messages.append({
                "role": "assistant",
                "content": msg.content,
                "tool_calls": [
                    {
                        "id": tc.id,
                        "type": tc.type,
                        "function": {"name": tc.function.name, "arguments": tc.function.arguments}
                    }
                    for tc in (msg.tool_calls or [])
                ] or None
            })

            if finish_reason == "stop":
                return msg.content

            if finish_reason == "tool_calls":
                import time
                for tool_call in msg.tool_calls:
                    start = time.time()
                    func_name = tool_call.function.name
                    func_args = json.loads(tool_call.function.arguments)

                    try:
                        result = tool_registry[func_name](**func_args)
                        duration = (time.time() - start) * 1000
                        self.traces.append(ToolCallTrace(
                            tool_name=func_name,
                            args=func_args,
                            result=result,
                            duration_ms=duration,
                            success=True
                        ))
                    except Exception as e:
                        duration = (time.time() - start) * 1000
                        result = {"error": str(e)}
                        self.traces.append(ToolCallTrace(
                            tool_name=func_name,
                            args=func_args,
                            result=result,
                            duration_ms=duration,
                            success=False,
                            error=str(e)
                        ))

                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": json.dumps(result, ensure_ascii=False)
                    })

        return "达到最大迭代次数，任务可能未完成"

    def print_traces(self):
        for i, trace in enumerate(self.traces):
            status = "✓" if trace.success else "✗"
            print(f"{status} [{i+1}] {trace.tool_name}({trace.args}) → {trace.duration_ms:.0f}ms")
            if not trace.success:
                print(f"   错误: {trace.error}")
```

## 最小可执行练习

```python
# 构建一个支付失败排查 Agent
def build_payment_agent():
    tools = [...]  # 定义工具
    agent = TrackedAgent()
    result = agent.run(
        "分析订单 PAY001 的支付失败原因，查询所有必要信息",
        tools=TOOLS,
        tool_registry=TOOL_REGISTRY
    )
    print("分析结果:", result)
    print("\n调用追踪:")
    agent.print_traces()

build_payment_agent()
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 工具从不被调用 | 工具描述不够清晰 | 完善 description，说明何时应该调用 |
| 工具参数格式错误 | Schema 定义不精确 | 添加 pattern、enum 等约束 |
| 无限循环工具调用 | 工具结果不满足 LLM 预期 | 设置最大迭代次数，检查工具返回格式 |
| 并行调用顺序依赖 | 工具A的结果需要传给工具B | 按顺序调用，不使用并行 |

## Java/Spring Boot 对接要点

```java
// LangChain4j 中的工具定义
@Service
public class PaymentTools {

    @Tool("查询支付订单的状态和失败原因")
    public OrderStatus getOrderStatus(
        @P("订单号，格式：PAY后跟数字") String orderId
    ) {
        return orderService.findById(orderId)
            .map(order -> new OrderStatus(order.getStatus(), order.getFailCode()))
            .orElseThrow(() -> new RuntimeException("订单不存在: " + orderId));
    }

    @Tool("查询用户账户余额")
    public AccountBalance getAccountBalance(
        @P("用户ID") String userId
    ) {
        return accountService.getBalance(userId);
    }
}

// 配置 Agent 使用工具
@Bean
public AiServices<PaymentAgent> paymentAgent(
    ChatLanguageModel model,
    PaymentTools tools
) {
    return AiServices.builder(PaymentAgent.class)
        .chatLanguageModel(model)
        .tools(tools)
        .build();
}
```

## 参考资料

- [OpenAI Function Calling 文档](https://platform.openai.com/docs/guides/function-calling)
- [LangChain Tools 文档](https://python.langchain.com/docs/modules/tools/)
- [ReAct Agent 原论文](https://arxiv.org/abs/2210.03629)
