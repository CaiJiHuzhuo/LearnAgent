# 结构化输出

## 目标与重要性

在 Agent 系统中，LLM 的输出通常需要被程序化处理：触发工具调用、更新数据库、返回 API 响应。结构化输出保证了输出的可靠解析，是 Agent 工程化的关键技术。

## 核心概念清单

- JSON Mode（格式保证）
- Structured Output（Schema 保证，OpenAI 2024新特性）
- Pydantic 模型验证
- 输出解析与错误修复
- 函数签名提取

## 学习路径

### 入门（第1层）

```python
from openai import OpenAI
from pydantic import BaseModel, Field
from typing import Literal
import json

client = OpenAI()

# 方式1：JSON Mode（只保证输出是合法JSON，不保证字段）
def analyze_with_json_mode(order_info: str) -> dict:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "system",
            "content": "你是支付分析助手。始终以JSON格式回复，包含root_cause和recommendations字段。"
        }, {
            "role": "user",
            "content": f"分析：{order_info}"
        }],
        response_format={"type": "json_object"},  # JSON Mode
        temperature=0.1
    )
    return json.loads(response.choices[0].message.content)

# 方式2：Structured Output（完全按 Schema 输出，推荐）
class FailureAnalysis(BaseModel):
    root_cause: str = Field(description="支付失败的根本原因")
    cause_category: Literal["账户问题", "渠道问题", "风控拦截", "系统故障", "用户操作"] = Field(
        description="原因类别"
    )
    severity: Literal["低", "中", "高"] = Field(description="严重程度")
    recommendations: list[str] = Field(description="建议措施列表，2-3条")
    needs_manual_review: bool = Field(description="是否需要人工审核")
    confidence: float = Field(ge=0.0, le=1.0, description="分析置信度 0-1")

def analyze_with_structured_output(order_info: str) -> FailureAnalysis:
    response = client.beta.chat.completions.parse(
        model="gpt-4o",  # Structured Output 需要 gpt-4o 系列
        messages=[{
            "role": "system",
            "content": "你是支付失败分析专家，请分析失败原因并给出结构化报告。"
        }, {
            "role": "user",
            "content": f"分析以下订单信息：{order_info}"
        }],
        response_format=FailureAnalysis,  # 直接传 Pydantic 类
        temperature=0.0
    )
    return response.choices[0].message.parsed

# 使用
result = analyze_with_structured_output(
    "订单PAY001，金额299元，失败代码INSUFFICIENT_BALANCE，账户余额50元"
)
print(f"根因：{result.root_cause}")
print(f"类别：{result.cause_category}")
print(f"置信度：{result.confidence:.0%}")
for rec in result.recommendations:
    print(f"  建议：{rec}")
```

### 进阶（第2层）

```python
# 复杂嵌套结构
from typing import Optional
from datetime import datetime

class OrderEvidence(BaseModel):
    field_name: str
    field_value: str
    interpretation: str

class RefundDecision(BaseModel):
    order_id: str
    eligible: bool
    decision: Literal["自动审批", "转人工", "自动拒绝"]
    rejection_reasons: list[str] = Field(default_factory=list)
    approved_amount: Optional[float] = None
    evidence: list[OrderEvidence] = Field(default_factory=list)
    policy_references: list[str] = Field(description="引用的政策条款")
    processing_notes: str = Field(description="处理备注")

def make_refund_decision(
    order: dict,
    policies: str,
    user_request: str
) -> RefundDecision:
    prompt = f"""基于以下信息，做出退款决策：

订单信息：
{json.dumps(order, ensure_ascii=False, indent=2)}

退款政策：
{policies}

用户申请：
{user_request}

请做出结构化的退款决策。"""

    response = client.beta.chat.completions.parse(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "你是退款审核专家，需要根据政策和订单信息做出准确的退款决策。"},
            {"role": "user", "content": prompt}
        ],
        response_format=RefundDecision,
        temperature=0.0
    )
    return response.choices[0].message.parsed

# 输出修复：当 LLM 输出不合规时
def safe_parse_json(text: str, model: type[BaseModel]) -> BaseModel | None:
    """尝试解析，失败时请求 LLM 修复"""
    # 首先尝试直接解析
    try:
        # 提取 JSON 块
        import re
        json_match = re.search(r'```json\s*([\s\S]*?)\s*```', text)
        if json_match:
            json_str = json_match.group(1)
        else:
            json_str = text

        data = json.loads(json_str)
        return model(**data)

    except (json.JSONDecodeError, Exception) as e:
        # 请求 LLM 修复
        fix_response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"""以下文本应该是符合此 Schema 的 JSON，但有格式错误。
请修复并只返回合法的 JSON，不要有任何解释：

Schema: {model.model_json_schema()}

待修复的文本：
{text}"""
            }],
            response_format={"type": "json_object"},
            temperature=0.0
        )
        try:
            data = json.loads(fix_response.choices[0].message.content)
            return model(**data)
        except Exception:
            return None
```

### 深入（第3层）

```python
# 流式结构化输出
from openai import OpenAI

def stream_structured_analysis(order_info: str):
    """流式获取结构化输出（用于实时展示进度）"""
    with client.beta.chat.completions.stream(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": f"分析支付失败：{order_info}"
        }],
        response_format=FailureAnalysis,
    ) as stream:
        # 可以监听流式事件
        for event in stream:
            if hasattr(event, 'type'):
                if event.type == 'content.delta':
                    print(".", end="", flush=True)

        # 获取最终完整对象
        final = stream.get_final_completion()
        return final.choices[0].message.parsed

# 动态 Schema 生成
def create_dynamic_schema(fields: list[dict]) -> type[BaseModel]:
    """根据配置动态创建 Pydantic 模型"""
    from pydantic import create_model
    field_definitions = {}
    for field in fields:
        field_type = {
            "string": str,
            "integer": int,
            "float": float,
            "boolean": bool,
            "list": list
        }.get(field["type"], str)

        if field.get("required", True):
            field_definitions[field["name"]] = (
                field_type,
                Field(description=field.get("description", ""))
            )
        else:
            field_definitions[field["name"]] = (
                Optional[field_type],
                Field(default=None, description=field.get("description", ""))
            )

    return create_model("DynamicModel", **field_definitions)
```

## 最小可执行练习

```python
# 构建一个通用的结构化输出管道
class StructuredOutputPipeline:
    def __init__(self, model_class: type[BaseModel]):
        self.client = OpenAI()
        self.model_class = model_class
        self.schema = model_class.model_json_schema()

    def extract(self, text: str, context: str = "") -> BaseModel:
        """从文本中提取结构化数据"""
        response = self.client.beta.chat.completions.parse(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": f"从文本中提取结构化信息。{context}"},
                {"role": "user", "content": text}
            ],
            response_format=self.model_class,
            temperature=0.0
        )
        return response.choices[0].message.parsed

# 测试
class PaymentIssueExtract(BaseModel):
    order_id: Optional[str] = Field(default=None, description="订单号")
    user_complaint: str = Field(description="用户投诉内容摘要")
    issue_type: Literal["支付失败", "退款问题", "账单问题", "其他"] = Field(description="问题类型")
    urgency: Literal["低", "中", "高"] = Field(description="紧急程度")

pipeline = StructuredOutputPipeline(PaymentIssueExtract)
result = pipeline.extract(
    "我昨天买东西，订单号PAY2024001，付了钱但是没到账，已经等了24小时了，很着急！！"
)
print(f"订单号: {result.order_id}")
print(f"问题类型: {result.issue_type}")
print(f"紧急程度: {result.urgency}")
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `parse()` 报错 | 模型不支持 Structured Output | 使用 gpt-4o 系列，不能用 mini 的旧版本 |
| 字段值不在 Literal 范围 | LLM 输出了不合法的枚举值 | Structured Output 会自动约束，或降低 temperature |
| 嵌套模型解析失败 | 复杂嵌套超出模型能力 | 拆分为多次调用，或简化 Schema |
| Optional 字段总是 None | Prompt 没有提供足够信息 | 在 Prompt 中明确要求提取该字段 |

## Java/Spring Boot 对接要点

```java
// 使用 Spring AI 的结构化输出
@Service
public class StructuredAnalysisService {

    private final ChatClient chatClient;

    public FailureAnalysis analyzeFailure(String orderInfo) {
        return chatClient.prompt()
            .user("分析支付失败：" + orderInfo)
            .call()
            .entity(FailureAnalysis.class);  // 自动解析为 Java 类
    }
}

// 对应的 Java 记录类
public record FailureAnalysis(
    String rootCause,
    String causeCategory,
    String severity,
    List<String> recommendations,
    boolean needsManualReview,
    double confidence
) {}
```

## 参考资料

- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Pydantic 文档](https://docs.pydantic.dev/)
- [Spring AI 结构化输出](https://docs.spring.io/spring-ai/reference/api/structured-output-converter.html)
