## Observability & Monitoring cho Enterprise RAG

---

### Tại sao validation ≠ observability

Validation (Phase 5 trong pipeline) là kiểm tra **từng query**: SQL có hợp lệ không, answer có hallucinate không.

Observability là nhìn **toàn hệ thống theo thời gian**: performance có degrading không, cost có vượt budget không, users có hài lòng không.

Với 20K users, không có observability = **lái xe ban đêm không đèn**.

---

### Stack đề xuất — Self-host, Open-source first

```
┌──────────────────────────────────────────────────────────────┐
│                   OBSERVABILITY STACK                          │
│                                                               │
│   ┌─────────────┐   ┌──────────────┐   ┌─────────────────┐  │
│   │  LangFuse   │   │  Prometheus  │   │    Grafana      │  │
│   │  (self-host)│   │              │   │                 │  │
│   │             │   │  System      │   │  Dashboards     │  │
│   │  LLM Traces │   │  metrics     │   │  Alerts         │  │
│   │  Scores     │   │  Latency     │   │  Reports        │  │
│   │  Cost track │   │  Error rates │   │                 │  │
│   └──────┬──────┘   └──────┬───────┘   └────────┬────────┘  │
│          │                 │                     │            │
│          └─────────────────┴─────────────────────┘            │
│                            │                                  │
│                     ┌──────▼───────┐                          │
│                     │   Alert      │                          │
│                     │   Manager    │ → Slack / Email / PagerDuty │
│                     └──────────────┘                          │
└──────────────────────────────────────────────────────────────┘
```

**Tại sao LangFuse?**
- Open-source, self-host được — quan trọng cho enterprise (data không rời hệ thống)
- Trace từng bước pipeline: retrieval → generation → validation
- Built-in scoring: faithfulness, relevancy, user feedback
- Cost tracking per query/user/department
- Docker deploy, tương thích với LlamaIndex + LangChain

---

### 5 layers metrics cần track

#### Layer 1: Query Metrics (Infrastructure)

```python
# Prometheus metrics
from prometheus_client import Counter, Histogram, Gauge

query_total = Counter(
    'rag_queries_total', 
    'Total queries', 
    ['route', 'department', 'status']  # status: success/error/denied
)

query_latency = Histogram(
    'rag_query_latency_seconds',
    'Query latency',
    ['route'],  # sql/doc/hybrid
    buckets=[0.5, 1, 2, 5, 10, 30]
)

active_queries = Gauge(
    'rag_active_queries',
    'Currently processing queries'
)
```

**Target SLAs:**

| Metric | Target | Alert threshold |
|--------|--------|-----------------|
| Latency p50 | < 3s | > 5s |
| Latency p95 | < 8s | > 15s |
| Error rate | < 2% | > 5% |
| Availability | > 99.5% | < 99% |

#### Layer 2: Router Metrics

```python
# Track router decisions
router_metrics = {
    "route_distribution": Counter(
        'rag_route_decisions', 
        'Route decisions', 
        ['route_type']  # sql_only / doc_only / hybrid / ambiguous
    ),
    
    "route_confidence": Histogram(
        'rag_route_confidence',
        'Router confidence score',
        ['route_type'],
        buckets=[0.3, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
    ),
    
    "ambiguous_queries": Counter(
        'rag_ambiguous_queries',
        'Queries that required LLM fallback for routing'
    ),
}
```

**Đo Route Accuracy hàng tuần:**
- Sample 100 queries random
- Human label đúng route
- Tính accuracy = đúng / tổng
- Target: > 85% Phase 2, > 92% Phase 3+

#### Layer 3: Retrieval Quality Metrics

```python
# LangFuse tracing cho retrieval
from langfuse import Langfuse

langfuse = Langfuse()  # self-hosted instance

def traced_retrieval(query: str, user_context: dict):
    trace = langfuse.trace(
        name="retrieval",
        user_id=user_context["user_id"],
        metadata={"department": user_context["department"]},
    )
    
    # Track retrieval step
    retrieval_span = trace.span(name="vector_search")
    results = retrieve_chunks(query)
    retrieval_span.end(
        metadata={
            "num_results": len(results),
            "top_score": results[0].score if results else 0,
            "cache_hit": results.from_cache,
        }
    )
    
    # Track reranking step
    rerank_span = trace.span(name="reranker")
    reranked = reranker.rerank(results)
    rerank_span.end()
    
    # Score retrieval quality (sampled — không score mọi query)
    if should_sample():  # 5-10% queries
        trace.score(
            name="retrieval_relevancy",
            value=compute_retrieval_relevancy(query, reranked),
        )
    
    return reranked
```

**Retrieval metrics:**

| Metric | Cách đo | Target |
|--------|---------|--------|
| Recall@5 | Chunk đúng có trong top 5? | > 80% |
| MRR | Chunk đúng đứng thứ mấy? | > 0.6 |
| Cache hit rate | % queries served from cache | > 30% |
| Embedding drift | Cosine sim of same queries over time | < 5% drift |

#### Layer 4: Generation Quality Metrics

```python
# Sampled evaluation — không eval mọi query (tốn cost)
def evaluate_generation(query, context, answer, sample_rate=0.05):
    """Chỉ evaluate 5% queries trên production."""
    if random.random() > sample_rate:
        return
    
    # Faithfulness: answer có bịa không?
    faithfulness = evaluate_faithfulness(answer, context)
    
    # Relevancy: answer có trả lời đúng câu hỏi?
    relevancy = evaluate_relevancy(query, answer)
    
    # Log to LangFuse
    langfuse.score(name="faithfulness", value=faithfulness)
    langfuse.score(name="answer_relevancy", value=relevancy)
    
    # Alert nếu dưới threshold
    if faithfulness < 0.7:
        alert("LOW_FAITHFULNESS", query=query, score=faithfulness)
```

**Generation targets:**

| Metric | Target | Alert |
|--------|--------|-------|
| Faithfulness | > 90% | < 80% |
| Answer relevancy | > 85% | < 75% |
| Hallucination rate | < 5% | > 10% |
| Token usage/query | < 4000 | > 8000 |

#### Layer 5: User Satisfaction Metrics

```python
# In-app feedback — đơn giản nhất nhưng quan trọng nhất
@app.post("/feedback")
def submit_feedback(query_id: str, rating: int, comment: str = None):
    """
    rating: 1 (thumbs down) | 2 (neutral) | 3 (thumbs up)
    """
    langfuse.score(
        trace_id=query_id,
        name="user_feedback",
        value=rating,
        comment=comment,
    )
    
    # Track cho dashboard
    feedback_counter.labels(rating=rating).inc()
    
    # Auto-escalate thumbs down
    if rating == 1:
        add_to_review_queue(query_id, comment)
```

**User metrics:**

| Metric | Cách đo | Healthy |
|--------|---------|---------|
| Thumbs up rate | %  queries nhận 👍 | > 70% |
| Reformulation rate | User hỏi lại cùng topic | < 20% |
| Session abandonment | User rời đi giữa chừng | < 15% |
| Weekly active queries/user | Engagement metric | > 5 queries/week |

---

### Grafana Dashboard Layout

```
┌─────────────────────────────────────────────────────────────┐
│                    RAG System Overview                        │
│                                                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │ Queries/min  │ │ P95 Latency  │ │ Error Rate   │         │
│  │    127 ↑     │ │   4.2s ↓     │ │   1.8% →     │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
│                                                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │ Daily Cost   │ │ Cache Hit %  │ │ User 👍 Rate │         │
│  │   $42.30     │ │    34%       │ │    78%       │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
│                                                              │
│  ┌─────────────────────────────────────────────────┐        │
│  │ Route Distribution (24h)                          │        │
│  │ ██████████ SQL: 35%                               │        │
│  │ ████████████████ Doc: 48%                         │        │
│  │ █████ Hybrid: 12%                                 │        │
│  │ ██ Denied: 5%                                     │        │
│  └─────────────────────────────────────────────────┘        │
│                                                              │
│  ┌─────────────────────────────────────────────────┐        │
│  │ Quality Scores (7d rolling)                       │        │
│  │ Faithfulness: 93% ✅                              │        │
│  │ Relevancy:    87% ✅                              │        │
│  │ Hallucination: 3.2% ✅                            │        │
│  └─────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

---

### Alerting Rules

```yaml
# alerting_rules.yaml — Prometheus AlertManager

groups:
  - name: rag_alerts
    rules:
      - alert: HighLatency
        expr: histogram_quantile(0.95, rag_query_latency_seconds) > 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RAG p95 latency > 15s cho 5 phút liên tiếp"

      - alert: HighErrorRate
        expr: rate(rag_queries_total{status="error"}[5m]) / rate(rag_queries_total[5m]) > 0.05
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% — kiểm tra LLM API hoặc vector store"

      - alert: LowFaithfulness
        expr: avg(rag_faithfulness_score) < 0.80
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Faithfulness trung bình dưới 80% — kiểm tra data quality"

      - alert: CostOverBudget
        expr: sum(rag_llm_cost_usd) > 100  # daily budget $100
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "LLM cost vượt budget $100/day"

      - alert: LowCacheHitRate
        expr: rag_cache_hit_rate < 0.20
        for: 1h
        labels:
          severity: info
        annotations:
          summary: "Cache hit rate dưới 20% — review cache config"
```

---

### Weekly Review Process

```
Mỗi tuần (30 phút):

1. Dashboard review:
   - Query volume trend — đang tăng hay giảm?
   - Latency trend — có component nào degrading?
   - Cost trend — đang nằm trong budget?

2. Quality review:
   - Top 10 thumbs-down queries → input cho eval harness
   - Faithfulness/relevancy scores → có đang giảm?
   - Route accuracy spot check (10 queries random)

3. Action items:
   - Thêm golden test cases từ fail cases
   - Tune retrieval params nếu quality giảm
   - Scale infrastructure nếu latency tăng

4. Monthly stakeholder report:
   - ROI: time saved per user * number of users
   - Quality: faithfulness + user satisfaction trend
   - Cost: LLM spend + infrastructure
```

---

### Checklist triển khai

```
Phase 2 — MVP Observability (1 tuần):
☐ Deploy LangFuse (Docker, self-host)
☐ Instrument pipeline với LangFuse traces
☐ Prometheus metrics cho latency + error rate
☐ Grafana dashboard cơ bản (4 panels chính)
☐ In-app thumbs up/down

Phase 3 — Production Grade (2 tuần):
☐ Sampled evaluation (faithfulness, relevancy)
☐ Alert rules (latency, error, cost, quality)
☐ Weekly review process documented
☐ Cost tracking per department
☐ User satisfaction tracking
```

---

### Liên quan

- Pipeline: [02_pipeline.md](./02_pipeline.md) — Phase 5 validation
- Security: [03_security_rbac.md](./03_security_rbac.md) — track access denied events
- Cost: [07_cost_optimization.md](./07_cost_optimization.md) — cost monitoring integration
