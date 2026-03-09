## Access Control & RBAC cho Enterprise RAG

---

### Tại sao đây là prerequisite, không phải "thêm sau"

Với 20K+ nhân viên và hàng trăm phòng ban, mỗi query đều tiềm ẩn rủi ro data leak:

```
Ví dụ thực tế:
- NV Marketing hỏi "Chính sách kỷ luật gần đây?" → RAG trả tài liệu HR nội bộ
- NV mới hỏi "Lương trung bình phòng kỹ thuật?" → SQL track trả data lương nhạy cảm
- NV bán hàng hỏi "Chiến lược Q4?" → RAG trả tài liệu C-level confidential
```

Không có RBAC = **một lần data leak có thể gây thiệt hại lớn hơn toàn bộ giá trị mà RAG mang lại**.

---

### Kiến trúc Access Control — 3 lớp

```
┌──────────────────────────────────────────────────────────────┐
│                        USER QUERY                             │
│                            │                                  │
│                    ┌───────▼────────┐                         │
│        LỚP 1:     │  Auth Gateway  │                         │
│    AUTHENTICATION  │                │                         │
│                    │  SSO / AD      │ ← Kết nối Active       │
│                    │  JWT Token     │   Directory / LDAP      │
│                    │  → user_id     │   (chắc chắn DN đã có) │
│                    │  → role        │                         │
│                    │  → department  │                         │
│                    │  → level       │                         │
│                    └───────┬────────┘                         │
│                            │                                  │
│                    ┌───────▼────────┐                         │
│        LỚP 2:     │ Permission     │                         │
│    AUTHORIZATION   │ Resolver       │                         │
│                    │                │                         │
│                    │ Input: role +  │                         │
│                    │   department   │                         │
│                    │                │                         │
│                    │ Output:        │                         │
│                    │   sql_tables[] │ ← Tables user được      │
│                    │   doc_scopes[] │   access                │
│                    │   data_level   │                         │
│                    └───────┬────────┘                         │
│                            │                                  │
│                    ┌───────▼────────┐                         │
│        LỚP 3:     │ Scoped RAG     │                         │
│    ENFORCEMENT     │ Pipeline       │                         │
│                    │                │                         │
│                    │ SQL: chỉ query │                         │
│                    │   tables trong │                         │
│                    │   sql_tables[] │                         │
│                    │                │                         │
│                    │ Doc: filter    │                         │
│                    │   chunks có    │                         │
│                    │   scope ∈      │                         │
│                    │   doc_scopes[] │                         │
│                    └────────────────┘                         │
└──────────────────────────────────────────────────────────────┘
```

---

### Lớp 1: Authentication — Định danh user

Doanh nghiệp 20K NV chắc chắn đã có hệ thống identity management. Chỉ cần tích hợp:

```python
# Middleware xác thực — chạy trước mọi query
class AuthMiddleware:
    def __init__(self, ad_client):
        self.ad = ad_client  # Active Directory / LDAP client
    
    def authenticate(self, request) -> UserContext:
        """Extract user info từ SSO token hoặc AD."""
        token = request.headers.get("Authorization")
        user = self.ad.verify_token(token)
        
        return UserContext(
            user_id=user.id,
            name=user.display_name,
            department=user.department,      # "Phòng Kỹ thuật"
            role=user.role,                  # "manager" | "staff" | "director"
            level=user.access_level,         # "public" | "internal" | "confidential"
            teams=user.team_memberships,     # ["team-backend", "project-alpha"]
        )
```

**Không tự build auth** — tích hợp với system đã có (Microsoft Entra ID, Okta, Google Workspace...).

---

### Lớp 2: Authorization — Xác định scope

```python
# Permission matrix — cấu hình 1 lần, maintain bởi admin
PERMISSION_MATRIX = {
    # role → (allowed_doc_scopes, allowed_sql_tables, max_data_level)
    
    "staff": {
        "doc_scopes": ["public", "{own_department}"],
        "sql_tables": ["products", "orders_summary", "general_reports"],
        "data_level": "internal",
    },
    
    "manager": {
        "doc_scopes": ["public", "{own_department}", "cross_department_summary"],
        "sql_tables": ["products", "orders", "revenue_by_dept", "team_metrics"],
        "data_level": "internal",
    },
    
    "hr_staff": {
        "doc_scopes": ["public", "hr_internal", "hr_policies"],
        "sql_tables": ["hr_records", "attendance", "leave_requests", "salary_bands"],
        "data_level": "confidential",
    },
    
    "director": {
        "doc_scopes": ["public", "all_departments", "executive"],
        "sql_tables": ["*"],  # Full access
        "data_level": "confidential",
    },
}

class PermissionResolver:
    def resolve(self, user: UserContext) -> AccessScope:
        perms = PERMISSION_MATRIX[user.role]
        
        # Thay thế {own_department} bằng department thực tế
        doc_scopes = [
            s.replace("{own_department}", user.department) 
            for s in perms["doc_scopes"]
        ]
        
        return AccessScope(
            doc_scopes=doc_scopes,
            sql_tables=perms["sql_tables"],
            max_data_level=perms["data_level"],
        )
```

**Nguyên tắc: Start restrictive, mở rộng dần.** Mặc định deny, chỉ allow những gì được config rõ ràng.

---

### Lớp 3: Enforcement — Filter trước khi search

**Quan trọng nhất: Filter TRƯỚC search, không phải filter kết quả.**

#### Doc RAG — RBAC tại ingestion time + retrieval time

```python
# ========== INGESTION TIME: Gắn metadata khi embed ==========

def ingest_document(doc, department, access_level, confidentiality):
    chunks = chunker.split(doc)
    
    for chunk in chunks:
        # Metadata RBAC gắn vào mỗi chunk
        metadata = {
            "department": department,           # "hr", "engineering", "sales"
            "access_level": access_level,       # "public", "internal", "confidential"
            "confidentiality": confidentiality,  # "open", "restricted", "secret"
            "doc_id": doc.id,
            "doc_title": doc.title,
            "ingested_at": datetime.now().isoformat(),
        }
        
        embedding = embed_model.encode(chunk.text)
        vector_store.upsert(
            id=chunk.id,
            vector=embedding,
            metadata=metadata,
        )

# ========== RETRIEVAL TIME: Filter theo user scope ==========

def retrieve_with_rbac(query: str, user: UserContext, scope: AccessScope):
    """Luôn filter TRƯỚC similarity search."""
    
    query_embedding = embed_model.encode(query)
    
    # Qdrant filter example
    results = vector_store.search(
        query_vector=query_embedding,
        query_filter=Filter(
            must=[
                # Chỉ chunks thuộc scope của user
                FieldCondition(
                    key="department",
                    match=MatchAny(any=scope.doc_scopes),
                ),
                # Chỉ chunks <= data level của user
                FieldCondition(
                    key="access_level",
                    match=MatchAny(any=get_allowed_levels(scope.max_data_level)),
                ),
            ]
        ),
        limit=10,
    )
    
    return results
```

#### SQL Track — Scope tables + validate

```python
def generate_sql_with_rbac(query: str, user: UserContext, scope: AccessScope):
    """Chỉ expose schema của tables trong scope."""
    
    # 1. Filter schema — chỉ giữ tables user được access
    allowed_schema = {
        table: schema[table] 
        for table in schema 
        if table in scope.sql_tables or scope.sql_tables == ["*"]
    }
    
    # 2. Generate SQL với schema bị giới hạn
    generated_sql = sql_generator.generate(
        query=query,
        schema=allowed_schema,  # LLM chỉ thấy tables trong scope
        few_shot_examples=get_scoped_examples(scope),
    )
    
    # 3. Validate — double check không access table ngoài scope
    validation = validate_sql_rbac(generated_sql, scope)
    if not validation.passed:
        return ErrorResponse(
            message="Bạn không có quyền truy cập dữ liệu này. "
                    "Liên hệ quản lý để được cấp quyền.",
            blocked_tables=validation.blocked_tables,
        )
    
    return execute_sql(generated_sql)
```

---

### Xử lý edge case quan trọng

#### 1. Cross-department queries

```
"So sánh doanh thu phòng A và phòng B"
→ User phải có scope cả 2 phòng, nếu không → từ chối một phần hoặc toàn bộ
```

```python
def check_cross_dept_access(query_intent, user_scope):
    required_depts = extract_departments(query_intent)
    allowed = all(dept in user_scope.doc_scopes for dept in required_depts)
    
    if not allowed:
        missing = [d for d in required_depts if d not in user_scope.doc_scopes]
        return AccessDenied(
            message=f"Bạn không có quyền truy cập dữ liệu phòng {', '.join(missing)}. "
                    f"Bạn có thể xem dữ liệu phòng {user_scope.doc_scopes}."
        )
```

#### 2. PII Masking trong response

Khi answer chứa thông tin cá nhân (tên, SĐT, email), và user không có quyền xem:

```python
PII_PATTERNS = {
    "phone": r"\d{10,11}",
    "email": r"[\w.-]+@[\w.-]+\.\w+",
    "id_number": r"\d{9,12}",
}

def mask_pii(response: str, user_scope: AccessScope) -> str:
    if user_scope.max_data_level != "confidential":
        for pattern_type, pattern in PII_PATTERNS.items():
            response = re.sub(pattern, f"[{pattern_type} - ẩn]", response)
    return response
```

#### 3. Audit Trail — bắt buộc cho compliance

```python
def log_query(user, query, route, response, accessed_sources):
    audit_logger.log({
        "timestamp": datetime.now().isoformat(),
        "user_id": user.id,
        "department": user.department,
        "role": user.role,
        "query": query,
        "route": route,                    # "sql" | "doc" | "hybrid"
        "accessed_tables": accessed_sources.get("tables", []),
        "accessed_docs": accessed_sources.get("doc_ids", []),
        "response_truncated": response[:200],
        "access_denied": response.is_denied if hasattr(response, 'is_denied') else False,
    })
```

Mọi query phải được log — không chỉ để audit mà còn để phát hiện pattern truy cập bất thường.

---

### Checklist triển khai RBAC

```
Phase 1 — MVP (2 tuần song song với RAG pipeline):
☐ Tích hợp SSO/AD — lấy user_id, role, department
☐ Định nghĩa Permission Matrix cho 4-5 roles chính
☐ Gắn RBAC metadata vào mỗi chunk khi embed
☐ Filter theo scope TRƯỚC similarity search
☐ SQL track: chỉ expose tables trong scope
☐ Audit log cho mọi query

Phase 2 — Nâng cao (4 tuần, sau khi pilot):
☐ PII masking trong response
☐ Cross-department access logic
☐ Admin dashboard — manage permissions
☐ Anomaly detection — phát hiện truy cập bất thường
☐ Data classification tool — tự động tag access_level cho docs mới
```

---

### Liên quan

- Pipeline tổng thể: [02_pipeline.md](./02_pipeline.md)
- Monitoring: [04_observability.md](./04_observability.md) — track access denied events
- Data lifecycle: [05_data_lifecycle.md](./05_data_lifecycle.md) — RBAC metadata khi re-embed
