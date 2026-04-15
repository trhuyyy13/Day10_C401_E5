# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| ___ | Ingestion / Raw Owner | ___ |
| ___ | Cleaning & Quality Owner | ___ |
| ___ | Embed & Idempotency Owner | ___ |
| ___ | Monitoring / Docs Owner | ___ |

**Ngày nộp:** ___________  
**Repo:** ___________  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**

Pipeline dùng export mẫu `data/raw/policy_export_dirty.csv` làm raw đầu vào. Dữ liệu đi qua `etl_pipeline.py run`, được clean theo allowlist `doc_id`, chuẩn hoá `effective_date`, quarantine các dòng lạ/thiếu/ngày cũ của HR, sửa stale refund 14→7, rồi embed vào Chroma collection `day10_kb`. `run_id` được ghi trực tiếp trong manifest và log, ví dụ run chuẩn là `2026-04-15T07-45Z` và run inject là `inject-bad`.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**

`python etl_pipeline.py run`

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| refund 14→7 fix + refund_no_stale_14d_window | before: q_refund_window contains_expected=yes, hits_forbidden=no | after inject-bad: q_refund_window contains_expected=yes, hits_forbidden=yes | artifacts/eval/before_after_eval.csv; artifacts/eval/after_inject_bad.csv; artifacts/logs/run_2026-04-15T07-45Z.log |
| stale HR policy quarantine + hr_leave_no_stale_10d_annual | cleaned_records=6, quarantine_records=4 | q_leave_version top1_doc_expected=yes, hits_forbidden=no | artifacts/quarantine/quarantine_2026-04-15T07-45Z.csv; artifacts/eval/before_after_eval.csv |
| allowlist doc_id + unknown_doc_id quarantine | 1 record lạ bị loại | quarantine reason unknown_doc_id xuất hiện trong quarantine CSV | artifacts/quarantine/quarantine_2026-04-15T07-45Z.csv |
| effective_date ISO + missing/invalid date quarantine | 1 record thiếu effective_date bị loại | non_iso_rows=0 sau clean | artifacts/logs/run_2026-04-15T07-45Z.log; artifacts/manifests/manifest_2026-04-15T07-45Z.json |

Hai guardrail mới trong code hiện tại là `no_missing_exported_at` và `no_special_char_in_chunk_text`, kèm rule chuẩn hoá viết hoa câu trong cleaned text. Bộ mẫu hiện tại chưa kích hoạt các guardrail này nên nhóm không đưa chúng vào bảng delta để tránh ghi số liệu giả; chúng vẫn được liệt kê dưới phần rule chính và trong code review.

**Rule chính (baseline + mở rộng):**

- Allowlist `doc_id` để chặn export lạ.
- Quarantine record thiếu `exported_at` để tránh publish dữ liệu chưa hoàn chỉnh.
- Quarantine chunk_text có ký tự bất thường để chặn raw noise.
- Chuẩn hoá `effective_date` sang ISO và quarantine nếu rỗng hoặc sai format.
- Loại HR cũ khi `effective_date < 2026-01-01`.
- Sửa refund window 14→7 để tránh context stale.
- Dedupe theo `chunk_text` để tránh chunk lặp trong index.
- Chuẩn hoá viết hoa đầu câu để giảm noise ở cleaned text.

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

Trong run inject-bad, expectation `refund_no_stale_14d_window` FAIL vì `--no-refund-fix` để lại câu 14 ngày. Cách xử lý là rerun pipeline chuẩn không dùng `--skip-validate`, publish lại index, rồi kiểm tra lại bằng `eval_retrieval.py`; sau khi sửa, `q_refund_window` không còn hits_forbidden.

Hai expectation mới `no_missing_exported_at` và `no_special_char_in_chunk_text` là guardrail đầu vào. Chúng chưa fail trên bộ mẫu hiện tại, nhưng đã được thêm để chặn những raw export bẩn trong các run sau.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

Nhóm dùng kịch bản Sprint 3 là `--no-refund-fix --skip-validate` để cố ý giữ chunk refund stale trong collection. Kết quả là record clean vẫn có 6 dòng nhưng index giữ lại cả câu 7 ngày và 14 ngày, nên truy hồi ở câu `q_refund_window` vẫn đúng top-1 nhưng lộ stale chunk trong top-k.

**Kết quả định lượng (từ CSV / bảng):**

Before clean: `artifacts/eval/before_after_eval.csv` cho thấy `q_refund_window contains_expected=yes, hits_forbidden=no` và `q_leave_version contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes`. After inject-bad: `artifacts/eval/after_inject_bad.csv` cho thấy `q_refund_window contains_expected=yes, hits_forbidden=yes`, còn `q_leave_version` vẫn ổn. Đây là bằng chứng before/after rõ nhất cho ảnh hưởng của stale chunk lên retrieval.

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

Nhóm đang dùng SLA freshness 24 giờ. PASS nghĩa là exported_at còn trong SLA và có thể phục vụ cho retrieval/agent; WARN nghĩa là gần ngưỡng hoặc cần chú ý nhưng chưa chặn; FAIL nghĩa là quá hạn và nên coi dữ liệu stale. Với manifest hiện tại, freshness_check=FAIL vì exported_at là 2026-04-10T08:00:00, tức khoảng 120 giờ trước thời điểm run, nên không nên dùng làm nguồn tin cậy cho agent.

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

Dữ liệu sau embed có thể phục vụ lại Day 09 vì cùng narrative CS + IT Helpdesk, nhưng nhóm đang tách collection `day10_kb` để giữ ranh giới publish và tránh trộn vector cũ với corpus Day 09. Nếu cần tích hợp lại cho multi-agent Day 09, có thể dùng cùng nguồn docs nhưng vẫn phải rerun pipeline Day 10 trước khi cập nhật collection dùng chung.

---

## 6. Rủi ro còn lại & việc chưa làm

- Freshness hiện vẫn FAIL theo SLA 24h, nên artifact này chỉ phù hợp demo hoặc grading nội bộ nếu có ghi chú rõ.
- Inject-bad cho thấy stale chunk có thể sống sót trong top-k dù top-1 đúng, nên cần theo dõi hits_forbidden chứ không chỉ top-1.
- Nếu lớp yêu cầu báo cáo cá nhân, mỗi người vẫn cần file riêng ở `reports/individual/*.md` với run_id và số liệu thật.
