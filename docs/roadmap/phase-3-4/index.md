# Phase 3-4 索引：RAG 与上下文工程

## 阶段目标

完成 Phase 3-4 后，你应该能够：

1. 理解并实现完整的 RAG（检索增强生成）流水线
2. 掌握 Embedding 原理，选择合适的向量模型
3. 设计文档分块策略，平衡检索精度与完整性
4. 操作主流向量数据库（ChromaDB、Milvus、Qdrant）
5. 实现引用溯源，让 LLM 输出可验证
6. 检测和拒绝幻觉输出
7. 将业务规则与 LLM 推理结合
8. 建立评估体系，支持回归测试
9. 完成生产级部署（Docker、API 封装、监控）

## 知识点列表（10个）

| 编号 | 文件 | 主题 | 预计学习时间 |
|------|------|------|-------------|
| 2.1 | [context-engineering.md](context-engineering.md) | 上下文工程 | 4小时 |
| 2.2 | [embedding-similarity.md](embedding-similarity.md) | Embedding 与相似度 | 4小时 |
| 2.3 | [chunking-strategies.md](chunking-strategies.md) | 分块策略 | 3小时 |
| 2.4 | [rag-pipeline.md](rag-pipeline.md) | RAG 流水线 | 6小时 |
| 2.5 | [vector-db.md](vector-db.md) | 向量数据库 | 4小时 |
| 2.6 | [citation-traceability.md](citation-traceability.md) | 引用溯源 | 3小时 |
| 2.7 | [hallucination-rejection.md](hallucination-rejection.md) | 幻觉检测与拒绝 | 4小时 |
| 2.8 | [business-rules.md](business-rules.md) | 业务规则引擎 | 3小时 |
| 2.9 | [evaluation-regression.md](evaluation-regression.md) | 评估与回归测试 | 4小时 |
| 2.10 | [deployment-engineering.md](deployment-engineering.md) | 部署工程化 | 4小时 |

**总计约 39小时（约2周）**

## 核心架构

```
用户查询
    │
    ▼
[查询处理层]
  ├── 查询改写
  ├── 意图识别
  └── 关键词提取
    │
    ▼
[检索层]
  ├── 向量检索 (Embedding)
  ├── 关键词检索 (BM25)
  └── 混合检索
    │
    ▼
[重排序层]
  ├── 相关性过滤
  └── 分数融合
    │
    ▼
[生成层]
  ├── Prompt 组装
  ├── LLM 调用
  └── 引用注入
    │
    ▼
[验证层]
  ├── 幻觉检测
  ├── 业务规则校验
  └── 引用核实
    │
    ▼
最终输出（带引用）
```

## 阶段产出物

1. **完整 RAG 服务**：支持文档上传、检索、生成
2. **评估报告**：至少100个测试用例的评估结果
3. **部署文档**：Docker Compose 配置 + 部署说明
