# 上下文工程

## 目标与重要性

上下文工程是 RAG 系统的核心：如何组织和压缩信息，使得 LLM 在有限的上下文窗口内获得最佳输出。好的上下文工程能将 RAG 系统的准确率提升20-40%。

## 核心概念清单

- 上下文窗口分配策略
- 信息优先级排序
- 上下文压缩技术
- 动态上下文构建
- 多轮对话的上下文管理

## 学习路径

### 入门（第1层）

```python
# 基础上下文构建
def build_rag_context(
    user_query: str,
    retrieved_docs: list,
    max_context_tokens: int = 2000
) -> str:
    """构建 RAG 上下文"""
    import tiktoken
    enc = tiktoken.encoding_for_model("gpt-4o-mini")

    context_parts = []
    used_tokens = 0

    for i, doc in enumerate(retrieved_docs):
        doc_text = f"[文档{i+1}] {doc['content']}"
        doc_tokens = len(enc.encode(doc_text))

        if used_tokens + doc_tokens > max_context_tokens:
            break

        context_parts.append(doc_text)
        used_tokens += doc_tokens

    return "\n\n".join(context_parts)

def create_rag_prompt(user_query: str, context: str) -> list:
    return [
        {
            "role": "system",
            "content": """你是一个基于知识库的问答助手。
严格基于提供的上下文内容回答问题。
如果上下文中没有相关信息，明确说明无法回答。
回答时引用相关文档编号。"""
        },
        {
            "role": "user",
            "content": f"""上下文信息：
{context}

用户问题：{user_query}"""
        }
    ]
```

### 进阶（第2层）

```python
# 上下文压缩：MapReduce 模式
from openai import OpenAI
client = OpenAI()

def compress_document(doc: str, query: str, max_tokens: int = 200) -> str:
    """将文档压缩为与查询相关的摘要"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"""请从以下文档中提取与问题最相关的信息，
压缩为不超过{max_tokens//10}句话。

文档：{doc}

问题：{query}

相关摘要："""
        }],
        max_tokens=max_tokens,
        temperature=0.0
    )
    return response.choices[0].message.content

def map_reduce_rag(user_query: str, docs: list) -> str:
    """Map-Reduce RAG：先压缩每个文档，再合并生成答案"""
    # Map：压缩每个文档
    compressed_docs = [
        compress_document(doc["content"], user_query)
        for doc in docs
    ]

    # Reduce：基于压缩后的内容生成答案
    merged_context = "\n\n".join([
        f"[来源{i+1}] {text}"
        for i, text in enumerate(compressed_docs)
    ])

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": f"基于以下信息回答：\n{merged_context}\n\n问题：{user_query}"
        }],
        temperature=0.1
    )
    return response.choices[0].message.content

# 上下文优先级排序
def rank_context_by_relevance(
    query: str,
    docs: list,
    top_k: int = 5
) -> list:
    """按相关性对上下文进行排序和过滤"""
    from sentence_transformers import SentenceTransformer, util
    import torch

    model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')

    query_embedding = model.encode(query, convert_to_tensor=True)
    doc_embeddings = model.encode(
        [d["content"] for d in docs],
        convert_to_tensor=True
    )

    # 计算余弦相似度
    scores = util.pytorch_cos_sim(query_embedding, doc_embeddings)[0]

    # 排序并返回 top_k
    ranked_indices = torch.argsort(scores, descending=True)[:top_k]
    return [
        {**docs[i], "relevance_score": scores[i].item()}
        for i in ranked_indices
    ]
```

### 深入（第3层）

```python
# 自适应上下文：根据问题类型动态调整
from enum import Enum
from pydantic import BaseModel

class QueryType(Enum):
    FACTUAL = "factual"         # 事实查询：需要精确信息
    ANALYTICAL = "analytical"   # 分析查询：需要多文档综合
    PROCEDURAL = "procedural"   # 步骤查询：需要完整流程
    COMPARATIVE = "comparative" # 对比查询：需要多个视角

class ContextStrategy(BaseModel):
    max_docs: int
    max_tokens_per_doc: int
    use_compression: bool
    include_metadata: bool

STRATEGIES = {
    QueryType.FACTUAL: ContextStrategy(
        max_docs=3, max_tokens_per_doc=500,
        use_compression=False, include_metadata=True
    ),
    QueryType.ANALYTICAL: ContextStrategy(
        max_docs=8, max_tokens_per_doc=300,
        use_compression=True, include_metadata=True
    ),
    QueryType.PROCEDURAL: ContextStrategy(
        max_docs=5, max_tokens_per_doc=600,
        use_compression=False, include_metadata=False
    ),
    QueryType.COMPARATIVE: ContextStrategy(
        max_docs=6, max_tokens_per_doc=400,
        use_compression=True, include_metadata=True
    ),
}

def classify_query(query: str) -> QueryType:
    """分类查询类型"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"""将以下问题分类为：factual/analytical/procedural/comparative
问题：{query}
只返回类型名称，不需要解释。"""
        }],
        temperature=0.0
    )
    type_str = response.choices[0].message.content.strip().lower()
    return QueryType(type_str) if type_str in QueryType._value2member_map_ else QueryType.FACTUAL

def adaptive_context_build(query: str, retrieved_docs: list) -> str:
    query_type = classify_query(query)
    strategy = STRATEGIES[query_type]

    selected_docs = retrieved_docs[:strategy.max_docs]

    parts = []
    for i, doc in enumerate(selected_docs):
        content = doc["content"][:strategy.max_tokens_per_doc * 4]

        if strategy.use_compression:
            content = compress_document(content, query, strategy.max_tokens_per_doc)

        metadata = ""
        if strategy.include_metadata:
            metadata = f"[来源: {doc.get('source', '未知')}] "

        parts.append(f"{metadata}文档{i+1}: {content}")

    return "\n\n".join(parts)
```

## 最小可执行练习

```python
# 对比不同上下文策略的效果
def compare_context_strategies(query: str, docs: list):
    strategies = {
        "全量上下文": lambda: "\n".join(d["content"] for d in docs[:5]),
        "相关性排序": lambda: "\n".join(
            d["content"] for d in rank_context_by_relevance(query, docs)
        ),
        "压缩上下文": lambda: "\n".join(
            compress_document(d["content"], query) for d in docs[:5]
        ),
        "自适应": lambda: adaptive_context_build(query, docs),
    }

    results = {}
    for strategy_name, build_context in strategies.items():
        context = build_context()
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": f"{context}\n\n{query}"}],
            max_tokens=300
        )
        results[strategy_name] = {
            "context_length": len(context),
            "answer": response.choices[0].message.content[:100]
        }
    return results
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 上下文太长导致遗忘 | LLM 对中间内容的注意力较弱 | 将最重要的信息放在上下文开头或结尾 |
| 噪声文档降低质量 | 检索到不相关文档 | 添加相关性过滤，设置最低分数阈值 |
| 压缩丢失关键信息 | 压缩 Prompt 不够精准 | 在压缩指令中明确关键信息类型 |

## Java/Spring Boot 对接要点

```java
@Service
public class ContextBuilderService {

    public String buildContext(String query, List<Document> docs, int maxTokens) {
        StringBuilder context = new StringBuilder();
        int usedTokens = 0;
        int avgTokensPerChar = 2; // 中文约每字1-2 token

        for (int i = 0; i < docs.size(); i++) {
            String docText = String.format("[文档%d] %s", i + 1, docs.get(i).getContent());
            int docTokens = docText.length() / avgTokensPerChar;

            if (usedTokens + docTokens > maxTokens) break;

            context.append(docText).append("\n\n");
            usedTokens += docTokens;
        }
        return context.toString();
    }
}
```

## 参考资料

- [RAG 原论文](https://arxiv.org/abs/2005.11401)
- [Lost in the Middle 研究](https://arxiv.org/abs/2307.03172)
- [LangChain Context Compression](https://python.langchain.com/docs/modules/data_connection/retrievers/contextual_compression/)

---

## 补充：高级上下文工程技巧

### HyDE：假设文档嵌入

```python
from openai import OpenAI
import numpy as np

client = OpenAI()

def hyde_retrieval(query: str, documents: list, top_k: int = 3) -> list:
    """HyDE: 先生成假设答案，再用假设答案检索"""
    # 步骤1：生成假设答案文档
    hypothetical_response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"请写一段包含以下问题答案的文档片段（约100字）：{query}"
        }],
        temperature=0.0,
        max_tokens=200
    )
    hypothetical_doc = hypothetical_response.choices[0].message.content

    # 步骤2：用假设文档的 Embedding 检索
    hypothesis_emb = np.array(
        client.embeddings.create(
            model="text-embedding-3-small",
            input=hypothetical_doc
        ).data[0].embedding
    )

    # 步骤3：与文档库比较
    doc_embeddings = [
        np.array(client.embeddings.create(
            model="text-embedding-3-small",
            input=doc["content"]
        ).data[0].embedding)
        for doc in documents
    ]

    scores = [
        np.dot(hypothesis_emb, doc_emb) /
        (np.linalg.norm(hypothesis_emb) * np.linalg.norm(doc_emb))
        for doc_emb in doc_embeddings
    ]

    ranked = sorted(
        zip(documents, scores),
        key=lambda x: x[1],
        reverse=True
    )
    return [{"doc": doc, "score": score} for doc, score in ranked[:top_k]]

# 上下文精排：基于相关性重新排序
def rerank_context(query: str, docs: list) -> list:
    """使用 LLM 对检索结果重排序"""
    if len(docs) <= 1:
        return docs

    docs_text = "\n".join([
        f"[文档{i+1}] {doc['content'][:200]}"
        for i, doc in enumerate(docs)
    ])

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"""将以下文档按与问题的相关性从高到低排序。
只返回文档编号序列，如：3,1,2

问题：{query}

{docs_text}

排序（只返回数字序列）："""
        }],
        temperature=0.0,
        max_tokens=50
    )

    order_str = response.choices[0].message.content.strip()
    order = [int(x.strip()) - 1 for x in order_str.split(",") if x.strip().isdigit()]
    return [docs[i] for i in order if i < len(docs)]
```
