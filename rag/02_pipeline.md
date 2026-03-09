## Pipeline DB + Văn bản — Theo Phase

---

### Khái niệm nền cần đồng thuận trước

Câu hỏi enterprise thực tế rơi vào 3 loại:

```
Loại 1 — Số liệu thuần:   "Doanh thu Q3 bao nhiêu?"
                           → Chỉ cần SQL

Loại 2 — Tri thức thuần:  "Quy trình xét duyệt hồ sơ là gì?"
                           → Chỉ cần RAG văn bản

Loại 3 — Kết hợp:         "Tại sao conversion tháng này giảm?"
                           → SQL lấy số + RAG lấy context + LLM synthesize
```

Hầu hết hệ thống chỉ làm tốt loại 1 hoặc 2. **Loại 3 mới là giá trị thực sự — và cũng là thứ khó nhất.** Lộ trình dưới đây build dần từ 1→2→3.

---

### Phase 1 — Dual Track Độc Lập (4–6 tuần)

**Mục tiêu:** Có 2 pipeline riêng biệt chạy được, đo được, trước khi merge.

```
┌─────────────────────────────────────────────────────┐
│                     USER QUERY                      │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────┴────────┐
              │   AUTH LAYER     │ ← SSO / Active Directory
              │   Who? Role?     │   Định danh user + phòng ban
              │   Department?    │
              └────────┬─────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
   TRACK A: SQL                TRACK B: DOC RAG
   (scoped by role)            (filtered by dept)
          │                         │
   Vanna.ai hoặc               LlamaIndex +
   LangChain SQLChain           Qdrant/pgvector
          │                         │
   DB Schema +               Chunked docs +
   Few-shot SQL samples       Hybrid search
   (chỉ tables trong scope)    (chỉ chunks user được phép)
          │                         │
      SQL Result               Doc chunks
          │                         │
          └────────────┬────────────┘
                       │
              (Chưa merge — user
               chọn track thủ công
               hoặc 2 tab riêng)
```

**Tại sao chưa merge ở Phase 1?**

Vì nếu merge sớm mà một track sai, không biết lỗi từ đâu. Tách ra để eval từng track độc lập trước.

**Stack cụ thể:**

Track SQL:
- **Vanna.ai** (open-source) — đã có built-in RAG trên SQL samples, train được với schema nội bộ
- Kết nối trực tiếp PostgreSQL/MySQL/MSSQL
- Thêm few-shot examples: 30–50 cặp (câu hỏi → SQL đúng) từ domain thật

Track Doc RAG:
- **LlamaIndex** làm orchestration
- **Qdrant** làm vector store (self-host, Docker, không cần cloud)
- Chunking: `SentenceSplitter`, chunk size 512 tokens, overlap 64
- Embedding: `bge-m3` nếu có tiếng Việt lẫn tiếng Anh trong tài liệu
- **RBAC metadata** phải gắn ngay khi embed: `access_level`, `allowed_departments`, `confidentiality`

Access Control — bắt buộc ngay Phase 1:
- Tích hợp **SSO/Active Directory** của doanh nghiệp
- Mỗi chunk embed đi kèm metadata RBAC → filter **trước** similarity search
- SQL track: mặc định chỉ expose tables thuộc domain của user

→ Xem chi tiết [03_security_rbac.md](./03_security_rbac.md)

**Eval harness — bắt buộc ngay Phase 1:**

```python
# Tạo golden dataset cho từng track
sql_golden = [
    {"question": "Doanh thu tháng 3?", 
     "expected_sql": "SELECT SUM(revenue) FROM sales WHERE month=3",
     "expected_result": 1250000},
    ...
]

doc_golden = [
    {"question": "Quy trình phê duyệt hợp đồng?",
     "expected_chunks": ["doc_id_123", "doc_id_456"],
     "expected_answer_contains": ["phòng pháp chế", "5 ngày làm việc"]},
    ...
]
```

Metric cần đo mỗi tuần:
- SQL track: execution success rate, result accuracy vs golden
- Doc track: recall@5 (có chunk đúng trong top 5 không), faithfulness (answer có bịa không)

---

### Phase 2 — Router Layer (3–4 tuần)

**Mục tiêu:** Hệ thống tự quyết định dùng track nào, không cần user chọn.

```
┌─────────────────────────────────────────────────────┐
│                     USER QUERY                      │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────▼────────┐
              │  QUERY ROUTER   │
              │                 │
              │  Rule-based     │
              │  + Embedding    │
              │  classifier     │
              └────────┬────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
      sql_only    doc_only      hybrid
         │             │             │
    SQL Track     Doc Track    Cả 2 song song
```

**Router implementation — quan trọng: đừng dùng LLM để route nếu không cần thiết.**

```python
class QueryRouter:
    
    # Tier 1: Rule-based (nhanh, 0 cost)
    SQL_KEYWORDS = ["bao nhiêu", "tổng", "trung bình", 
                    "doanh thu", "so sánh", "tháng", "quý", "năm"]
    DOC_KEYWORDS = ["quy trình", "chính sách", "hướng dẫn", 
                    "quy định", "điều khoản", "như thế nào"]
    
    def route(self, query: str) -> str:
        sql_score = sum(1 for kw in self.SQL_KEYWORDS if kw in query)
        doc_score = sum(1 for kw in self.DOC_KEYWORDS if kw in query)
        
        if sql_score > 0 and doc_score == 0:
            return "sql_only"
        elif doc_score > 0 and sql_score == 0:
            return "doc_only"
        elif sql_score > 0 and doc_score > 0:
            return "hybrid"
        else:
            # Tier 2: Embedding similarity với anchor examples
            return self._embedding_classify(query)
    
    def _embedding_classify(self, query: str) -> str:
        # So sánh query với ~20 anchor examples cho mỗi loại
        # Dùng cosine similarity với pre-computed embeddings
        ...
```

Chỉ gọi LLM để classify khi cả 2 tiers trên không confident (threshold < 0.6). Trong thực tế, rule-based xử lý được ~70% query.

**Metric mới Phase 2:**
- Route accuracy: router chọn đúng track bao nhiêu % (cần ~100 labeled test queries)
- Latency breakdown: bao nhiêu ms ở router, bao nhiêu ở execution

---

### Phase 3 — Context Quality (4–6 tuần)

**Mục tiêu:** Cải thiện chất lượng retrieval, không phải thêm tính năng mới.

Đây là phase mà hầu hết team bỏ qua nhưng tạo impact lớn nhất trên production.

**3A — Hybrid Search cho Doc Track:**

```
Thay vì: Query → Vector Search → Top K chunks
Thành:   Query → Query Rewrite → [Vector Search + BM25] → RRF Merge → Reranker → Top K chunks
```

Lý do BM25 quan trọng với enterprise Việt Nam: tài liệu nội bộ có nhiều từ viết tắt, mã sản phẩm, tên phòng ban — những thứ mà vector search thường miss vì không đủ training data.

```python
from llama_index.retrievers import BM25Retriever, VectorIndexRetriever
from llama_index.retrievers import QueryFusionRetriever

retriever = QueryFusionRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    similarity_top_k=10,
    num_queries=1,  # Không sinh query variant — giữ đơn giản
    mode="reciprocal_rerank",
)
```

Thêm reranker ở cuối: `BAAI/bge-reranker-v2-m3` (hỗ trợ multilingual, self-host được). Cải thiện precision 15–25%.

**3A.5 — Query Rewriting cho Doc Track:**

Người dùng enterprise hỏi mơ hồ hoặc dùng ngôn ngữ không match tài liệu. Thêm bước rewrite:

```python
from llama_index.indices.query.query_transform import HyDEQueryTransform
# Hoặc đơn giản hơn:
# LLM rewrite user query → 2-3 variants → multi-query retrieval → RRF merge

# Ví dụ: "Tình hình tháng này?" →
# Variant 1: "Báo cáo doanh thu tháng hiện tại"
# Variant 2: "Kết quả kinh doanh tháng 3"
# Variant 3: "Tổng hợp chỉ số hoạt động tháng này"
```

LlamaIndex có built-in `SubQuestionQueryEngine` và `HyDEQueryTransform` cho việc này.

**3B — Contextual Chunking cho tài liệu dài:**

Thay vì chunk cơ học, prepend document-level context vào mỗi chunk trước khi embed:

```python
CONTEXT_PROMPT = """
Đây là đoạn từ tài liệu: {doc_title}
Phòng ban: {department} | Ngày ban hành: {date} | Loại: {doc_type}

Nội dung đoạn:
{chunk_content}
"""
```

Hướng tiếp cận này — prepend context trước khi embed — giúp bridging semantic gap giữa raw data và câu hỏi thực tế của người dùng, đặc biệt hiệu quả với tài liệu có cấu trúc phức tạp.

**3C — Metadata Filtering cho SQL Track:**

Thêm lớp schema metadata để SQL track không cần scan toàn bộ DB:

```python
# Mỗi table có metadata profile
TABLE_METADATA = {
    "sales_transactions": {
        "domain": ["doanh thu", "bán hàng", "đơn hàng"],
        "time_columns": ["created_at", "transaction_date"],
        "key_metrics": ["revenue", "quantity", "discount"],
        "joins_with": ["customers", "products"]
    },
    "hr_records": {
        "domain": ["nhân sự", "nhân viên", "lương"],
        ...
    }
}

# Router dùng metadata này để pre-filter tables
# trước khi đưa vào SQL generation
```

Đây là bản rút gọn của Table Agent trong QueryGPT — không cần build agent phức tạp, chỉ cần metadata lookup là đủ cho 80% cases.

**3D — Column Pruning cho SQL Track (học từ Uber QueryGPT):**

Với DB enterprise có tables 200+ columns (rất phổ biến), đưa toàn bộ schema vào prompt = 40–60K tokens = tốn cost + giảm chất lượng.

```python
def prune_columns(table_name: str, user_query: str, schema: dict) -> list:
    """
    Dùng LLM call nhẹ để chọn chỉ các columns liên quan đến câu hỏi.
    Giảm 60-80% token usage cho SQL generation.
    """
    all_columns = schema[table_name]
    prompt = f"""
    Bảng: {table_name}
    Tất cả columns: {all_columns}
    Câu hỏi: {user_query}
    
    Chọn CHỈ các columns cần thiết để trả lời câu hỏi trên.
    Trả về danh sách column names, không giải thích.
    """
    return llm_call(prompt)  # Dùng model nhẹ như GPT-4o-mini cho bước này
```

Uber dùng "Column Prune Agent" riêng cho việc này — ta chỉ cần 1 function call đơn giản là đủ.

---

### Phase 4 — Hybrid Synthesis (4–6 tuần)

**Mục tiêu:** Xử lý query loại 3 — kết hợp số liệu + văn bản trong một câu trả lời.

```
┌──────────────────────────────────────────────────────────┐
│                      USER QUERY                          │
│    "Tại sao tỷ lệ chuyển đổi tháng này giảm?"           │
└────────────────────────┬─────────────────────────────────┘
                         │
                 HYBRID route
                         │
          ┌──────────────┴──────────────┐
          │                             │
    SQL Branch                    DOC Branch
    (song song)                   (song song)
          │                             │
   SELECT conversion_rate         Tìm tài liệu:
   FROM funnel_metrics             - Chính sách xét duyệt
   WHERE month = current           - Biên bản họp gần đây
          │                        - SOP thay đổi gần đây
          │                             │
     {số liệu thô}              {relevant chunks}
          │                             │
          └──────────────┬──────────────┘
                         │
              SYNTHESIS PROMPT
                         │
    "Dữ liệu cho thấy: [số liệu]
     Tài liệu liên quan: [chunks]
     Hãy giải thích mối liên hệ,
     ghi rõ nguồn cho từng điểm,
     nêu giả định và độ tin cậy."
                         │
              STRUCTURED ANSWER
              ├── Số liệu (nguồn: DB, table X)
              ├── Nguyên nhân khả dĩ (nguồn: doc Y, trang Z)
              ├── Độ tin cậy: cao/trung bình/thấp
              └── Gợi ý kiểm tra thêm
```

**Synthesis prompt template:**

```python
SYNTHESIS_PROMPT = """
Bạn là data analyst nội bộ. Hãy trả lời câu hỏi dựa trên:

DỮ LIỆU SỐ LIỆU:
{sql_results}

TÀI LIỆU THAM CHIẾU:
{doc_chunks}

YÊU CẦU:
1. Chỉ kết luận những gì có bằng chứng từ nguồn trên
2. Với mỗi điểm, ghi rõ nguồn (DB: tên_bảng | Doc: tên_tài_liệu, trang X)
3. Nếu không đủ thông tin để kết luận, nói thẳng là không biết
4. Cuối cùng: liệt kê 2-3 câu hỏi cần kiểm tra thêm

CÂU HỎI: {user_query}
"""
```

**Điểm quan trọng nhất Phase 4:** Đừng để LLM tự suy luận nhân quả không có bằng chứng. Prompt phải ép buộc citation. Nếu DB không có dữ liệu conversion breakdown, answer phải nói "không đủ dữ liệu để kết luận" thay vì bịa.

---

### Phase 5 — Validation & Observability (ongoing)

Không phải phase cuối mà là lớp chạy song song từ Phase 2 trở đi.

**SQL Validation:**
```python
def validate_sql(generated_sql: str, schema: dict) -> ValidationResult:
    checks = [
        check_table_exists(generated_sql, schema),      # Hallucinate table?
        check_columns_exist(generated_sql, schema),     # Hallucinate column?
        dry_run_explain(generated_sql, db_conn),        # Query plan hợp lệ?
        check_no_write_operations(generated_sql),       # Chỉ SELECT thôi
        check_result_row_limit(generated_sql),          # Tránh query trả 10M rows
        check_rbac_table_access(generated_sql, user),   # User có quyền access?
    ]
    return ValidationResult(passed=all(checks), errors=[...])
```

**RAG Faithfulness Check:**
Sau khi generate answer, gọi thêm một LLM call nhỏ để verify:
```
"Answer này có điểm nào không được support bởi chunks dưới đây không?
 Answer: {answer}
 Chunks: {retrieved_chunks}
 Trả lời: CÓ hoặc KHÔNG, và giải thích điểm nào nếu CÓ."
```

Chi phí thêm ~10% nhưng giảm đáng kể hallucination rate trên production.

**Production Observability Stack:**

Với 20K users, cần monitoring thực sự, không chỉ validation:

| Layer | Metric | Tool gợi ý |
|-------|--------|------------|
| Query | Volume/s, latency p50/p95/p99, error rate | Prometheus + Grafana |
| Router | Route accuracy, distribution, ambiguous rate | Custom metrics |
| Retrieval | Recall@k, MRR, cache hit rate, embedding drift | LangFuse (self-host) |
| Generation | Faithfulness, hallucination rate, token/cost per query | RAGAS + LangFuse |
| User | Thumbs up/down, reforms, session abandonment | Analytics events |

**Alerting bắt buộc:**
- Hallucination rate > 10% baseline → alert
- Latency p95 > 10s → alert
- LLM cost/day vượt budget → alert
- Cache hit rate < 20% (expected 30%+) → kiểm tra cache config

→ Chi tiết: xem [04_observability.md](./04_observability.md)

---

### Tóm tắt lộ trình

```
Phase 1 (4-6w): Dual track độc lập + Eval harness + RBAC
                → Biết baseline của từng track, bảo mật từ đầu

Phase 2 (3-4w): Router layer + Semantic Cache
                → Tự động chọn track, đo route accuracy, giảm cost

Phase 3 (4-6w): Context quality + Column Pruning
                → Hybrid search + Reranker + Query Rewriting 
                → Impact lớn nhất trên accuracy, ít rủi ro nhất

Phase 4 (4-6w): Hybrid synthesis
                → Xử lý query loại 3, answer có citation

Phase 5 (ongoing): Validation + Observability + Data Freshness
                → SQL safety, faithfulness monitor, incremental re-embed
```

Tổng thời gian thực tế: **4–6 tháng** cho đội 2–3 engineer có kinh nghiệm.

---

### Tài liệu liên quan

| Tài liệu | Nội dung |
|----------|----------|
| [03_security_rbac.md](./03_security_rbac.md) | Access control, RBAC, data governance |
| [04_observability.md](./04_observability.md) | Monitoring, alerting, LLM ops |
| [05_data_lifecycle.md](./05_data_lifecycle.md) | Data freshness, incremental ingestion |
| [06_adoption_rollout.md](./06_adoption_rollout.md) | Pilot plan, change management, training |
| [07_cost_optimization.md](./07_cost_optimization.md) | Semantic caching, token optimization, budget |