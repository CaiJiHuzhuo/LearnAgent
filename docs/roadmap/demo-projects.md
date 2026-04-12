# Demo 项目详解

## 项目一：支付失败排查助手

### 项目概述

一个能够自动分析支付失败原因、调用多个业务系统 API、生成结构化排查报告的 AI Agent。

### 架构图

```
用户输入（订单号/问题描述）
          │
          ▼
    ┌─────────────┐
    │  Intent     │  意图识别：是排查请求还是退款请求？
    │  Classifier │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Agent      │  ReAct 循环
    │  Executor   │  ┌─ Think: 我需要哪些信息？
    └──────┬──────┘  ├─ Act: 调用工具
           │         └─ Observe: 分析结果
           │
    ┌──────┴──────────────────────────────────┐
    │              工具集合                    │
    │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
    │  │查询订单   │  │查询账户  │  │查询渠道│ │
    │  │get_order │  │get_acct  │  │get_ch  │ │
    │  └──────────┘  └──────────┘  └────────┘ │
    │  ┌──────────┐  ┌──────────┐             │
    │  │查询风控  │  │查询限额  │             │
    │  │get_risk  │  │get_limit │             │
    │  └──────────┘  └──────────┘             │
    └─────────────────────────────────────────┘
           │
           ▼
    ┌─────────────┐
    │  Report     │  结构化输出：JSON 报告
    │  Generator  │  + Markdown 格式化
    └─────────────┘
```

### 工具列表

| 工具名称 | 功能 | 输入 | 输出 |
|---------|------|------|------|
| `get_order_info` | 查询订单详情 | order_id | OrderInfo |
| `get_account_status` | 查询账户状态 | account_id | AccountStatus |
| `get_payment_channel_status` | 查询支付渠道状态 | channel_id, time | ChannelStatus |
| `get_risk_result` | 查询风控结果 | order_id | RiskResult |
| `get_account_limit` | 查询账户限额 | account_id, limit_type | LimitInfo |
| `search_similar_failures` | 搜索相似失败案例 | keywords | List[Case] |

### Python Agent 实现

```python
import json
from typing import Any
from openai import OpenAI
from pydantic import BaseModel
from datetime import datetime

client = OpenAI()

# 数据模型
class OrderInfo(BaseModel):
    order_id: str
    amount: float
    currency: str
    status: str
    fail_code: str | None
    fail_reason: str | None
    created_at: datetime
    payment_channel: str
    account_id: str

class FailureReport(BaseModel):
    order_id: str
    root_cause: str
    cause_category: str  # "账户问题" | "渠道问题" | "风控拦截" | "系统故障" | "用户操作"
    evidence: list[str]
    recommended_actions: list[str]
    confidence: float
    analyst_notes: str

# 工具实现（实际生产中调用真实 API）
def get_order_info(order_id: str) -> dict:
    """查询订单信息"""
    # 实际生产：调用支付系统 API
    # response = requests.get(f"{PAYMENT_API}/orders/{order_id}", headers=auth_headers)
    # 此处用模拟数据
    return {
        "order_id": order_id,
        "amount": 299.00,
        "currency": "CNY",
        "status": "FAILED",
        "fail_code": "INSUFFICIENT_BALANCE",
        "fail_reason": "账户余额不足",
        "created_at": "2024-01-15T10:30:00Z",
        "payment_channel": "ALIPAY",
        "account_id": "ACC_001"
    }

def get_account_status(account_id: str) -> dict:
    """查询账户状态"""
    return {
        "account_id": account_id,
        "status": "ACTIVE",
        "balance": 50.00,
        "frozen_balance": 0.00,
        "daily_limit": 5000.00,
        "daily_used": 4980.00,
        "risk_level": "LOW"
    }

def get_payment_channel_status(channel_id: str, query_time: str) -> dict:
    """查询支付渠道状态"""
    return {
        "channel_id": channel_id,
        "status": "NORMAL",
        "success_rate_1h": 0.99,
        "avg_latency_ms": 350,
        "incidents": []
    }

def get_risk_result(order_id: str) -> dict:
    """查询风控结果"""
    return {
        "order_id": order_id,
        "risk_passed": True,
        "risk_score": 15,
        "risk_factors": [],
        "decision": "PASS"
    }

def get_account_limit(account_id: str, limit_type: str) -> dict:
    """查询账户限额"""
    return {
        "account_id": account_id,
        "limit_type": limit_type,
        "total_limit": 5000.00,
        "used_amount": 4980.00,
        "remaining": 20.00,
        "reset_time": "2024-01-16T00:00:00Z"
    }

# 工具注册
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_order_info",
            "description": "查询支付订单的详细信息，包括状态、失败原因、金额等",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string", "description": "订单号，格式如 PAY202401001"}
                },
                "required": ["order_id"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_account_status",
            "description": "查询用户账户状态，包括余额、限额使用情况、风险等级",
            "parameters": {
                "type": "object",
                "properties": {
                    "account_id": {"type": "string", "description": "账户ID"}
                },
                "required": ["account_id"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_payment_channel_status",
            "description": "查询支付渠道在指定时间的运行状态",
            "parameters": {
                "type": "object",
                "properties": {
                    "channel_id": {"type": "string", "description": "渠道ID，如 ALIPAY, WECHAT"},
                    "query_time": {"type": "string", "description": "查询时间，ISO格式"}
                },
                "required": ["channel_id", "query_time"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_risk_result",
            "description": "查询订单的风控审核结果",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string", "description": "订单号"}
                },
                "required": ["order_id"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_account_limit",
            "description": "查询账户的限额信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "account_id": {"type": "string", "description": "账户ID"},
                    "limit_type": {"type": "string", "description": "限额类型：DAILY/MONTHLY/SINGLE"}
                },
                "required": ["account_id", "limit_type"]
            }
        }
    }
]

TOOL_FUNCTIONS = {
    "get_order_info": get_order_info,
    "get_account_status": get_account_status,
    "get_payment_channel_status": get_payment_channel_status,
    "get_risk_result": get_risk_result,
    "get_account_limit": get_account_limit,
}

def execute_tool(tool_name: str, tool_args: dict) -> str:
    """执行工具调用"""
    if tool_name not in TOOL_FUNCTIONS:
        return json.dumps({"error": f"工具 {tool_name} 不存在"})
    result = TOOL_FUNCTIONS[tool_name](**tool_args)
    return json.dumps(result, ensure_ascii=False)

def analyze_payment_failure(order_id: str) -> FailureReport:
    """支付失败排查主函数"""
    system_prompt = """你是一个专业的支付系统故障分析助手。
你的职责是通过调用工具收集信息，分析支付失败的根本原因。

分析框架：
1. 首先查询订单基本信息，了解失败代码
2. 根据失败代码，有针对性地查询相关信息
   - 余额不足 → 查账户余额和限额
   - 风控拦截 → 查风控结果
   - 渠道异常 → 查渠道状态
3. 综合所有信息，给出根因分析和建议

请用中文回答，最终输出结构化的分析报告。"""

    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": f"请帮我分析订单 {order_id} 的支付失败原因，给出详细的排查报告。"}
    ]

    # ReAct 循环
    max_iterations = 10
    for i in range(max_iterations):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,
            tool_choice="auto",
            temperature=0.1
        )

        msg = response.choices[0].message
        messages.append({"role": "assistant", "content": msg.content, "tool_calls": msg.tool_calls})

        if response.choices[0].finish_reason == "stop":
            break

        if msg.tool_calls:
            for tool_call in msg.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                print(f"  调用工具: {tool_name}({tool_args})")
                result = execute_tool(tool_name, tool_args)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })

    # 最终结构化输出
    final_response = client.beta.chat.completions.parse(
        model="gpt-4o",
        messages=messages + [{
            "role": "user",
            "content": "请将分析结果整理为结构化的 JSON 格式报告"
        }],
        response_format=FailureReport
    )
    return final_response.choices[0].message.parsed

if __name__ == "__main__":
    report = analyze_payment_failure("PAY202401001")
    print(f"根因：{report.root_cause}")
    print(f"类别：{report.cause_category}")
    print(f"置信度：{report.confidence:.0%}")
    print(f"建议措施：")
    for action in report.recommended_actions:
        print(f"  - {action}")
```

### Java/Spring Boot 集成

```java
// PaymentFailureAnalysisService.java
@Service
@Slf4j
public class PaymentFailureAnalysisService {

    private final OpenAiClient openAiClient;
    private final OrderService orderService;
    private final AccountService accountService;
    private final PaymentChannelService channelService;

    public FailureAnalysisReport analyzeFailure(String orderId) {
        log.info("开始分析支付失败, orderId={}", orderId);

        // 构建初始消息
        List<ChatMessage> messages = new ArrayList<>();
        messages.add(new SystemMessage(SYSTEM_PROMPT));
        messages.add(new UserMessage("请分析订单 " + orderId + " 的支付失败原因"));

        // ReAct 循环
        for (int i = 0; i < MAX_ITERATIONS; i++) {
            ChatResponse response = openAiClient.chat(messages, TOOLS);
            ChatMessage assistantMsg = response.getMessage();
            messages.add(assistantMsg);

            if (response.isComplete()) break;

            // 处理工具调用
            for (ToolCall toolCall : assistantMsg.getToolCalls()) {
                String result = executeToolCall(toolCall);
                messages.add(new ToolMessage(toolCall.getId(), result));
            }
        }

        // 解析最终结果
        return parseReport(messages);
    }

    private String executeToolCall(ToolCall toolCall) {
        return switch (toolCall.getName()) {
            case "get_order_info" -> orderService.getOrderJson(
                toolCall.getArg("order_id"));
            case "get_account_status" -> accountService.getStatusJson(
                toolCall.getArg("account_id"));
            case "get_payment_channel_status" -> channelService.getStatusJson(
                toolCall.getArg("channel_id"), toolCall.getArg("query_time"));
            default -> "{"error": "unknown tool"}";
        };
    }
}
```

---

## 项目二：退款工单助手

### 项目概述

基于 RAG + 规则引擎的退款申请自动处理系统，能够理解用户退款诉求，查询相关政策，自动审核并生成处理决策。

### 架构图

```
用户提交退款申请
       │
       ▼
┌─────────────────┐
│  输入安全检查   │  过滤注入攻击、违规内容
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│  意图分类       │  退款申请/进度查询/投诉/其他
└───────┬─────────┘
        │ 退款申请
        ▼
┌─────────────────┐
│  信息抽取       │  订单号、退款原因、金额
└───────┬─────────┘
        │
        ├─────────────────────────────────────┐
        │                                     │
        ▼                                     ▼
┌─────────────────┐                 ┌─────────────────┐
│  RAG 检索       │                 │  业务系统查询   │
│  退款政策文档   │                 │  订单/交易信息  │
└───────┬─────────┘                 └───────┬─────────┘
        │                                   │
        └──────────────┬────────────────────┘
                       │
                       ▼
               ┌─────────────────┐
               │  规则引擎       │
               │  ├─ 时限检查    │
               │  ├─ 金额检查    │
               │  ├─ 次数限制    │
               │  └─ 黑名单      │
               └───────┬─────────┘
                       │
                       ▼
               ┌─────────────────┐
               │  LLM 决策       │  综合政策+规则+上下文
               └───────┬─────────┘
                       │
                       ▼
               ┌─────────────────┐
               │  人工审核队列   │  高风险/高金额 → 转人工
               │  或自动处理     │  低风险 → 自动审批
               └───────┬─────────┘
                       │
                       ▼
               ┌─────────────────┐
               │  结果通知       │  短信/邮件/App推送
               └─────────────────┘
```

### 安全设计

```python
# 安全层实现
import re
from typing import Optional

class RefundRequestSanitizer:
    """退款请求安全处理"""

    # 敏感字段脱敏
    SENSITIVE_PATTERNS = [
        (r'\d{16,19}', '****卡号****'),           # 银行卡号
        (r'\d{3}-\d{8}|\d{4}-\d{7,8}', '***'),  # 电话号码
        (r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+', '***@***.***'),  # 邮箱
    ]

    # Prompt Injection 特征
    INJECTION_PATTERNS = [
        r'忽略.*指令',
        r'ignore.*instructions',
        r'你现在是',
        r'system:',
        r'<.*>.*<\/.*>',
    ]

    def sanitize(self, user_input: str) -> tuple[str, list[str]]:
        """返回 (净化后文本, 告警列表)"""
        warnings = []

        # 检测注入
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, user_input, re.IGNORECASE):
                warnings.append(f"检测到潜在注入: {pattern}")

        # 脱敏
        sanitized = user_input
        for pattern, replacement in self.SENSITIVE_PATTERNS:
            sanitized = re.sub(pattern, replacement, sanitized)

        return sanitized, warnings

class RefundBusinessRules:
    """退款业务规则引擎"""

    def check_eligibility(self, order: dict, refund_request: dict) -> dict:
        rules_result = {
            "eligible": True,
            "rejections": [],
            "warnings": [],
            "require_manual_review": False
        }

        # 规则1：退款时限（7天）
        from datetime import datetime, timedelta
        order_time = datetime.fromisoformat(order["created_at"])
        days_since_order = (datetime.now() - order_time).days
        if days_since_order > 7:
            rules_result["eligible"] = False
            rules_result["rejections"].append(
                f"超过退款时限（已过{days_since_order}天，限7天）"
            )

        # 规则2：金额检查
        if refund_request.get("amount", 0) > order["amount"]:
            rules_result["eligible"] = False
            rules_result["rejections"].append("退款金额超过订单金额")

        # 规则3：大额审核（超过1000元需人工审核）
        if order["amount"] > 1000:
            rules_result["require_manual_review"] = True
            rules_result["warnings"].append("大额退款，需人工审核")

        # 规则4：退款次数限制
        if order.get("refund_count", 0) >= 3:
            rules_result["eligible"] = False
            rules_result["rejections"].append("该订单退款次数已达上限")

        return rules_result

async def process_refund_request(user_input: str, user_id: str) -> dict:
    """退款处理主流程"""

    sanitizer = RefundRequestSanitizer()
    rules = RefundBusinessRules()

    # 1. 安全检查
    clean_input, warnings = sanitizer.sanitize(user_input)
    if warnings:
        # 记录安全告警
        log_security_event(user_id, warnings)

    # 2. 信息抽取
    extraction = await extract_refund_info(clean_input)
    order_id = extraction.get("order_id")
    if not order_id:
        return {"status": "need_info", "message": "请提供您的订单号"}

    # 3. 查询订单
    order = await get_order(order_id)

    # 4. 规则检查
    rule_result = rules.check_eligibility(order, extraction)
    if not rule_result["eligible"]:
        return {
            "status": "rejected",
            "reasons": rule_result["rejections"]
        }

    # 5. RAG 检索退款政策
    policy_docs = await search_refund_policies(
        query=f"退款申请 {extraction.get('reason', '')}",
        k=3
    )

    # 6. LLM 决策
    decision = await llm_refund_decision(
        order=order,
        user_request=clean_input,
        policies=policy_docs,
        rule_result=rule_result
    )

    # 7. 根据决策执行
    if rule_result["require_manual_review"] or decision["confidence"] < 0.8:
        # 转人工审核
        ticket_id = await create_manual_review_ticket(order, decision)
        return {"status": "pending_review", "ticket_id": ticket_id}
    else:
        # 自动处理
        refund_id = await execute_refund(order_id, decision["amount"])
        return {"status": "approved", "refund_id": refund_id}
```

### 数据来源

| 数据类型 | 来源 | 更新频率 |
|---------|------|---------|
| 退款政策文档 | 产品/法务团队维护的 Confluence | 月更 |
| 订单信息 | 支付系统数据库 | 实时 |
| 用户信用评分 | 风控系统 | 小时级 |
| 历史退款记录 | 退款系统 | 实时 |

### 关键指标

| 指标 | 目标 | 测量方式 |
|------|------|---------|
| 自动处理率 | >70% | 统计自动审批比例 |
| 准确率 | >95% | 人工抽检核实 |
| 平均处理时间 | <30秒 | P50 延迟监控 |
| 用户满意度 | >4.2/5 | 后续问卷调查 |
