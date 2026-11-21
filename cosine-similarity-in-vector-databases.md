# Cosine Similarity in Vector Databases

## 1. TL;DR

Cosine similarity measures how close two vectors are by comparing their **angle**, not their length. In vector databases, you embed both documents and queries into vectors, then retrieve the vectors with the **highest cosine similarity** (smallest angle) to the query vector. It works well for text embeddings because direction tends to encode meaning, while magnitude can vary for reasons unrelated to semantics.

> **Scope assumption:** This note focuses on cosine similarity for dense embedding retrieval (semantic search / RAG). It does not cover learning embeddings.

## 2. Background & Motivation

### Why do we need a similarity metric?

Once text is converted into vectors, a vector database must answer:

> “Which stored vectors are most like this query vector?”

That requires a function that maps a pair of vectors to a score. The score should correlate with semantic similarity.

### Why cosine specifically?

Embedding models often produce vectors where:

- Similar meanings point in similar directions.
- Vector length (magnitude) can vary due to sentence length, token frequency, or model internals.

Cosine similarity ignores magnitude and focuses on direction, making it robust for semantic matching.

## 3. Core Concepts

| Term | Meaning |
|---|---|
| Vector | A point in $\mathbb{R}^d$ (array of floats) representing meaning. |
| Magnitude / norm | Vector length: $\lVert a \rVert = \sqrt{\sum a_i^2}$. |
| Dot product | $a \cdot b = \sum a_i b_i$; mixes magnitude and alignment. |
| Angle | The geometric angle $\theta$ between vectors. |
| Cosine similarity | $\cos(\theta)$; ranges from -1 to 1 (often 0..1 for embeddings). |
| Distance vs similarity | DBs often minimize a distance (smaller is better) rather than maximize a similarity. |

### Intuition from first principles

If two vectors point in the same direction, they represent similar concepts. If they’re orthogonal (90°), they’re unrelated. If they point in opposite directions, they’re opposites.

### Analogy (hardest idea)

Think of each vector as an arrow:

- The **direction** is the meaning.
- The **length** is how “loud” or “confident” the model produced the embedding.

Cosine similarity asks: *Do the arrows point the same way?* It doesn’t care how long they are.

## 4. Theory (Deep Dive)

### 4.1 Formula

For vectors $a, b \in \mathbb{R}^d$:

```math
\text{cosine}(a, b) = \frac{a \cdot b}{\lVert a \rVert \, \lVert b \rVert}
```

where

```math
a \cdot b = \sum_{i=1}^d a_i b_i \quad\text{and}\quad
\lVert a \rVert = \sqrt{\sum_{i=1}^d a_i^2}
```

### 4.2 Geometric meaning

The dot product can be written as:

```math
a \cdot b = \lVert a \rVert \, \lVert b \rVert \, \cos(\theta)
```

Rearranging gives the cosine formula. So cosine similarity is literally *the cosine of the angle* between the vectors.

- $\cos(0°)=1$ → identical direction → very similar
- $\cos(90°)=0$ → orthogonal → unrelated
- $\cos(180°)=-1$ → opposite direction

Most modern text embeddings (especially after normalization) tend to have similarities in a narrower band (for example 0.2–0.9), not spanning the full -1..1.

### 4.3 Normalization and why dot product often replaces cosine

If you **L2-normalize** vectors:

```math
\hat{a} = \frac{a}{\lVert a \rVert}, \quad \hat{b} = \frac{b}{\lVert b \rVert}
```

then:

```math
\text{cosine}(a, b) = \hat{a} \cdot \hat{b}
```

So cosine similarity reduces to a dot product of normalized vectors, which is computationally convenient.

### 4.4 Cosine distance

Databases frequently work with a distance to minimize instead of a similarity to maximize.

A common definition is:

```math
\text{cosine\_distance}(a, b) = 1 - \text{cosine}(a, b)
```

- Smaller distance means closer.
- Minimizing $1 - \cos$ is equivalent to maximizing $\cos$.

### 4.5 Relationship to Euclidean distance (important invariant)

If vectors are L2-normalized ($\lVert a \rVert = \lVert b \rVert = 1$):

```math
\lVert a - b \rVert_2^2 = 2 - 2(a \cdot b) = 2 - 2\,\cos(a, b)
```

That means on normalized vectors:

- Maximizing cosine similarity
- Maximizing dot product
- Minimizing Euclidean distance

all produce the **same nearest-neighbor ranking**.

This is why many systems normalize vectors once, then use whatever operator/index is best supported.

### 4.6 How a vector DB uses cosine to find “closest” vectors

At query time:

1. **Embed the query** → query vector $q$.
2. (Often) **Normalize** $q$ and stored vectors.
3. Use an ANN index (HNSW/IVF) to search for candidates with high similarity.
4. Return the top-k by cosine similarity.

#### Brute force scoring

If you had no index:

- Compute $\cos(q, v_i)$ for every stored vector $v_i$.
- Sort by score.

Time: $O(N \cdot d)$ per query, too slow at scale.

#### With ANN indexes

ANN indexes avoid scoring every vector.

- **HNSW:** navigates a graph toward vectors that increase similarity to $q$.
- **IVF:** probes only the most relevant clusters.

They still use cosine (or its equivalent) as the score function during traversal.

### 4.7 Performance characteristics

- Computing cosine similarity is essentially a dot product plus two norms.
- If vectors are normalized and stored normalized, per-comparison cost is a single dot product: $O(d)$.
- Index search cost is sublinear (often close to $O(\log N)$ for HNSW-like behavior), but depends heavily on tuning and filters.

### 4.8 Practical guarantees / trade-offs

- Cosine similarity is **scale-invariant**: scaling a vector by $\alpha>0$ does not change cosine.
- If the embedding model encodes meaning partly in magnitude (some do), cosine may discard useful signal. Dot product can sometimes work better, depending on the model.
- ANN retrieval is approximate: the "closest" vectors returned might not be the true nearest neighbors unless you tune for high recall.

## 5. Practical Usage

### 5.1 Minimal example (NumPy)

```python
import numpy as np


def l2_normalize(x: np.ndarray) -> np.ndarray:
    return x / np.clip(np.linalg.norm(x), 1e-12, None)


def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    a = l2_normalize(a)
    b = l2_normalize(b)
    return float(a @ b)


a = np.array([1.0, 2.0, 0.0])
b = np.array([2.0, 4.0, 0.0])  # same direction as a
c = np.array([0.0, 0.0, 1.0])  # orthogonal

print(cosine_similarity(a, b))  # ~1.0
print(cosine_similarity(a, c))  # ~0.0
```

### 5.2 Realistic example: top-k cosine search

```python
import numpy as np


def normalize_rows(X: np.ndarray) -> np.ndarray:
    norms = np.linalg.norm(X, axis=1, keepdims=True)
    return X / np.clip(norms, 1e-12, None)


def topk_cosine(doc_vecs: np.ndarray, query_vec: np.ndarray, k: int = 5):
    D = normalize_rows(doc_vecs)
    q = query_vec / np.clip(np.linalg.norm(query_vec), 1e-12, None)
    scores = D @ q  # cosine similarities
    idx = np.argpartition(-scores, k - 1)[:k]
    idx = idx[np.argsort(-scores[idx])]
    return [(int(i), float(scores[i])) for i in idx]


rng = np.random.default_rng(0)
docs = rng.normal(size=(10000, 384)).astype(np.float32)
q = rng.normal(size=(384,)).astype(np.float32)
print(topk_cosine(docs, q, k=3))
```

### 5.3 In actual vector DBs

- Some DBs ask you to specify `metric="cosine"`.
- Some DBs provide a **cosine distance** operator.
- Many DBs internally normalize vectors for cosine, or expect you to normalize.

**Best practice:** treat normalization as part of your embedding pipeline and make it consistent for both documents and queries.

## 6. Common Pitfalls & Best Practices

| Mistake | Why it happens | Fix |
|---|---|---|
| Not normalizing when using dot product for cosine-like behavior | Confusing dot product vs cosine | Normalize embeddings and queries, then dot = cosine. |
| Using cosine on a model trained for dot product | Metric mismatch | Follow the embedding model’s recommended metric. |
| Comparing vectors from different models | “Vectors are vectors” misconception | Re-embed with a single model per collection. |
| Assuming ANN returns exact nearest neighbors | ANN is approximate | Measure recall@k; tune `ef_search` / `nprobe`. |
| Mixing normalized and non-normalized vectors in the same collection | Inconsistent ingestion | Validate and enforce normalization at write time. |

## 7. Comparison with Alternatives

| Metric | When it’s good | Downsides |
|---|---|---|
| Cosine similarity | Meaning is encoded in direction; magnitude is noisy (common for text embeddings) | Discards magnitude signal if it contains useful info |
| Dot product (inner product) | If model training optimizes for dot product; sometimes better when magnitude matters | Sensitive to vector length; can be dominated by large norms |
| Euclidean (L2) | Geometry-friendly; equivalent to cosine on normalized vectors | On non-normalized vectors it mixes magnitude and direction |

## 8. Real-World Applications

- Semantic search and RAG (most common)
- Similar-issue / ticket deduplication
- Recommendation candidate retrieval
- Similarity-based clustering

## 9. Practice Questions

### 9.1 Conceptual (5 questions)

1. 🟢 Easy: What does cosine similarity measure geometrically?
2. 🟢 Easy: Why might cosine similarity be preferred over dot product for text embeddings?
3. 🟡 Medium: If you scale a vector by 10×, what happens to cosine similarity and dot product, and why?
4. 🟡 Medium: Explain why cosine similarity and Euclidean distance give the same ranking on normalized vectors.
5. 🔴 Hard: Give a scenario where dot product could outperform cosine similarity for retrieval.

### 9.2 Applied / Scenario-Based (4 questions)

6. 🟡 Medium: Your DB is configured for cosine, but you forgot to normalize vectors and queries. What symptoms might you see?
7. 🟡 Medium: After switching embedding models, search quality drops drastically even though cosine similarity still “works.” Why?
8. 🔴 Hard: You apply strict metadata filters. Why does post-filtering degrade results more for cosine/ANN search, and how do you mitigate it?
9. 🟡 Medium: You get many semantically correct results but missing exact code matches like `ERR-4471`. How do you fix it?

### 9.3 Coding / Hands-On (3 tasks)

10. 🟢 Easy: Implement cosine similarity for two vectors without using NumPy.
11. 🟡 Medium: Implement brute-force top-k cosine retrieval over 100k vectors and measure latency.
12. 🔴 Hard: For a sample of queries, compare ANN results vs brute-force results and compute recall@10.

### 9.4 Interview-Style (3 questions)

13. 🟡 Medium: “Explain cosine similarity and why it’s used in vector search.”
14. 🔴 Hard: “How would you decide between cosine and dot product for a new embedding model?”
15. 🔴 Hard: “Why do some systems normalize embeddings and then use L2 distance indexes instead of cosine?”

## 10. Answers & Explanations

<details>
<summary>Answer to Q1</summary>

Cosine similarity measures the **angle** between vectors. If the angle is small (vectors point in the same direction), cosine is close to 1 and the vectors are considered similar.

</details>

<details>
<summary>Answer to Q2</summary>

Cosine ignores magnitude. If embedding length varies due to sentence length or model quirks, cosine still compares meaning (direction) without being biased toward vectors with larger norms.

</details>

<details>
<summary>Answer to Q3</summary>

Cosine similarity stays the same because both the numerator ($a\cdot b$) and denominator ($\lVert a\rVert\lVert b\rVert$) scale by the same factor. Dot product increases by 10× because it scales directly with magnitude.

</details>

<details>
<summary>Answer to Q4</summary>

For normalized vectors, $\lVert a \rVert = \lVert b \rVert = 1$, and:

$\lVert a - b \rVert_2^2 = 2 - 2(a \cdot b) = 2 - 2\cos(a,b)$.

Since this is a monotonic transformation, ordering by smallest L2 distance is the same as ordering by largest cosine similarity.

</details>

<details>
<summary>Answer to Q5</summary>

If an embedding model encodes additional signal in the norm (for example confidence, specificity, or importance), dot product can exploit that because it increases when both vectors have larger magnitude and similar direction. Cosine would remove that signal.

A practical example: some recommendation embeddings use norm to represent popularity or user engagement; dot product can naturally boost those items.

</details>

<details>
<summary>Answer to Q6</summary>

You may see unstable or unintuitive results: long chunks (or vectors with larger norms) could dominate retrieval because their dot products become larger even if direction differs. Similarity scores may not be comparable across queries. Fix by normalizing consistently or using the metric recommended by the model.

</details>

<details>
<summary>Answer to Q7</summary>

Vectors from different embedding models live in different coordinate systems. Cosine similarity still produces a number, but it no longer correlates with semantic similarity because “direction” has a different meaning. Fix by re-embedding the whole corpus and switching query embeddings at the same time.

</details>

<details>
<summary>Answer to Q8</summary>

ANN search returns top-k candidates globally; post-filtering then removes those that don’t match the filter, leaving too few results. This gets worse when the filter is selective (small tenant, rare tag). Mitigate by using filtered ANN (filter during traversal), partitioning by tenant, or over-fetching and reranking.

</details>

<details>
<summary>Answer to Q9</summary>

Dense embeddings often don’t preserve exact matching for rare tokens like codes. Use hybrid search (BM25 + vector), extract the codes into metadata for exact filtering/boosting, or rerank candidates with a cross-encoder.

</details>

<details>
<summary>Answer to Q10</summary>

Compute dot product and norms manually:

1. $\text{dot} = \sum a_i b_i$
2. $\lVert a \rVert = \sqrt{\sum a_i^2}$ and $\lVert b \rVert = \sqrt{\sum b_i^2}$
3. return $\text{dot} / (\lVert a \rVert \lVert b \rVert)$, guarding against zero norms.

</details>

<details>
<summary>Answer to Q11</summary>

Use row-normalized document vectors, normalize the query, then compute scores by matrix-vector multiplication and select top-k with `argpartition`. Latency depends on CPU vectorization and cache. 100k×768 float32 dot products is typically a few milliseconds to tens of milliseconds in optimized NumPy/BLAS.

</details>

<details>
<summary>Answer to Q12</summary>

Run brute force for each query to get exact top-10. Run ANN to get approximate top-10. Compute recall@10 as the overlap ratio. Use this to tune HNSW `ef_search` or IVF `nprobe`.

</details>

<details>
<summary>Answer to Q13</summary>

Cosine similarity compares vectors by their angle. It’s used because embedding direction usually captures semantic meaning; cosine is scale-invariant, so it’s less sensitive to embedding magnitude differences.

</details>

<details>
<summary>Answer to Q14</summary>

Check the embedding model documentation and evaluate on your task:

- If the model expects normalized embeddings and uses cosine during training/evaluation, use cosine.
- If the model expects dot product, use dot product.
- Validate by measuring retrieval metrics (recall@k, nDCG) on a labeled dataset.

</details>

<details>
<summary>Answer to Q15</summary>

Because on normalized vectors, L2 distance and cosine similarity induce the same ranking. Some systems have more mature or faster L2 indexing support, or they reuse existing ANN implementations that are optimized for L2.

</details>

## 11. Summary / Cheat Sheet

- Cosine similarity measures **directional alignment**: $\cos(a,b) = \frac{a\cdot b}{\lVert a\rVert\lVert b\rVert}$.
- It is **scale invariant**: scaling vectors doesn’t change cosine.
- If vectors are normalized, **cosine = dot product**.
- On normalized vectors, cosine ranking equals L2 ranking.
- Vector DBs find nearest neighbors by maximizing cosine (or minimizing cosine distance), usually with ANN indexes.
- Always keep metric + normalization consistent between ingestion and query.

## 12. Further Reading

- HNSW paper: https://arxiv.org/abs/1603.09320
- Sentence-BERT paper: https://arxiv.org/abs/1908.10084
- FAISS: https://github.com/facebookresearch/faiss
- pgvector: https://github.com/pgvector/pgvector
