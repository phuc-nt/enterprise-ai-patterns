## Vấn đề cốt lõi trước khi nói kiến trúc

Phần lớn dự án AI enterprise thất bại không phải vì kiến trúc sai, mà vì **bắt đầu quá tham**. Planner + MCP + Agentic RAG + Hybrid Retrieval cùng lúc = không bao giờ lên production.

Nguyên tắc mình đề xuất: **Ship một lớp, đo, rồi mới thêm lớp tiếp theo.**

---

## Lộ trình 4 giai đoạn thực tế

---

### Giai đoạn 1 — Foundation (4–8 tuần)
**Mục tiêu: Có thứ gì đó chạy được và người dùng thật sự dùng**

Bắt đầu với **một nguồn dữ liệu duy nhất** — chọn cái đau nhất với người dùng nội bộ, thường là DB hoặc tài liệu chính sách.

```
User → Single RAG Pipeline → 1 nguồn dữ liệu → Answer
```

Stack đề xuất:
- **LlamaIndex** hoặc **LangChain** làm orchestration layer
- **pgvector** hoặc **Qdrant** làm vector store (tự host được, không phụ thuộc cloud)
- **Chunking strategy**: thử recursive character splitting trước, đừng over-engineer
- **Embedding**: `text-embedding-3-small` của OpenAI, hoặc `bge-m3` nếu cần multilingual Vietnamese
- LLM: Claude Sonnet hoặc GPT-4o — không dùng model yếu ở giai đoạn này, cần baseline tốt để đo

**Điều quan trọng nhất giai đoạn này:** Xây **evaluation harness** ngay từ đầu. Tạo 30–50 câu hỏi "golden" có đáp án đúng từ domain thật. Không có cái này, các giai đoạn sau không biết mình đang tiến bộ hay thụt lùi.

**Access Control — phải có ngay Phase 1:**

Với 20K+ nhân viên và hàng trăm phòng ban, RBAC không phải "thêm sau" — là prerequisite:
- Tích hợp **SSO/Active Directory** ngay từ đầu (chắc chắn doanh nghiệp đã có)
- Mỗi chunk khi embed phải gắn metadata: `access_level`, `allowed_departments`, `confidentiality`
- Filter theo RBAC **trước khi** similarity search — không phải filter kết quả sau
- SQL track: mỗi user role chỉ access được subset tables theo scope

Không có lớp này, nhân viên bất kỳ có thể hỏi ra dữ liệu lương, kỷ luật, chiến lược nội bộ — **unacceptable cho enterprise**.

→ Chi tiết: xem [03_security_rbac.md](./03_security_rbac.md)

**Chỉ số cần đo:** Answer relevancy, faithfulness (không hallucinate), latency p95.

**Adoption — bắt đầu pilot đúng cách:**
- Chọn **1 phòng ban** pain point lớn nhất (vd: Operations hoặc Sales), 20–50 users
- Thu feedback hàng tuần qua in-app thumbs up/down
- Có 1 "Champion" ở phòng pilot để training và collect feedback

---

### Giai đoạn 2 — Multi-source Routing (6–10 tuần)
**Mục tiêu: Hệ thống tự biết hỏi đâu**

Đây là lúc thêm **Router layer** — tương đương Intent Agent của Uber nhưng đơn giản hơn.

```
User → Query Classifier → Route đến đúng nguồn → RAG → Answer
         ↓
    DB (Text-to-SQL)
    Văn bản (Vector RAG)
    Media (transcript/OCR → Vector RAG)
```

**Điểm quan trọng:** Classifier này **không cần LLM**. Dùng embedding similarity + keyword rule đã đủ cho 80% trường hợp. Gọi LLM chỉ khi ambiguous. Lý do: latency và cost. Tuy nhiên, với 20K users, nên cân nhắc train **intent classification model nhẹ** (fine-tuned PhoBERT) để tăng accuracy cho tiếng Việt — cost rất thấp nhưng route accuracy tăng đáng kể.

Cho Text-to-SQL riêng: đừng làm từ đầu, dùng **Vanna.ai** hoặc **SQLAI** làm base, fine-tune với schema và SQL samples nội bộ. Uber mất 20 iterations vì họ build from scratch — bạn không cần làm vậy năm 2026.

Với media (audio/video): pipeline thực tế là `Whisper → transcript → chunk → embed`, không cần xử lý frame-by-frame trừ khi yêu cầu visual.

**Semantic Caching — nên thêm ở giai đoạn này:**

Với 20K users, 30%+ queries sẽ lặp lại hoặc tương tự. Thêm semantic cache layer trước pipeline:
- Cache hit (cosine similarity > 0.92) → trả kết quả ngay, skip pipeline
- Giảm 40–60% cost LLM, latency từ ~6s → ~100ms cho cache hit
- Cache key = `(query_embedding, user_role)` — phải respect RBAC
- TTL: SQL results 1–4h, doc knowledge 24–72h

→ Chi tiết: xem [07_cost_optimization.md](./07_cost_optimization.md)

**Chỉ số mới cần thêm:** Route accuracy (classifier chọn đúng nguồn bao nhiêu %), cross-source query detection, cache hit rate.

---

### Giai đoạn 3 — Context Quality (6–8 tuần)
**Mục tiêu: Retrieval tốt hơn, không phải generation tốt hơn**

Đây là giai đoạn mà hầu hết team bỏ qua nhưng lại tạo ra impact lớn nhất.

Vấn đề thực tế của enterprise: data messy, tên cột không nhất quán, tài liệu có nhiều version, media transcript đầy noise.

Ba việc cần làm theo thứ tự ưu tiên:

**1. Hybrid Search** — kết hợp dense (vector) + sparse (BM25) retrieval. Với tài liệu nội bộ có nhiều thuật ngữ chuyên ngành, BM25 thường thắng pure vector. Dùng **Reciprocal Rank Fusion** để merge kết quả. LlamaIndex có built-in.

**2. Metadata Filtering** — tag mỗi chunk với: `source_type`, `department`, `date`, `document_id`. Cho phép filter trước khi search. Giảm noise đáng kể.

**3. Contextual Chunking** — thay vì chunk cơ học, prepend document context vào mỗi chunk trước khi embed. Anthropic có paper về cái này ("Contextual Retrieval"). Cải thiện recall ~20-30% mà không cần thay model.

**4. Query Rewriting** — người dùng enterprise thường hỏi mơ hồ hoặc dùng ngôn ngữ không match với tài liệu. Thêm bước LLM rewrite query thành 2–3 variants → multi-query retrieval → RRF merge. LlamaIndex có built-in support.

**5. Reranker** — sau hybrid search, thêm reranker (`BAAI/bge-reranker-v2-m3`, multilingual, self-host). Cải thiện precision 15–25% với chi phí latency tối thiểu.

**Chưa cần:** Agentic RAG, multi-hop reasoning, graph RAG — những cái này có ROI thấp hơn các việc trên ở giai đoạn này.

---

### Giai đoạn 4 — Agentic Layer (optional, 8–12 tuần)
**Mục tiêu: Xử lý câu hỏi phức tạp multi-step**

Chỉ làm giai đoạn này nếu: evaluation harness từ G1 cho thấy có một loại câu hỏi nhất định mà pipeline hiện tại không xử lý được, và loại câu hỏi đó thực sự quan trọng với business.

Kiến trúc đề xuất lúc này:

```
User → Router → Planner (LLM)
                    ↓
            Sub-task 1: SQL query
            Sub-task 2: Doc retrieval  
            Sub-task 3: API call
                    ↓
              Synthesizer → Answer với sources + confidence
```

**Dùng LangGraph** cho orchestration agentic — không dùng AutoGen hay CrewAI cho production enterprise vì khó debug và không predictable đủ.

**Validation layer** — bắt buộc ở đây: dry-run SQL trước khi execute, semantic checker so sánh answer với nguồn gốc, circuit breaker nếu planner loop quá 3 bước.

**Fallback / Graceful Degradation** — bắt buộc ở mọi giai đoạn:
- LLM API down → fallback to keyword search + template answers
- Retrieval confidence thấp → "Tôi không đủ thông tin" thay vì hallucinate
- Query ngoài scope → redirect user đến expert/FAQ
- SQL generation fail → suggest manual query builder thay vì trả error trống

**MCP:** Đúng là hướng đúng cho tool standardization, nhưng chỉ adopt khi vendor của bạn (Jira, Confluence, Salesforce...) đã có stable MCP server. Đừng tự viết MCP server cho internal tools ở giai đoạn này — chi phí maintenance cao hơn benefit.

---

## Một cảnh báo thực tế

Câu ví dụ trong bài gốc — *"Doanh thu giảm vì conversion giảm 18%, do thay đổi quy trình từ văn bản Y"* — đây là **causal reasoning**, không phải retrieval. Không có pipeline RAG hay Text-to-SQL nào tự suy ra nhân quả được. Hệ thống chỉ có thể: lấy số liệu từ DB, tìm văn bản liên quan, rồi **để LLM synthesize**. Nếu LLM synthesis sai thì không có cách nào biết được trừ khi có human-in-the-loop validation.

Điều này không có nghĩa là đừng làm — mà là cần set expectation đúng với stakeholder ngay từ đầu.

---

## Cross-cutting concerns — chạy song song từ G1

Những thứ sau không thuộc giai đoạn nào cụ thể mà phải có từ đầu và cải thiện liên tục:

| Concern | Bắt đầu từ | Chi tiết |
|---------|-----------|----------|
| **Access Control / RBAC** | G1 | [03_security_rbac.md](./03_security_rbac.md) |
| **Observability & Monitoring** | G2 | [04_observability.md](./04_observability.md) |
| **Data Freshness Pipeline** | G2 | [05_data_lifecycle.md](./05_data_lifecycle.md) |
| **Adoption & Rollout** | G1 | [06_adoption_rollout.md](./06_adoption_rollout.md) |
| **Cost Optimization / Caching** | G2 | [07_cost_optimization.md](./07_cost_optimization.md) |

---

## Tóm lại thứ tự ưu tiên

| Giai đoạn | Thời gian | ROI | Rủi ro | Lưu ý thêm |
|---|---|---|---|---|
| G1: Single-source RAG + Eval + RBAC | 4–8 tuần | Cao nhất | Thấp | Pilot 1 phòng ban, 20–50 users |
| G2: Multi-source Router + Caching | 6–10 tuần | Cao | Trung bình | Thêm semantic cache, monitoring |
| G3: Context Quality + Reranker | 6–8 tuần | Cao | Thấp | Query rewriting, data freshness |
| G4: Agentic Layer | 8–12 tuần | Trung bình | Cao | Rollout toàn công ty nếu G1–3 ổn |

Quan trọng nhất: **Eval harness ở G1 quyết định thành bại của toàn bộ lộ trình**. Không có nó, mọi quyết định kỹ thuật sau đó đều là đoán mò.