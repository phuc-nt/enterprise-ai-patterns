# RAG — Retrieval-Augmented Generation

Blueprint xây dựng hệ thống RAG kết hợp DB + Văn bản, từ prototype đến production.

## Tài liệu

| # | File | Nội dung |
|---|------|----------|
| 01 | [Idea & Roadmap](01_idea.md) | Lộ trình 4 giai đoạn, nguyên tắc "ship → đo → mở rộng" |
| 02 | [Pipeline](02_pipeline.md) | Dual-track SQL + Doc RAG, Router, Hybrid Synthesis |
| 03 | [Security & RBAC](03_security_rbac.md) | 3-layer access control, PII masking, audit trail |
| 04 | [Observability](04_observability.md) | 5-layer metrics, LangFuse + Prometheus + Grafana |
| 05 | [Data Lifecycle](05_data_lifecycle.md) | Incremental ingestion, change detection, freshness |
| 06 | [Adoption & Rollout](06_adoption_rollout.md) | Pilot → GA rollout, champion program, ROI framework |
| 07 | [Cost Optimization](07_cost_optimization.md) | Semantic caching, model tiering, budget template |

## Đọc theo thứ tự

```
01 (chiến lược) → 02 (kỹ thuật) → 03 (bảo mật) → 04-07 (vận hành)
```

## Bối cảnh

- Doanh nghiệp lớn, 20K+ nhân viên, hàng trăm phòng ban
- Nguồn dữ liệu: Database (SQL) + Tài liệu (văn bản, chính sách, SOP)
- Yêu cầu: RBAC, audit, data freshness, cost control
- Tham khảo: [Uber QueryGPT](https://www.uber.com/blog/query-gpt/), Anthropic Contextual Retrieval, RAGAS
