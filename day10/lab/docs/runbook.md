# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

User / agent trả lời đúng ở top-1 nhưng vẫn còn chunk stale trong top-k, ví dụ refund window có lúc lộ nội dung 14 ngày làm việc thay vì chuẩn 7 ngày. Ngoài ra, raw export có thể bị thiếu `exported_at` hoặc chứa ký tự bất thường trong `chunk_text`, làm quality gate fail trước khi embed. Trong thực tế pipeline còn có thể báo freshness FAIL dù ingestion chạy xong vì dữ liệu nguồn đã quá SLA.

---

## Detection

Detection dựa trên 3 tín hiệu chính: freshness check từ manifest, expectation fail trong log pipeline, và cột hits_forbidden trong CSV eval. Với run hiện tại, freshness_check=FAIL vì latest_exported_at=2026-04-10T08:00:00 trong khi SLA là 24h; với inject-bad, expectation refund_no_stale_14d_window fail và q_refund_window trong eval chuyển sang hits_forbidden=yes.
Detection dựa trên 3 tín hiệu chính: freshness check từ manifest, expectation fail trong log pipeline, và cột hits_forbidden trong CSV eval. Với run hiện tại, freshness_check=FAIL vì latest_exported_at=2026-04-10T08:00:00 trong khi SLA là 24h; với inject-bad, expectation refund_no_stale_14d_window fail và q_refund_window trong eval chuyển sang hits_forbidden=yes. Nếu raw có dòng thiếu `exported_at` hoặc chunk_text chứa ký tự bất thường, các expectation mới `no_missing_exported_at` và `no_special_char_in_chunk_text` sẽ báo trước khi publish.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Xác nhận run_id, raw_records=10, cleaned_records=6, quarantine_records=4, và các cờ no_refund_fix / skipped_validate. |
| 2 | Mở `artifacts/quarantine/*.csv` | Thấy các record bị loại theo reason như duplicate_chunk_text, missing_effective_date, stale_hr_policy_effective_date, unknown_doc_id; nếu có dòng thiếu export time hoặc text bất thường thì sẽ thấy thêm `missing_exported_at` / `suspicious_special_char_in_chunk_text`. |
| 3 | Chạy `python eval_retrieval.py` | So sánh before/after trên q_refund_window và q_leave_version; after inject-bad thì q_refund_window phải chuyển sang hits_forbidden=yes. |

---

## Mitigation

Rerun pipeline với flag đúng, bỏ inject-bad, và publish lại collection để prune vector cũ. Nếu freshness FAIL vẫn còn thì banner dữ liệu stale ở tầng serving và chặn dùng cho demo/agent cho tới khi có manifest mới đạt SLA. Với run demo, tránh dùng --skip-validate ngoài mục đích Sprint 3.

---

## Prevention

Thêm expectation mới hoặc nâng severity lên halt cho các rule dễ gây stale context; ghi owner rõ cho ingest, quality, embed, monitoring; và theo dõi một đường cảnh báo freshness. Với code hiện tại, nên coi `no_missing_exported_at` và `no_special_char_in_chunk_text` là guardrail đầu vào, còn `refund_no_stale_14d_window` là guardrail publish. Nếu lớp sang Day 11, nên nối sang alert cho manifest FAIL và hits_forbidden tăng.
