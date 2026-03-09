# Enterprise AI Patterns

Tài liệu kiến trúc và blueprint thực tế cho triển khai AI trong doanh nghiệp lớn (20K+ nhân viên).

## RAG — Retrieval-Augmented Generation

Blueprint xây dựng hệ thống RAG kết hợp DB + Văn bản, từ prototype đến production.

| # | Tài liệu | Nội dung |
|---|----------|----------|
| 01 | [Idea & Roadmap](rag/01_idea.md) | Lộ trình 4 giai đoạn, nguyên tắc "ship → đo → mở rộng" |
| 02 | [Pipeline](rag/02_pipeline.md) | Dual-track SQL + Doc RAG, Router, Hybrid Synthesis |
| 03 | [Security & RBAC](rag/03_security_rbac.md) | 3-layer access control, PII masking, audit trail |
| 04 | [Observability](rag/04_observability.md) | 5-layer metrics, LangFuse + Prometheus + Grafana |
| 05 | [Data Lifecycle](rag/05_data_lifecycle.md) | Incremental ingestion, change detection, freshness |
| 06 | [Adoption & Rollout](rag/06_adoption_rollout.md) | Pilot → GA rollout, champion program, ROI framework |
| 07 | [Cost Optimization](rag/07_cost_optimization.md) | Semantic caching, model tiering, budget template |

### Đọc theo thứ tự

```
01 (chiến lược) → 02 (kỹ thuật) → 03 (bảo mật) → 04-07 (vận hành)
```

### Bối cảnh thiết kế

- Doanh nghiệp lớn, hàng trăm phòng ban
- Nguồn dữ liệu: Database (SQL) + Tài liệu (văn bản, chính sách, SOP)
- Yêu cầu: RBAC, audit, data freshness, cost control
- Tham khảo: [Uber QueryGPT](https://www.uber.com/blog/query-gpt/), Anthropic Contextual Retrieval, RAGAS

## License

Internal reference — dùng tự do.
