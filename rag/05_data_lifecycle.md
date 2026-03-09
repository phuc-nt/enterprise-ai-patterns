## Data Lifecycle & Freshness Pipeline

---

### Vấn đề: RAG chỉ tốt bằng data nó có

Với doanh nghiệp 20K nhân viên:
- Hàng trăm tài liệu chính sách thay đổi hàng tháng
- DB data thay đổi real-time (doanh thu, nhân sự, đơn hàng)
- Biên bản họp, SOP mới ra hàng ngày
- Nhân viên mới/nghỉ việc → org chart thay đổi liên tục

**Stale data = wrong answers = mất trust = không ai dùng nữa.**

Cả `01_idea.md` và `02_pipeline.md` nói về chunking + embedding nhưng không đề cập **khi nào và cách nào** data được cập nhật. File này fill gap đó.

---

### Kiến trúc Data Lifecycle

```
┌──────────────────────────────────────────────────────────────────┐
│                      DATA SOURCES                                 │
│                                                                   │
│   Documents        Database          Media          Knowledge     │
│   (SharePoint,     (PostgreSQL,      (Video,        (Confluence,  │
│    Google Drive,    MySQL, MSSQL)     Audio)          Wiki)        │
│    File server)                                                   │
│        │                │               │               │         │
└────────┼────────────────┼───────────────┼───────────────┼─────────┘
         │                │               │               │
    ┌────▼────┐      ┌────▼────┐     ┌────▼────┐    ┌────▼────┐
    │ Change  │      │ Change  │     │ Change  │    │ Change  │
    │ Detector│      │ Detector│     │ Detector│    │ Detector│
    └────┬────┘      └────┬────┘     └────┬────┘    └────┬────┘
         │                │               │               │
         └────────────────┴───────┬───────┴───────────────┘
                                  │
                          ┌───────▼───────┐
                          │  Change Queue │
                          │  (message     │
                          │   queue)      │
                          └───────┬───────┘
                                  │
                          ┌───────▼───────┐
                          │  Incremental  │
                          │  Processor    │
                          │               │
                          │  1. Diff      │
                          │  2. Re-chunk  │
                          │  3. Re-embed  │
                          │  4. Update    │
                          │     metadata  │
                          └───────┬───────┘
                                  │
                     ┌────────────┼────────────┐
                     │            │            │
              ┌──────▼──────┐ ┌──▼────┐ ┌─────▼──────┐
              │ Vector Store│ │ Cache │ │ Audit Log  │
              │ (upsert/    │ │ Inval │ │            │
              │  delete)    │ │       │ │            │
              └─────────────┘ └───────┘ └────────────┘
```

---

### 4 loại Change Detection

#### 1. Document Changes — File-based

```python
class DocumentChangeDetector:
    """Detect changes in document sources (SharePoint, Google Drive, file server)."""
    
    def __init__(self, doc_store, vector_store):
        self.doc_store = doc_store
        self.vector_store = vector_store
        self.state_file = "ingestion_state.json"  # Track last processed state
    
    def detect_changes(self) -> list[Change]:
        current_state = self.load_state()
        changes = []
        
        for doc in self.doc_store.list_all():
            prev = current_state.get(doc.id)
            
            if prev is None:
                # Tài liệu mới
                changes.append(Change(
                    type="NEW",
                    doc_id=doc.id,
                    path=doc.path,
                ))
            elif doc.modified_at > prev["last_processed"]:
                # Tài liệu đã thay đổi
                changes.append(Change(
                    type="MODIFIED",
                    doc_id=doc.id,
                    path=doc.path,
                    previous_hash=prev["content_hash"],
                ))
            # Không thay đổi → skip
        
        # Detect deletions
        current_ids = {doc.id for doc in self.doc_store.list_all()}
        deleted_ids = set(current_state.keys()) - current_ids
        for doc_id in deleted_ids:
            changes.append(Change(type="DELETED", doc_id=doc_id))
        
        return changes
```

**Schedule:** Daily scan, hoặc webhook nếu document store hỗ trợ (Google Drive, SharePoint đều có).

#### 2. Database Schema Changes

```python
class SchemaChangeDetector:
    """Detect schema changes that affect SQL track."""
    
    def detect_schema_changes(self) -> list[SchemaChange]:
        current_schema = self.db.get_schema()
        previous_schema = self.load_previous_schema()
        
        changes = []
        
        for table in current_schema:
            if table not in previous_schema:
                changes.append(SchemaChange(type="TABLE_ADDED", table=table))
            else:
                # Check column changes
                new_cols = set(current_schema[table]) - set(previous_schema[table])
                removed_cols = set(previous_schema[table]) - set(current_schema[table])
                
                if new_cols:
                    changes.append(SchemaChange(
                        type="COLUMNS_ADDED", table=table, columns=list(new_cols)
                    ))
                if removed_cols:
                    changes.append(SchemaChange(
                        type="COLUMNS_REMOVED", table=table, columns=list(removed_cols)
                    ))
        
        return changes
```

**Schedule:** Weekly validation. Schema changes ít xảy ra nhưng impact lớn — cần alert ngay.

#### 3. SQL Sample Freshness

```python
class SQLSampleValidator:
    """Validate SQL examples còn hoạt động không."""
    
    def validate_samples(self, samples: list[SQLSample]) -> list:
        stale = []
        
        for sample in samples:
            try:
                # Dry run — không execute thật
                self.db.explain(sample.sql)
            except Exception as e:
                stale.append({
                    "sample_id": sample.id,
                    "question": sample.question,
                    "sql": sample.sql,
                    "error": str(e),
                })
        
        if stale:
            alert(f"{len(stale)} SQL samples đã stale, cần review")
        
        return stale
```

**Schedule:** Monthly review. SQL samples stale = RAG generate SQL sai = mất trust.

#### 4. Org Chart / Permission Changes

```python
class OrgChangeDetector:
    """Detect organizational changes affecting RBAC."""
    
    def detect_org_changes(self):
        current_org = self.ad.get_org_structure()
        previous_org = self.load_previous_org()
        
        changes = []
        
        # Nhân viên mới — cần provision access
        new_employees = current_org.users - previous_org.users
        
        # Nhân viên chuyển phòng — cần update scope
        for user in current_org.users & previous_org.users:
            if current_org.dept(user) != previous_org.dept(user):
                changes.append(OrgChange(
                    type="DEPT_TRANSFER",
                    user=user,
                    old_dept=previous_org.dept(user),
                    new_dept=current_org.dept(user),
                ))
        
        # Nhân viên nghỉ — cần revoke access
        departed = previous_org.users - current_org.users
        
        return changes
```

**Schedule:** Daily sync từ HR system / Active Directory.

---

### Incremental Processing — Chỉ xử lý cái thay đổi

**Nguyên tắc: KHÔNG re-embed toàn bộ corpus. Chỉ incremental.**

```python
class IncrementalProcessor:
    
    def process_change(self, change: Change):
        if change.type == "NEW":
            self._process_new_document(change)
        elif change.type == "MODIFIED":
            self._process_modified_document(change)
        elif change.type == "DELETED":
            self._process_deleted_document(change)
    
    def _process_new_document(self, change):
        """Embed toàn bộ document mới."""
        doc = self.doc_store.get(change.doc_id)
        chunks = self.chunker.split(doc)
        
        for chunk in chunks:
            metadata = self._build_metadata(doc, chunk)
            embedding = self.embed_model.encode(chunk.text)
            self.vector_store.upsert(chunk.id, embedding, metadata)
        
        self._update_state(change.doc_id, doc)
    
    def _process_modified_document(self, change):
        """Chỉ re-embed chunks thay đổi."""
        doc = self.doc_store.get(change.doc_id)
        new_chunks = self.chunker.split(doc)
        old_chunks = self.vector_store.get_by_doc_id(change.doc_id)
        
        # Diff: tìm chunks nào thay đổi
        new_set = {self._hash(c.text): c for c in new_chunks}
        old_set = {self._hash(c.text): c for c in old_chunks}
        
        # Chunks mới hoặc thay đổi → upsert
        to_add = {h: c for h, c in new_set.items() if h not in old_set}
        for chunk in to_add.values():
            metadata = self._build_metadata(doc, chunk)
            embedding = self.embed_model.encode(chunk.text)
            self.vector_store.upsert(chunk.id, embedding, metadata)
        
        # Chunks không còn → delete
        to_remove = {h: c for h, c in old_set.items() if h not in new_set}
        for chunk in to_remove.values():
            self.vector_store.delete(chunk.id)
        
        # Invalidate cache cho doc này
        self.cache.invalidate_by_doc(change.doc_id)
        
        logger.info(
            f"Doc {change.doc_id}: "
            f"+{len(to_add)} chunks, -{len(to_remove)} chunks, "
            f"={len(new_set) - len(to_add)} unchanged"
        )
    
    def _process_deleted_document(self, change):
        """Xóa toàn bộ chunks của document."""
        self.vector_store.delete_by_doc_id(change.doc_id)
        self.cache.invalidate_by_doc(change.doc_id)
```

**Lợi ích incremental vs full re-embed:**

| Metric | Full re-embed | Incremental |
|--------|--------------|-------------|
| Thời gian | Giờ (toàn bộ corpus) | Phút (chỉ thay đổi) |
| Embedding cost | Toàn bộ tokens | Chỉ chunks thay đổi |
| Downtime | Cần maintenance window | Zero downtime |
| Cache impact | Invalidate toàn bộ | Chỉ invalidate affected |

---

### Schedule tổng hợp

```python
# scheduler.py — Cron jobs cho data freshness

from apscheduler.schedulers.blocking import BlockingScheduler

scheduler = BlockingScheduler()

# Documents: Daily 2AM
@scheduler.scheduled_job('cron', hour=2)
def sync_documents():
    changes = doc_detector.detect_changes()
    for change in changes:
        processor.process_change(change)
    logger.info(f"Document sync: {len(changes)} changes processed")

# DB Schema: Weekly Sunday 3AM
@scheduler.scheduled_job('cron', day_of_week='sun', hour=3)
def validate_schema():
    changes = schema_detector.detect_schema_changes()
    if changes:
        alert(f"Schema changes detected: {changes}")
        update_table_metadata(changes)

# SQL Samples: Monthly 1st, 4AM
@scheduler.scheduled_job('cron', day=1, hour=4)
def validate_sql_samples():
    stale = sql_validator.validate_samples(all_samples)
    if stale:
        alert(f"{len(stale)} SQL samples need review")

# Org chart: Daily 1AM
@scheduler.scheduled_job('cron', hour=1)
def sync_org_chart():
    changes = org_detector.detect_org_changes()
    for change in changes:
        update_rbac_permissions(change)
```

---

### Monitoring Data Freshness

```python
# Metrics cho data freshness
data_freshness_gauge = Gauge(
    'rag_data_freshness_hours',
    'Hours since last successful sync',
    ['source_type']  # documents / schema / sql_samples / org
)

stale_documents_gauge = Gauge(
    'rag_stale_documents_count',
    'Number of documents not synced in > 7 days'
)

# Alert nếu sync fail
# → Document sync miss > 24h → warning
# → Schema validation miss > 2 weeks → critical
# → SQL samples miss > 2 months → warning
```

---

### Checklist triển khai

```
Phase 2 — Basic Freshness (1 tuần):
☐ Document change detector (file modified_at comparison)
☐ Incremental processor (diff + re-embed changed chunks)
☐ Cache invalidation khi doc thay đổi
☐ Daily cron job cho document sync
☐ Basic freshness metrics

Phase 3 — Full Lifecycle (2 tuần):
☐ Schema change detector + alert
☐ SQL sample validator
☐ Org chart sync → RBAC update
☐ Webhook integration (nếu doc store hỗ trợ)
☐ Freshness dashboard trong Grafana
```

---

### Liên quan

- Pipeline: [02_pipeline.md](./02_pipeline.md) — data feeding vào pipeline
- Security: [03_security_rbac.md](./03_security_rbac.md) — RBAC metadata khi re-embed
- Cost: [07_cost_optimization.md](./07_cost_optimization.md) — incremental vs full re-embed cost
