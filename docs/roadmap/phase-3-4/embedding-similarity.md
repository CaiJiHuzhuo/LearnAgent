# Embedding 与相似度

## 目标与重要性

Embedding 是将文本转换为向量的技术，是 RAG 系统的基础。理解 Embedding 的工作原理和相似度计算方法，能帮助你选择合适的模型和优化检索质量。

## 核心概念清单

- Embedding 的本质（语义向量空间）
- 主流 Embedding 模型（OpenAI、BGE、E5）
- 余弦相似度 vs 点积 vs 欧氏距离
- 双语 Embedding（中英文）
- Embedding 的归一化

## 学习路径

### 入门（第1层）

```python
import numpy as np
from openai import OpenAI

client = OpenAI()

def get_embedding(text: str, model: str = "text-embedding-3-small") -> list[float]:
    """获取文本的 Embedding 向量"""
    response = client.embeddings.create(
        model=model,
        input=text,
        encoding_format="float"
    )
    return response.data[0].embedding

def cosine_similarity(vec1: list, vec2: list) -> float:
    """计算余弦相似度"""
    v1, v2 = np.array(vec1), np.array(vec2)
    return np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))

# 演示语义相似度
texts = [
    "支付失败，余额不足",
    "账户余额不够支付",          # 语义相似
    "退款申请已提交",             # 不相关
    "payment failed, insufficient balance",  # 英文语义相同
]

reference = texts[0]
ref_embedding = get_embedding(reference)

print(f"参考文本：{reference}\n")
for text in texts[1:]:
    embedding = get_embedding(text)
    sim = cosine_similarity(ref_embedding, embedding)
    print(f"相似度 {sim:.3f}: {text}")
```

### 进阶（第2层）

```python
# 使用开源 Embedding 模型（本地运行，无需 API）
from sentence_transformers import SentenceTransformer
import torch

# BGE 模型：专为中文设计的高质量 Embedding
class BGEEmbedder:
    def __init__(self, model_name: str = "BAAI/bge-small-zh-v1.5"):
        self.model = SentenceTransformer(model_name)
        self.device = "cuda" if torch.cuda.is_available() else "cpu"
        self.model.to(self.device)

    def encode(self, texts: list[str], batch_size: int = 32) -> np.ndarray:
        """批量编码文本"""
        # BGE 模型需要在文本前加 instruction
        instruction = "为这个句子生成表示以用于检索相关文章："
        texts_with_instruction = [instruction + t for t in texts]

        embeddings = self.model.encode(
            texts_with_instruction,
            batch_size=batch_size,
            show_progress_bar=True,
            normalize_embeddings=True  # 归一化后可直接用点积计算相似度
        )
        return embeddings

    def similarity(self, text1: str, text2: str) -> float:
        embs = self.encode([text1, text2])
        return float(np.dot(embs[0], embs[1]))

# 对比不同 Embedding 模型
def compare_embedding_models(query: str, docs: list[str]) -> dict:
    models = {
        "openai-small": lambda texts: [get_embedding(t, "text-embedding-3-small") for t in texts],
        "openai-large": lambda texts: [get_embedding(t, "text-embedding-3-large") for t in texts],
    }

    results = {}
    for model_name, embed_fn in models.items():
        query_emb = np.array(embed_fn([query])[0])
        doc_embs = np.array(embed_fn(docs))

        # 归一化后点积等于余弦相似度
        query_norm = query_emb / np.linalg.norm(query_emb)
        doc_norms = doc_embs / np.linalg.norm(doc_embs, axis=1, keepdims=True)
        scores = np.dot(doc_norms, query_norm)

        results[model_name] = sorted(
            zip(docs, scores.tolist()),
            key=lambda x: x[1],
            reverse=True
        )
    return results
```

### 深入（第3层）

```python
# Embedding 的维度压缩（Matryoshka Representation Learning）
# OpenAI text-embedding-3 支持降维

def get_embedding_with_dimensions(
    text: str,
    dimensions: int = 256,  # 支持 256, 512, 1536, 3072
    model: str = "text-embedding-3-small"
) -> list[float]:
    """获取指定维度的 Embedding（节省存储和计算）"""
    response = client.embeddings.create(
        model=model,
        input=text,
        dimensions=dimensions,  # 截断维度
        encoding_format="float"
    )
    return response.data[0].embedding

# 对比不同维度的效果 vs 存储成本
dimension_comparison = {}
for dim in [256, 512, 1536]:
    emb = get_embedding_with_dimensions("支付失败排查", dimensions=dim)
    dimension_comparison[dim] = {
        "dimensions": len(emb),
        "storage_bytes": len(emb) * 4,  # float32 = 4 bytes
        "storage_kb": len(emb) * 4 / 1024,
    }

for dim, info in dimension_comparison.items():
    print(f"维度 {dim}: 存储 {info['storage_kb']:.1f} KB/向量")

# 双塔模型：查询和文档使用不同的 Embedding（更精准）
class AsymmetricEmbedder:
    """查询和文档使用不同的 Embedding 指令"""
    def __init__(self):
        self.model = SentenceTransformer("BAAI/bge-large-zh-v1.5")

    def encode_query(self, query: str) -> np.ndarray:
        instruction = "为这个问题生成查询向量，用于检索相关文档："
        return self.model.encode(
            [instruction + query],
            normalize_embeddings=True
        )[0]

    def encode_document(self, doc: str) -> np.ndarray:
        # 文档不需要 instruction
        return self.model.encode(
            [doc],
            normalize_embeddings=True
        )[0]
```

## 最小可执行练习

```python
# 构建一个简单的语义搜索系统
class SimpleSemanticSearch:
    def __init__(self):
        self.client = OpenAI()
        self.documents = []
        self.embeddings = []

    def add_documents(self, docs: list[str]):
        new_embeddings = [get_embedding(d) for d in docs]
        self.documents.extend(docs)
        self.embeddings.extend(new_embeddings)

    def search(self, query: str, top_k: int = 3) -> list[dict]:
        query_emb = np.array(get_embedding(query))
        doc_embs = np.array(self.embeddings)

        # 计算所有相似度
        scores = np.dot(doc_embs, query_emb) / (
            np.linalg.norm(doc_embs, axis=1) * np.linalg.norm(query_emb)
        )

        # 返回 top_k
        top_indices = np.argsort(scores)[::-1][:top_k]
        return [
            {"doc": self.documents[i], "score": float(scores[i])}
            for i in top_indices
        ]

# 测试
search = SimpleSemanticSearch()
search.add_documents([
    "支付失败的常见原因包括余额不足、卡片过期等",
    "退款流程：提交申请后3-7个工作日到账",
    "风控拦截的处理方式：联系客服验证身份",
    "账户被冻结的原因和解冻步骤",
])

results = search.search("我的支付为什么失败了")
for r in results:
    print(f"相似度 {r['score']:.3f}: {r['doc'][:50]}")
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 相似度都很高（>0.9）| 文档过于相似 | 检查文档多样性，优化分块策略 |
| 相似度都很低（<0.3）| 查询与文档语言风格不匹配 | 使用双语模型或查询改写 |
| 批量编码 OOM | 批量太大 | 减小 batch_size |
| 中文效果差 | 使用了英文专用模型 | 使用 BGE、M3E 等中文模型 |

## Java/Spring Boot 对接要点

```java
@Service
public class EmbeddingService {

    private final OpenAiEmbeddingModel embeddingModel;

    public float[] embed(String text) {
        EmbeddingResponse response = embeddingModel.embedForResponse(List.of(text));
        return response.getResults().get(0).getOutput().getVector();
    }

    public double cosineSimilarity(float[] v1, float[] v2) {
        double dot = 0, norm1 = 0, norm2 = 0;
        for (int i = 0; i < v1.length; i++) {
            dot += v1[i] * v2[i];
            norm1 += v1[i] * v1[i];
            norm2 += v2[i] * v2[i];
        }
        return dot / (Math.sqrt(norm1) * Math.sqrt(norm2));
    }
}
```

## 参考资料

- [OpenAI Embeddings 文档](https://platform.openai.com/docs/guides/embeddings)
- [BGE 模型](https://huggingface.co/BAAI/bge-large-zh-v1.5)
- [Massive Text Embedding Benchmark (MTEB)](https://huggingface.co/spaces/mteb/leaderboard)

---

## 补充：Embedding 工程实践

### 批量 Embedding 与缓存

```python
import hashlib
import json
from pathlib import Path
from openai import OpenAI
import numpy as np

client = OpenAI()

class CachedEmbeddingService:
    """带磁盘缓存的 Embedding 服务"""

    def __init__(self, cache_dir: str = ".embedding_cache"):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(exist_ok=True)
        self.model = "text-embedding-3-small"

    def _cache_key(self, text: str) -> str:
        return hashlib.md5(f"{self.model}:{text}".encode()).hexdigest()

    def _cache_path(self, key: str) -> Path:
        return self.cache_dir / f"{key}.json"

    def get_embedding(self, text: str) -> list:
        """获取 Embedding，优先使用缓存"""
        key = self._cache_key(text)
        cache_file = self._cache_path(key)

        if cache_file.exists():
            return json.loads(cache_file.read_text())["embedding"]

        response = client.embeddings.create(model=self.model, input=text)
        embedding = response.data[0].embedding

        cache_file.write_text(json.dumps({"text": text[:100], "embedding": embedding}))
        return embedding

    def batch_embed(self, texts: list, batch_size: int = 100) -> list:
        """批量获取 Embedding（复用缓存，减少 API 调用）"""
        all_embeddings = []
        to_embed = []
        to_embed_indices = []

        # 检查哪些需要实际调用 API
        for i, text in enumerate(texts):
            key = self._cache_key(text)
            cache_file = self._cache_path(key)
            if cache_file.exists():
                all_embeddings.append((i, json.loads(cache_file.read_text())["embedding"]))
            else:
                to_embed.append(text)
                to_embed_indices.append(i)

        # 批量调用 API
        for batch_start in range(0, len(to_embed), batch_size):
            batch = to_embed[batch_start:batch_start + batch_size]
            response = client.embeddings.create(model=self.model, input=batch)
            for j, emb_data in enumerate(response.data):
                idx = to_embed_indices[batch_start + j]
                embedding = emb_data.embedding
                all_embeddings.append((idx, embedding))
                # 写入缓存
                key = self._cache_key(batch[j])
                self._cache_path(key).write_text(
                    json.dumps({"text": batch[j][:100], "embedding": embedding})
                )

        # 按原始顺序返回
        all_embeddings.sort(key=lambda x: x[0])
        return [emb for _, emb in all_embeddings]

    def similarity_matrix(self, texts: list) -> np.ndarray:
        """计算文本列表的相似度矩阵"""
        embeddings = np.array(self.batch_embed(texts))
        # 归一化
        norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
        normalized = embeddings / norms
        # 相似度矩阵
        return np.dot(normalized, normalized.T)

# 使用示例
service = CachedEmbeddingService()

# 支付相关文档
docs = [
    "账户余额不足导致支付失败",
    "银行卡被冻结无法支付",
    "支付渠道临时维护",
    "用户输入了错误的支付密码",
    "风控系统拦截了可疑交易",
]

# 查询
query = "余额问题导致支付不成功"
query_emb = np.array(service.get_embedding(query))
doc_embs = np.array(service.batch_embed(docs))

# 计算相似度
scores = np.dot(doc_embs, query_emb) / (
    np.linalg.norm(doc_embs, axis=1) * np.linalg.norm(query_emb)
)

print("查询:", query)
print("\n相关文档排序:")
for idx in np.argsort(scores)[::-1]:
    print(f"  {scores[idx]:.3f}: {docs[idx]}")
```

### Embedding 质量评估

```python
def evaluate_embedding_quality(
    queries: list,
    relevant_docs: dict,  # {query: [relevant_doc_indices]}
    all_docs: list,
    embedding_service
) -> dict:
    """评估 Embedding 检索质量"""
    hits_at_1 = 0
    hits_at_3 = 0
    mrr_sum = 0

    doc_embs = np.array(embedding_service.batch_embed(all_docs))

    for query, relevant_indices in zip(queries, relevant_docs.values()):
        query_emb = np.array(embedding_service.get_embedding(query))
        scores = np.dot(doc_embs, query_emb) / (
            np.linalg.norm(doc_embs, axis=1) * np.linalg.norm(query_emb)
        )
        ranked = np.argsort(scores)[::-1]

        # Hit@1
        if ranked[0] in relevant_indices:
            hits_at_1 += 1

        # Hit@3
        if any(r in relevant_indices for r in ranked[:3]):
            hits_at_3 += 1

        # MRR
        for rank, idx in enumerate(ranked):
            if idx in relevant_indices:
                mrr_sum += 1 / (rank + 1)
                break

    n = len(queries)
    return {
        "hit_at_1": hits_at_1 / n,
        "hit_at_3": hits_at_3 / n,
        "mrr": mrr_sum / n,
    }
```
