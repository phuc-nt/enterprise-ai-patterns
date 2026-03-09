## Adoption & Rollout Strategy

---

### Tại sao adoption strategy quan trọng ngang kỹ thuật

Thống kê industry: **70% dự án AI nội bộ thất bại không phải vì kỹ thuật kém mà vì không ai dùng.**

Với 20K nhân viên:
- Không ai biết hệ thống tồn tại → không dùng
- Hệ thống trả sai 2-3 lần đầu → mất trust vĩnh viễn
- Không có training → dùng sai cách → kết quả tệ → blame tool
- Không có feedback loop → team kỹ thuật không biết cải thiện gì

**Kỹ thuật tốt + adoption tệ = thất bại.**

---

### Lộ trình Rollout — 4 giai đoạn

```
┌─────────────────────────────────────────────────────────────────┐
│                     ROLLOUT TIMELINE                              │
│                                                                   │
│  Phase A          Phase B           Phase C          Phase D      │
│  PILOT            CONTROLLED        GENERAL          OPTIMIZE     │
│  20-50 users      200-500 users     5K-20K users     Ongoing      │
│  1 phòng ban      3-5 phòng ban     Toàn công ty     Continuous   │
│                                                                   │
│  ├─── 4-6 tuần ──├─── 6-8 tuần ──├─── 4-6 tuần ──├─── ∞        │
│                                                                   │
│  Pipeline:        Pipeline:         Pipeline:        Pipeline:    │
│  Phase 1-2        Phase 2-3         Phase 3-4        Phase 4-5   │
└─────────────────────────────────────────────────────────────────┘
```

---

### Phase A: Pilot (4-6 tuần, 20-50 users)

#### Chọn phòng ban pilot

```
Tiêu chí chọn:
✓ Có pain point rõ ràng với data access (hỏi MIS/IT hàng ngày)
✓ Có leader ủng hộ AI (sponsor)
✓ Nhân viên có digital literacy trung bình trở lên
✓ Data/tài liệu của phòng đã tương đối sạch
✓ Phòng đủ lớn để có dữ liệu hỏi đa dạng (20-50 người)

Phòng thường phù hợp nhất:
- Operations / Vận hành (Uber chọn phòng này, giảm 18% productivity)
- Sales / Kinh doanh (hỏi sản phẩm, giá, chính sách thường xuyên)
- Customer Service (hỏi quy trình, chính sách để trả lời khách)
```

#### Champion Program

```
Chọn 2-3 "Champions" trong phòng pilot:
- Người hiểu domain tốt + open với công nghệ mới
- Vai trò:
  1. Training đồng nghiệp cách dùng
  2. Collect feedback hàng tuần
  3. Tạo golden test cases từ real questions
  4. Cầu nối giữa team kỹ thuật và users
  
Incentive:
- Early access to features
- "AI Champion" title trong org
- Input trực tiếp vào product direction
```

#### Onboarding cho pilot users

```
Tuần 1: Kickoff session (30 phút)
  - Demo live: 5 use cases thực tế từ domain của phòng
  - Set expectations: "Đây là MVP, sẽ sai — feedback giúp nó tốt hơn"
  - Hướng dẫn feedback: khi nào bấm 👍/👎, khi nào report issue
  
Tuần 2-4: Daily standup 5 phút với Champions
  - "Hôm qua hỏi gì? Trả lời đúng không? Có issue gì?"
  - Bug/feedback → Jira backlog ngay trong ngày

Tuần 5-6: Review & quyết định
  - Eval metrics đạt threshold chưa?
  - User satisfaction > 60%?
  - Nếu YES → Phase B
  - Nếu NO → iterate thêm 2-4 tuần, fix top issues
```

#### Metrics Phase A

| Metric | Target để sang Phase B | Cách đo |
|--------|----------------------|---------|
| Daily Active Users | > 50% pilot users dùng hàng ngày | Analytics |
| User Satisfaction | > 60% 👍 rate | In-app feedback |
| Query Success Rate | > 70% queries trả lời hữu ích | Champion review |
| Response Accuracy | > 80% trên golden test set | Eval harness |
| Time Saved | > 20% so với workflow cũ | Survey + observation |

---

### Phase B: Controlled Rollout (6-8 tuần, 200-500 users)

#### Mở rộng chiến lược

```
Chọn thêm 2-4 phòng ban:
- Ưu tiên phòng có data overlap với pilot (vd: Sales → Marketing)
- Mỗi phòng mới có 1-2 Champions (đào tạo bởi Champions Phase A)
- Mỗi phòng thêm = thêm Workspace + curated SQL samples + doc corpus

Timeline:
  Tuần 1: Onboard phòng 2 (Champions từ Phase A hỗ trợ)
  Tuần 3: Onboard phòng 3
  Tuần 5: Onboard phòng 4 (nếu metrics ổn)
  Tuần 6-8: Stabilize + optimize
```

#### Training Program

```
Tạo training materials chuẩn:

1. Quick Start Guide (1 trang):
   - Cách access hệ thống
   - 5 loại câu hỏi có thể hỏi (+ ví dụ cụ thể)
   - 3 loại câu hỏi KHÔNG nên hỏi (+ lý do)
   - Cách bấm feedback

2. FAQ Document:
   - "Tại sao trả lời sai?" — Giải thích AI giới hạn
   - "Data có up-to-date không?" — Giải thích data freshness
   - "Ai xem được câu hỏi của tôi?" — Giải thích privacy
   - "Tôi muốn hệ thống biết thêm về X" — Hướng dẫn request
   
3. Video Tutorial (3-5 phút):
   - Screen recording: hỏi câu hỏi thực tế, show kết quả
   - Highlight: citation/nguồn, confidence indicator, feedback
```

#### Feedback Loop mạnh hơn

```python
# Automated weekly report gửi team kỹ thuật

def weekly_adoption_report():
    return {
        "active_users": count_active_users(period="7d"),
        "queries_total": count_queries(period="7d"),
        "queries_per_user": queries_total / active_users,
        
        "satisfaction": {
            "thumbs_up_rate": get_thumbs_up_rate(period="7d"),
            "thumbs_down_queries": get_thumbs_down_queries(top=10),  # Top 10 fails
        },
        
        "by_department": {
            dept: {
                "users": count_active_users(dept=dept),
                "satisfaction": get_satisfaction(dept=dept),
                "top_queries": get_top_queries(dept=dept, top=5),
            }
            for dept in active_departments
        },
        
        "action_items": [
            f"Thêm {len(new_golden_from_fails)} test cases từ fails",
            f"Phòng {worst_dept} cần attention: {worst_dept_score}% satisfaction",
        ],
    }
```

#### Metrics Phase B

| Metric | Target để sang Phase C |
|--------|----------------------|
| DAU/total enrolled | > 40% |
| User Satisfaction | > 70% 👍 rate |
| Cross-department queries working | > 80% |
| Eval harness score | > 85% |
| Support tickets/week | < 10 |

---

### Phase C: General Availability (4-6 tuần, 5K-20K users)

#### Rollout toàn công ty

```
Chiến lược: Big Bang + Safety Net

1. Announce toàn công ty:
   - Email từ CTO/CDO/sponsor cấp cao
   - Internal blog post với success stories từ pilot
   - Live demo event (recorded)

2. Self-service onboarding:
   - Video tutorial accessible 24/7
   - In-app "Getting Started" wizard
   - Contextual tooltips cho first-time users
   
3. Safety net:
   - FAQ chatbot (dùng chính RAG system)
   - Slack channel #ai-assistant-support
   - Champions network (1 champion per 50-100 users)
   - Weekly office hours với team kỹ thuật (Q&A)
```

#### Scaling Considerations

```
20K users đồng thời:
- Estimate: 5-10% online cùng lúc = 1000-2000 concurrent
- Peak queries: 50-100 queries/phút
- Cần:
  ☐ Load balancer cho API
  ☐ Horizontal scaling cho retrieval service
  ☐ Rate limiting per user (prevent abuse)
  ☐ Queue system cho peak (đừng để user thấy timeout)
  ☐ Circuit breaker khi LLM API overwhelmed
```

---

### Phase D: Continuous Optimization (ongoing)

```
Monthly:
☐ Review top 20 fail queries → thêm golden test cases
☐ Review usage by department → identify depts chưa adopt
☐ ROI report cho stakeholders

Quarterly:
☐ User survey (NPS score)
☐ Feature request prioritization
☐ Eval harness expansion (thêm test cases mới)
☐ Data quality review

Annual:
☐ Full system audit (security, performance, accuracy)
☐ Technology refresh (new models, new tools)
☐ Training refresh cho org
```

---

### Xử lý resistance — Các phản đối thường gặp

| Phản đối | Cách xử lý |
|----------|-----------|
| *"AI sẽ thay thế tôi"* | "AI là công cụ bổ sung, giúp bạn tập trung vào việc quan trọng hơn. Bạn là expert, AI là assistant." |
| *"Data của tôi bị lộ?"* | Giải thích RBAC, show audit log, cam kết privacy policy rõ ràng |
| *"Nó trả lời sai hoài"* | Acknowledge, hỏi ví dụ cụ thể, thêm vào eval harness, báo khi đã fix |
| *"Tôi không có thời gian học"* | Quick start = 5 phút, hầu hết chỉ cần type câu hỏi như Google |
| *"Sếp bắt dùng nhưng không hữu ích"* | Tìm 1 use case cụ thể giúp họ save time, tạo habit từ đó |

---

### ROI Framework cho Stakeholders

```
Đầu vào (đo trước khi deploy):
  T₀ = Thời gian trung bình để tìm thông tin (phút)
  N  = Số lượng queries/ngày (estimate từ MIS/IT tickets)
  S  = Số nhân viên liên quan
  H  = Chi phí/giờ trung bình nhân viên

Sau deploy:
  T₁ = Thời gian trung bình với RAG (phút)
  
ROI = (T₀ - T₁) × N × S × H × 252 ngày/năm
                    vs
      Chi phí LLM + Infrastructure + Đội ngũ

Ví dụ Uber:
  T₀ = 10 phút
  T₁ = 3 phút  
  → Tiết kiệm 7 phút × queries → 18% daily productivity gain

Ví dụ cho DN 20K NV (estimate):
  T₀ = 15 phút (estimate)
  T₁ = 5 phút (conservative)
  N  = 500 queries/ngày (1K active users × 0.5 queries/user)
  H  = 200K VND/giờ (~ $8/h)
  
  Saving = 10/60 × 500 × 200K × 252 = ~4.2 tỷ VND/năm
  
  Cost estimate:
  LLM API: ~$50-100/ngày = ~1 tỷ VND/năm
  Infra: ~500 triệu VND/năm
  Team (2 engineers): ~1.2 tỷ VND/năm
  
  Net saving: ~1.5 tỷ VND/năm + intangible benefits
```

---

### Checklist triển khai

```
Phase A — Pilot (song song Phase 1 kỹ thuật):
☐ Chọn phòng ban pilot
☐ Recruit 2-3 Champions
☐ Tạo 30 golden test cases từ domain pilot
☐ Kickoff session + demo
☐ Daily standup với Champions (đầu)
☐ In-app feedback (thumbs up/down)
☐ Weekly report template

Phase B — Controlled Rollout:
☐ Training materials (Quick Start, FAQ, Video)
☐ Champion network mở rộng
☐ Cross-department onboarding
☐ Automated weekly adoption report
☐ Support channel (Slack/Teams)

Phase C — GA:
☐ Company-wide announcement
☐ Self-service onboarding flow
☐ Scaling infrastructure
☐ Rate limiting + abuse prevention
☐ ROI tracking dashboard
```

---

### Liên quan

- Pipeline: [02_pipeline.md](./02_pipeline.md) — timeline aligns với pipeline phases
- Security: [03_security_rbac.md](./03_security_rbac.md) — privacy messaging cho users
- Observability: [04_observability.md](./04_observability.md) — user satisfaction metrics
