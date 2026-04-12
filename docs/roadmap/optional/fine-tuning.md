# 模型微调（Fine-tuning）（可选专题）

## 目标与重要性

通过微调将通用 LLM 适配到特定业务领域，在专业准确性和响应风格上超越 Prompt Engineering 的极限。

## 核心概念清单

- 模型微调（Fine-tuning）的核心原理
- 主流工具和框架
- 适用场景和限制
- 与主线 Agent 系统的集成
- 生产环境注意事项

## 学习路径

### 入门（第1层）

```python
# 模型微调（Fine-tuning） - 基础入门
from openai import OpenAI
from pydantic import BaseModel
from typing import Optional, List, Dict, Any
import json, asyncio

client = OpenAI()

# 模型微调（Fine-tuning） 的基础使用模式
class FineTuningConfig(BaseModel):
    """配置"""
    enabled: bool = True
    timeout_seconds: int = 30
    max_retries: int = 3
    debug_mode: bool = False

class FineTuningClient:
    """基础客户端"""

    def __init__(self, config: FineTuningConfig):
        self.config = config
        self.client = OpenAI()

    def execute(self, task: str, context: Dict[str, Any] = None) -> Dict:
        """执行任务"""
        system_prompt = f"你是专业的 模型微调（Fine-tuning） 执行器。按照用户要求完成任务。"
        user_msg = task
        if context:
            user_msg += f"\n\n相关信息：{json.dumps(context, ensure_ascii=False)}"

        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_msg}
            ],
            temperature=0.1,
            max_tokens=1000
        )

        return {
            "task": task,
            "result": response.choices[0].message.content,
            "tokens_used": response.usage.total_tokens,
            "success": True
        }

# 基础使用示例
config = FineTuningConfig()
executor = FineTuningClient(config)
result = executor.execute("演示 模型微调（Fine-tuning） 的基本用法")
print(result["result"][:300])
```

### 进阶（第2层）

```python
# 模型微调（Fine-tuning） 进阶：完整业务场景
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)

@dataclass
class TaskResult:
    success: bool
    output: Any
    steps: List[Dict] = field(default_factory=list)
    errors: List[str] = field(default_factory=list)
    metadata: Dict = field(default_factory=dict)

class AdvancedFineTuning:
    """进阶 模型微调（Fine-tuning） 实现"""

    def __init__(self, config: FineTuningConfig):
        self.config = config
        self.client = OpenAI()
        self.step_history: List[Dict] = []

    def plan_steps(self, goal: str) -> List[Dict]:
        """将目标分解为执行步骤"""
        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "你是任务规划专家，将复杂目标分解为具体步骤。"},
                {"role": "user", "content": f"请将以下目标分解为3-5个具体步骤（JSON格式）：{goal}"}
            ],
            response_format={"type": "json_object"},
            temperature=0.1
        )
        plan = json.loads(response.choices[0].message.content)
        return plan.get("steps", [])

    def execute_step(self, step: Dict, previous_results: List) -> TaskResult:
        """执行单个步骤"""
        context = {
            "step": step,
            "previous_results": previous_results[-3:] if previous_results else []
        }

        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "你是 模型微调（Fine-tuning） 执行专家，按步骤完成任务。"},
                {"role": "user", "content": f"执行步骤：{json.dumps(context, ensure_ascii=False)}"}
            ],
            temperature=0.1
        )

        result = TaskResult(
            success=True,
            output=response.choices[0].message.content,
            metadata={"step_name": step.get("name", "未命名")}
        )
        self.step_history.append({"step": step, "result": result})
        return result

    def execute_plan(self, goal: str) -> TaskResult:
        """执行完整计划"""
        logger.info(f"开始执行 模型微调（Fine-tuning） 任务: {goal[:50]}")

        # 规划步骤
        steps = self.plan_steps(goal)
        logger.info(f"共 {len(steps)} 个步骤")

        # 逐步执行
        all_results = []
        for i, step in enumerate(steps):
            logger.info(f"执行步骤 {i+1}/{len(steps)}: {step.get('name', '未命名')}")
            step_result = self.execute_step(step, all_results)
            all_results.append(step_result)

            if not step_result.success:
                return TaskResult(
                    success=False,
                    output=None,
                    errors=[f"步骤 {i+1} 失败"],
                    steps=[r.__dict__ for r in all_results]
                )

        # 汇总结果
        final_response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "user", "content": f"汇总以下步骤结果，生成最终报告：\n{json.dumps([r.__dict__ for r in all_results], ensure_ascii=False, default=str)}"}
            ],
            temperature=0.1
        )

        return TaskResult(
            success=True,
            output=final_response.choices[0].message.content,
            steps=[r.__dict__ for r in all_results],
            metadata={"total_steps": len(steps)}
        )

# 演示
agent = AdvancedFineTuning(FineTuningConfig())
result = agent.execute_plan("模型微调（Fine-tuning） 在支付系统中的应用场景分析")
print(f"成功: {result.success}")
print(f"步骤数: {len(result.steps)}")
print(f"输出: {str(result.output)[:200]}...")
```

### 深入（第3层）

```python
# 模型微调（Fine-tuning） 生产级集成
import asyncio
from typing import AsyncGenerator
import uuid

class ProductionFineTuningPipeline:
    """生产级 模型微调（Fine-tuning） 流水线"""

    def __init__(self):
        self.client = OpenAI()
        self._pipeline_stages = []
        self._running_tasks: Dict[str, asyncio.Task] = {}

    def add_stage(self, name: str, processor: callable):
        """添加流水线阶段"""
        self._pipeline_stages.append({"name": name, "processor": processor})
        return self

    async def execute_pipeline(
        self,
        input_data: Dict,
        pipeline_id: str = None
    ) -> AsyncGenerator[Dict, None]:
        """流式执行流水线，实时输出每个阶段的结果"""
        pipeline_id = pipeline_id or str(uuid.uuid4())[:8]
        current_data = input_data

        for stage in self._pipeline_stages:
            stage_start = asyncio.get_event_loop().time()

            try:
                # 执行阶段
                result = await asyncio.get_event_loop().run_in_executor(
                    None,
                    stage["processor"],
                    current_data
                )
                stage_time = (asyncio.get_event_loop().time() - stage_start) * 1000

                stage_result = {
                    "pipeline_id": pipeline_id,
                    "stage": stage["name"],
                    "success": True,
                    "output": result,
                    "duration_ms": stage_time
                }
                yield stage_result

                # 传递到下一阶段
                current_data = result if isinstance(result, dict) else {"data": result}

            except Exception as e:
                yield {
                    "pipeline_id": pipeline_id,
                    "stage": stage["name"],
                    "success": False,
                    "error": str(e)
                }
                return  # 流水线中断

    async def run(self, input_data: Dict) -> Dict:
        """运行完整流水线，返回最终结果"""
        final_output = input_data
        async for stage_result in self.execute_pipeline(input_data):
            if not stage_result["success"]:
                return {"success": False, "error": stage_result.get("error"), "stage": stage_result["stage"]}
            final_output = stage_result.get("output", final_output)
            print(f"  阶段 {stage_result['stage']}: ✓ ({stage_result.get('duration_ms', 0):.0f}ms)")

        return {"success": True, "result": final_output}

# 构建 模型微调（Fine-tuning） 流水线
pipeline = ProductionFineTuningPipeline()

def stage_validate(data: Dict) -> Dict:
    """验证输入数据"""
    assert "query" in data, "缺少 query 字段"
    return {**data, "validated": True}

def stage_process(data: Dict) -> Dict:
    """核心处理"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": f"处理 模型微调（Fine-tuning） 请求: {data.get('query')}"}],
        max_tokens=200
    )
    return {**data, "result": response.choices[0].message.content}

def stage_format(data: Dict) -> Dict:
    """格式化输出"""
    return {
        "final_result": data.get("result", ""),
        "query": data.get("query", ""),
        "pipeline": "模型微调（Fine-tuning）"
    }

pipeline.add_stage("验证", stage_validate)
pipeline.add_stage("处理", stage_process)
pipeline.add_stage("格式化", stage_format)

async def run_pipeline_demo():
    print(f"=== 模型微调（Fine-tuning） 流水线演示 ===")
    result = await pipeline.run({"query": "支付系统中 模型微调（Fine-tuning） 的实际应用"})
    print(f"\n最终结果: {result['success']}")
    if result["success"]:
        print(f"输出: {str(result['result'])[:200]}...")

asyncio.run(run_pipeline_demo())
```

## 最小可执行练习

```python
# 模型微调（Fine-tuning） 综合练习
async def complete_optional_exercise():
    """完整的 模型微调（Fine-tuning） 练习"""
    print(f"\n=== 模型微调（Fine-tuning） 可选专题练习 ===\n")

    # 初始化
    config = FineTuningConfig(debug_mode=True)
    client_basic = FineTuningClient(config)

    # 场景1：基础执行
    print("1. 基础功能验证")
    result = client_basic.execute(
        f"用3句话介绍 模型微调（Fine-tuning） 的主要应用场景",
        {"domain": "支付系统", "use_case": "业务优化"}
    )
    print(f"   ✓ 基础执行成功，使用 {result['tokens_used']} tokens")

    # 场景2：规划执行
    print("\n2. 多步骤规划执行")
    agent = AdvancedFineTuning(config)
    planned_result = agent.execute_plan(
        f"设计 模型微调（Fine-tuning） 在退款工单系统中的集成方案"
    )
    print(f"   ✓ 规划执行: {len(planned_result.steps)} 步骤完成")

    # 场景3：流水线执行
    print("\n3. 流水线执行")
    pipe = ProductionFineTuningPipeline()
    pipe.add_stage("验证", lambda d: {**d, "valid": True})
    pipe.add_stage("处理", lambda d: client_basic.execute(d.get("query", ""), d))
    pipe_result = await pipe.run({"query": f"模型微调（Fine-tuning） 最佳实践总结"})
    print(f"   ✓ 流水线执行: {'成功' if pipe_result['success'] else '失败'}")

    print(f"\n=== 模型微调（Fine-tuning） 练习完成 ===")

asyncio.run(complete_optional_exercise())
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 初始化失败 | 依赖未安装或配置错误 | 检查 pip 安装和环境变量 |
| 执行超时 | 处理逻辑过于复杂 | 设置合理超时，拆分步骤 |
| 输出不稳定 | temperature 过高 | 设置 temperature=0 或添加验证层 |
| 成本过高 | 频繁调用大模型 | 使用缓存，简单任务用小模型 |
| 安全漏洞 | 未对输入做安全检查 | 在入口处添加输入验证和过滤 |

## Java/Spring Boot 对接要点

```java
// 模型微调（Fine-tuning） 的 Java 集成示例
@RestController
@RequestMapping("/api/v1/fine-tuning")
@Slf4j
public class FineTuningController {

    private final FineTuningService service;

    @PostMapping("/execute")
    public ResponseEntity<ExecutionResult> execute(
        @RequestBody ExecutionRequest request,
        @RequestHeader(value = "X-Trace-Id", defaultValue = "") String traceId
    ) {
        log.info("模型微调（Fine-tuning） 请求: traceId={}, query={}", traceId, request.query());

        try {
            ExecutionResult result = service.execute(request.query(), request.context());
            return ResponseEntity.ok(result);
        } catch (Exception e) {
            log.error("模型微调（Fine-tuning） 执行失败: {}", e.getMessage(), e);
            return ResponseEntity.internalServerError()
                .body(ExecutionResult.failure(e.getMessage()));
        }
    }

    // 流式输出端点
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> stream(@RequestParam String query) {
        return service.streamExecute(query)
            .map(chunk -> ServerSentEvent.<String>builder()
                .data(chunk)
                .build());
    }
}

// 服务实现
@Service
public class FineTuningService {

    private final ChatClient chatClient;

    public ExecutionResult execute(String query, Map<String, Object> context) {
        String contextStr = context != null ? context.toString() : "无";
        String response = chatClient.prompt()
            .system("你是专业的 模型微调（Fine-tuning） 执行器")
            .user(String.format("任务：%s\n上下文：%s", query, contextStr))
            .call()
            .content();

        return ExecutionResult.success(response);
    }

    public Flux<String> streamExecute(String query) {
        return chatClient.prompt()
            .user(query)
            .stream()
            .content();
    }
}
```

## 参考资料

- [LangChain 模型微调（Fine-tuning） 文档](https://python.langchain.com/docs)
- [Spring AI 官方文档](https://docs.spring.io/spring-ai/reference/)
- [OpenAI API 参考](https://platform.openai.com/docs/api-reference)
- [生产级 AI Agent 最佳实践](https://www.deeplearning.ai/the-batch/issue-242/)
