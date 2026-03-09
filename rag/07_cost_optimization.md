## Cost Optimization & Semantic Caching

---

### Tại sao cost control là bắt buộc với 20K users

Ước tính chi phí KHÔNG có optimization:

```
20K nhân viên, 5% active daily = 1000 users
Mỗi user trung bình 3 queries/ngày = 3000 queries/ngày

Per query cost (không cache, không optimize):
  - Embedding: ~500 tokens × $0.00002/token = $0.01
  - LLM generation: ~3000 tokens × $0.01/1K token (GPT-4o) = $0.03
  - Reranker: ~$0.005
  - Faithfulness check: ~$0.01
  - Total per query: ~$0.06

Daily cost: 3000 × $0.06 = $180/ngày
Monthly: ~$5,400/tháng
Annual: ~$65,000/năm (chỉ LLM API, chưa tính infra)
```

**Với semantic caching + token optimization → giảm 40-60% = tiết kiệm ~$26K-39K/năm.**

---

### Layer 1: Semantic Caching

#### Kiến trúc

```
┌─────────────────────────────────────────────────────────┐
│                     User Query                           │
│                         │                                │
│                 ┌───────▼───────┐                        │
│                 │ Query Embedder│                        │
│                 └───────┬───────┘                        │
│                         │                                │
│              ┌──────────▼──────────┐                     │
│              │   Semantic Cache    │                     │
│              │                     │                     │
│              │ Key: (embedding,    │                     │
│              │       user_role)    │                     │
│              │                     │                     │
│              │ cosine_sim > 0.92?  │                     │
│              └──────┬──────┬───────┘                     │
│                     │      │                             │
│              HIT    │      │  MISS                       │
│                     │      │                             │
│           ┌─────────▼┐  ┌──▼────────────┐               │
│           │ Return   │  │ RAG Pipeline  │               │
│           │ cached   │  │ (full)        │               │
│           │ response │  │               │               │
│           │          │  │ Store result  │               │
│           │ Latency: │  │ to cache      │               │
│           │ ~100ms   │  │ + TTL         │               │
│           └──────────┘  └───────────────┘               │
└─────────────────────────────────────────────────────────┘
```

#### Implementation

```python
import numpy as np
from datetime import datetime, timedelta

class SemanticCache:
    def __init__(self, vector_store, similarity_threshold=0.92):
        self.vector_store = vector_store  # Dùng collection riêng trong Qdrant
        self.threshold = similarity_threshold
    
    def get(self, query: str, user_role: str) -> CacheResult | None:
        """Tìm cached response cho query tương tự."""
        query_embedding = embed_model.encode(query)
        
        results = self.vector_store.search(
            collection="semantic_cache",
            query_vector=query_embedding,
            query_filter=Filter(
                must=[
                    # RBAC: chỉ match cache entries cùng role
                    FieldCondition(key="user_role", match=MatchValue(value=user_role)),
                    # TTL: chỉ entries chưa expire
                    FieldCondition(
                        key="expires_at", 
                        range=Range(gte=datetime.now().isoformat())
                    ),
                ]
            ),
            limit=1,
        )
        
        if results and results[0].score >= self.threshold:
            cache_hit_counter.inc()  # Prometheus metric
            return CacheResult(
                response=results[0].payload["response"],
                original_query=results[0].payload["original_query"],
                similarity=results[0].score,
                cached_at=results[0].payload["cached_at"],
            )
        
        cache_miss_counter.inc()
        return None
    
    def put(self, query: str, response: str, user_role: str, route: str):
        """Cache response mới."""
        query_embedding = embed_model.encode(query)
        ttl = self._get_ttl(route)
        
        self.vector_store.upsert(
            collection="semantic_cache",
            id=generate_id(),
            vector=query_embedding,
            payload={
                "original_query": query,
                "response": response,
                "user_role": user_role,
                "route": route,
                "cached_at": datetime.now().isoformat(),
                "expires_at": (datetime.now() + ttl).isoformat(),
            },
        )
    
    def _get_ttl(self, route: str) -> timedelta:
        """TTL tùy loại data."""
        return {
            "sql_only": timedelta(hours=2),     # DB data thay đổi nhanh
            "doc_only": timedelta(hours=48),     # Tài liệu ít thay đổi
            "hybrid": timedelta(hours=4),        # Compromise
        }.get(route, timedelta(hours=12))
    
    def invalidate_by_doc(self, doc_id: str):
        """Xóa cache entries liên quan khi document thay đổi."""
        self.vector_store.delete(
            collection="semantic_cache",
            filter=FieldCondition(
                key="source_doc_ids", 
                match=MatchAny(any=[doc_id])
            ),
        )
    
    def cleanup_expired(self):
        """Cron job: xóa entries đã expire."""
        self.vector_store.delete(
            collection="semantic_cache",
            filter=FieldCondition(
                key="expires_at",
                range=Range(lt=datetime.now().isoformat()),
            ),
        )
```

#### Cache Design Decisions

| Decision | Choice | Lý do |
|----------|--------|-------|
| **Similarity threshold** | 0.92 | Cao để tránh false positive. 0.85 quá thấp → trả sai answer |
| **Cache key includes role** | Yes | Cùng query, khác role → có thể khác answer (RBAC) |
| **TTL by route type** | Yes | SQL data thay đổi nhanh hơn doc |
| **Store in Qdrant** | Yes | Tận dụng infra đã có, không thêm Redis |
| **Cache invalidation** | On doc update | Sync với data freshness pipeline |

#### Expected Impact

```
Trước caching:
  3000 queries/ngày × $0.06 = $180/ngày

Sau caching (estimate 35% cache hit rate):
  1050 cache hits × $0.001 (chỉ embedding) = $1.05
  1950 cache misses × $0.06 = $117
  Total: $118/ngày
  
Saving: $62/ngày = $1,860/tháng = $22,300/năm
```

---

### Layer 2: Token Optimization

#### 2A — Column Pruning (SQL Track)

Xem chi tiết `02_pipeline.md` Phase 3D. Impact:

```
Trước: 200 columns × ~20 tokens/column = 4000 tokens schema
Sau prune: 15-20 columns = 300-400 tokens
Saving: ~90% tokens cho schema portion
```

#### 2B — Smart Context Window Management

```python
def optimize_context(retrieved_chunks: list, max_tokens: int = 4000) -> list:
    """Chọn chunks tối ưu cho context window."""
    
    # 1. Deduplicate: chunks có overlap content > 70% → giữ 1
    deduped = deduplicate_chunks(retrieved_chunks, overlap_threshold=0.7)
    
    # 2. Budget allocation
    total_tokens = sum(count_tokens(c.text) for c in deduped)
    
    if total_tokens <= max_tokens:
        return deduped  # Fit hết → dùng hết
    
    # 3. Truncate by relevance score — giữ chunks quan trọng nhất
    sorted_by_score = sorted(deduped, key=lambda c: c.score, reverse=True)
    
    selected = []
    current_tokens = 0
    for chunk in sorted_by_score:
        chunk_tokens = count_tokens(chunk.text)
        if current_tokens + chunk_tokens <= max_tokens:
            selected.append(chunk)
            current_tokens += chunk_tokens
        else:
            # Truncate chunk cuối nếu vẫn còn budget
            remaining = max_tokens - current_tokens
            if remaining > 100:  # Ít nhất 100 tokens mới đáng
                selected.append(truncate_chunk(chunk, remaining))
            break
    
    return selected
```

#### 2C — Model Tiering

Không phải mọi query đều cần GPT-4o. Dùng model nhẹ cho tasks đơn giản:

```python
MODEL_TIERS = {
    # Task → Model → Cost/1K tokens
    "routing":          ("gpt-4o-mini", 0.00015),   # Rẻ nhất
    "column_pruning":   ("gpt-4o-mini", 0.00015),   
    "query_rewriting":  ("gpt-4o-mini", 0.00015),   
    "sql_generation":   ("gpt-4o",      0.005),     # Cần accuracy cao
    "synthesis":        ("gpt-4o",      0.005),     # Cần chất lượng
    "faithfulness":     ("gpt-4o-mini", 0.00015),   # Chỉ yes/no check
}

def get_model(task: str) -> str:
    return MODEL_TIERS[task][0]

# Impact:
# Routing + pruning + rewriting + faithfulness dùng mini = ~$0.001/query
# Chỉ SQL gen + synthesis dùng full model = ~$0.02/query
# So với dùng GPT-4o cho tất cả = ~$0.04/query
# → Saving ~50% production LLM cost
```

#### 2D — Prompt Optimization

```python
# TRƯỚC: Verbose prompt (800 tokens)
BEFORE = """
Bạn là một trợ lý AI thông minh hoạt động trong một doanh nghiệp lớn. 
Nhiệm vụ của bạn là phân tích dữ liệu được cung cấp bên dưới và đưa ra 
câu trả lời chính xác, chi tiết, có trích dẫn nguồn cho mỗi điểm trong 
câu trả lời. Hãy đảm bảo rằng bạn không bịa đặt thông tin và chỉ trả lời 
dựa trên dữ liệu thực tế...
[nhiều instruction nữa]
"""

# SAU: Concise prompt (200 tokens) — cùng quality
AFTER = """
Data analyst nội bộ. Trả lời dựa trên dữ liệu cung cấp.
Mỗi điểm → ghi nguồn (DB: bảng | Doc: tài_liệu, trang).
Không đủ thông tin → nói không biết.
Cuối: 2-3 câu hỏi kiểm tra thêm.
"""

# Saving: 600 tokens/query × 3000 queries/ngày = 1.8M tokens/ngày
# = ~$9/ngày saved chỉ từ prompt optimization
```

---

### Layer 3: Infrastructure Cost

#### Self-host vs Cloud

```
Component: Vector Store (Qdrant)
  Self-host (Docker):  ~$50/tháng (VM 4CPU, 16GB RAM)
  Cloud (Qdrant Cloud): ~$200-500/tháng
  → Self-host saving: $150-450/tháng

Component: LLM
  Azure OpenAI: Per-token pricing (production stable)
  Self-host (Ollama + Llama): $200-500/tháng cho GPU server
  → Self-host chỉ có ý nghĩa nếu >10K queries/ngày
  
Component: Embedding
  OpenAI embedding API: ~$0.02/1M tokens
  Self-host bge-m3: $100/tháng cho GPU server
  → Self-host có ý nghĩa khi >50M tokens/tháng

Recommendation cho 3000 queries/ngày:
  ✅ Self-host Qdrant (dễ, rẻ)
  ✅ Cloud LLM (Azure OpenAI — ổn định, dễ scale)
  ⚠️ Self-host embedding chỉ nếu có GPU rỗi
```

#### Batch Processing cho Non-urgent Queries

```python
# Không phải mọi query cần real-time
# Reports, summaries → batch processing giảm cost

class QueryPrioritizer:
    def prioritize(self, query: str, user_context: dict) -> str:
        if self._is_report_request(query):
            return "batch"      # Queue, process mỗi 15 phút
        elif self._is_urgent(query, user_context):
            return "realtime"   # Process ngay
        else:
            return "standard"   # Process ngay nhưng cost-optimized

    # Batch queries có thể aggregate → 1 LLM call cho nhiều queries
```

---

### Budget Template

```
Monthly Budget Breakdown (estimate cho 3000 queries/ngày):

┌─────────────────────────────────────────────────────┐
│                  MONTHLY BUDGET                      │
│                                                      │
│  LLM API (Azure OpenAI)                              │
│    Generation (GPT-4o):        $900-1,200            │
│    Mini tasks (GPT-4o-mini):   $50-100               │
│    Embedding:                  $30-50                 │
│    Subtotal LLM:               $980-1,350            │
│                                                      │
│  Infrastructure                                      │
│    Vector Store (Qdrant VM):   $50-100               │
│    App Server:                 $100-200              │
│    LangFuse (observability):   $50                   │
│    Monitoring (Prometheus):    $50                   │
│    Subtotal Infra:             $250-400              │
│                                                      │
│  ─────────────────────────────────                   │
│  TOTAL MONTHLY:                $1,230-1,750          │
│  TOTAL ANNUAL:                 $14,760-21,000        │
│                                                      │
│  vs. KHÔNG optimize:           $65,000/năm           │
│  SAVING:                       ~$43,000-50,000/năm   │
└─────────────────────────────────────────────────────┘
```

---

### Cost Monitoring Dashboard

```python
# Track cost per query, per department, per route

cost_tracker = {
    "per_query": Histogram(
        'rag_query_cost_usd',
        'Cost per query in USD',
        ['route', 'department', 'cache_hit'],
        buckets=[0.001, 0.005, 0.01, 0.03, 0.05, 0.1]
    ),
    
    "daily_total": Counter(
        'rag_daily_cost_usd',
        'Total daily cost',
        ['component']  # llm / embedding / infra
    ),
    
    "budget_remaining": Gauge(
        'rag_budget_remaining_usd',
        'Remaining daily budget'
    ),
}

# Alert khi cost vượt ngưỡng
DAILY_BUDGET = 60  # USD
if daily_cost > DAILY_BUDGET * 0.8:
    alert("COST_WARNING", f"Daily cost at {daily_cost/DAILY_BUDGET*100:.0f}% of budget")
if daily_cost > DAILY_BUDGET:
    alert("COST_CRITICAL", f"Daily cost exceeded budget: ${daily_cost:.2f}")
    # Option: switch to cheaper model, increase cache aggressiveness
```

---

### Checklist triển khai

```
Phase 2 — Basic Cost Control (1 tuần):
☐ Semantic caching implementation
☐ Model tiering (mini cho routing + aux tasks)
☐ Prompt optimization (trim verbose prompts)
☐ Cost tracking metrics

Phase 3 — Advanced Optimization (2 tuần):
☐ Column pruning cho SQL track
☐ Smart context window management
☐ Cache TTL tuning based on data
☐ Cost dashboard trong Grafana
☐ Daily budget alerts

Ongoing:
☐ Monthly cost review
☐ Evaluate new cheaper models khi available
☐ Cache hit rate optimization
☐ Token usage audit
```

---

### Liên quan

- Pipeline: [02_pipeline.md](./02_pipeline.md) — caching integrates vào Phase 2
- Observability: [04_observability.md](./04_observability.md) — cost monitoring
- Data lifecycle: [05_data_lifecycle.md](./05_data_lifecycle.md) — cache invalidation khi data update
