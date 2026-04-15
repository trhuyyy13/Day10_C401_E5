# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Data Observability

**Họ và tên:** Vu Hai Dang - 2A202600339  
**Vai trò:** Documentation / Monitoring / Eval  
**Ngày nộp:** 15/04/2026  
**Độ dài:** ~520 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong Day 10, tôi tập trung vào ba cụm việc: chỉnh tài liệu kiến trúc pipeline, sửa script đánh giá retrieval, và chạy thử dữ liệu raw theo nhiều cấu hình để quan sát output. Hai file tôi chỉnh trực tiếp là `docs/pipeline_architecture.md` và `eval_retrieval.py`. Với dữ liệu nguồn, tôi dùng `data/raw/policy_export_dirty.csv` để chạy các run khác nhau rồi đối chiếu manifest, freshness, và file eval. Tôi phối hợp với phần embed bằng cách xác nhận collection có dữ liệu trước khi eval, sau đó xuất `artifacts/eval/before_after_eval.csv` làm bằng chứng cho report nhóm/cá nhân. Công việc của tôi nghiêng về tính đúng của luồng dữ liệu và tính nhất quán giữa pipeline chạy thật với kết quả đo chất lượng.

**Bằng chứng thao tác:**
- `python etl_pipeline.py run --run-id ci-smoke2`
- `python etl_pipeline.py run --run-id freshness-utc-now`
- `python etl_pipeline.py run --run-id rerun-a`
- `python eval_retrieval.py --out artifacts/eval/before_after_eval.csv`

Commit: 1dfeab9a7cd1e94988eebada86b4c66d4492239e - `docs/pipeline_architecture.md` và `eval_retrieval.py`

Commit: 5b8bb98e7dd44bef310ed0fe3e8720ec1802b056 - dùng `data/raw/policy_export_dirty.csv` để chạy các run khác nhau rồi đối chiếu manifest, freshness, và file eval.
---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật quan trọng nhất của tôi là **đồng bộ cơ chế embedding giữa ETL và script eval**. Ban đầu `eval_retrieval.py` luôn khởi tạo SentenceTransformer theo `EMBEDDING_MODEL`, trong khi collection `day10_kb` đã được publish bằng OpenAI embedding (`text-embedding-3-small`). Kết quả là eval lỗi ngay ở bước load model và không tạo được CSV. Tôi sửa script theo hướng giống pipeline: đọc `EMBEDDING_PROVIDER`; nếu là `openai` (hoặc model dạng `text-embedding-*`) thì dùng OpenAI embedding function, còn local mới dùng SentenceTransformer. Quyết định này giúp test bám đúng môi trường chạy thật, tránh trường hợp “pipeline pass nhưng eval fail vì config lệch”, và đặc biệt hữu ích khi thử nhiều cấu hình của cùng file `policy_export_dirty.csv`.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Anomaly chính tôi xử lý là freshness fail do dữ liệu nguồn cũ. Khi chạy kiểm tra trên `manifest_ci-smoke2.json`, kết quả trả về: `FAIL {... "age_hours": 126.635, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}`. Triệu chứng này cho thấy bản ghi mới nhất (`latest_exported_at`) đã quá hạn SLA, dù các expectation chất lượng khác vẫn không vi phạm nghiêm trọng. Tôi xác nhận nguyên nhân nằm ở timestamp export chứ không phải lỗi transform hay embed. Sau khi cập nhật exported_at và chạy lại run `freshness-utc-now`, lệnh freshness trả về `PASS {... "age_hours": 5.764, "sla_hours": 24.0}`. Tôi cũng đối chiếu log `run_freshness-utc-now.log` để chắc rằng expectation vẫn ổn định (`refund_no_stale_14d_window OK`, `effective_date_iso_yyyy_mm_dd OK`) trước khi xuất kết quả eval.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**run_id trước:** `ci-smoke2`  
**run_id sau:** `freshness-utc-now`

**Freshness trước/sau (tương đương before/after):**

- Trước: `FAIL` với `age_hours=126.635` và `sla_hours=24.0` (manifest `ci-smoke2`).
- Sau: `PASS` với `age_hours=5.764` và `sla_hours=24.0` (manifest `freshness-utc-now`).

**2 dòng từ `before_after_eval.csv`:**

- `gq_d10_01,"Theo policy hoàn tiền nội bộ, khách có tối đa bao nhiêu ngày làm việc để gửi yêu cầu hoàn tiền sau khi đơn được xác nhận?",policy_refund_v4,Yêu cầu hoàn tiền được chấp nhận trong vòng 7 ngày làm việc kể từ xác nhận đơn (ghi chú: bản sync cũ policy-v3 — lỗi migration). [cleaned: stale_refund_window],yes,no,,3`

- `gq_d10_03,"Theo chính sách nghỉ phép hiện hành (2026), nhân viên dưới 3 năm kinh nghiệm được bao nhiêu ngày phép năm?",hr_leave_policy,Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026.,yes,no,yes,3`

Hai cụm bằng chứng trên cho thấy sau khi xử lý freshness, bộ retrieval vẫn giữ chất lượng tốt trên các câu hỏi vàng của lab.


---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ đưa rule cutoff ngày hiệu lực ra contract (đọc từ `contracts/data_contract.yaml`) thay vì hard-code trong cleaning logic, đồng thời bổ sung một expectation chuyên phát hiện marker migration cũ trong text (ví dụ cụm `policy-v3`) để giảm nhiễu ngữ nghĩa khi synthesis trích bằng chứng từ top-k.
