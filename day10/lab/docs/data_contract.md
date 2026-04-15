# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| policy_refund_v4 | CSV export (manual/scheduled) | Duplicate record, stale version, wrong refund window | duplicate_count, stale_chunk_detected, refund_window_mismatch |
| hr_leave_policy | CSV export (manual/scheduled) | Version conflict (10 vs 12 days), missing effective_date | version_conflict_count, missing_date_count |
| sla_p1_2026 | CSV export (manual/scheduled) | Missing/invalid SLA, wrong date format | missing_sla_count, invalid_date_count |
| it_helpdesk_faq | CSV export (manual/scheduled) | Missing chunk_text, invalid effective_date | missing_text_count, invalid_date_count |
| legacy_catalog_xyz_zzz | CSV export (manual/scheduled) | Unknown doc_id, fails expectation (length, schema) | unknown_docid_count, expectation_fail_count |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | … |
| doc_id | string | Có | … |
| chunk_text | string | Có | … |
| effective_date | date | Có | … |
| exported_at | datetime | Có | … |

---

## 3. Quy tắc quarantine vs drop

> Record bị flag đi đâu? Ai approve merge lại?

---

## 4. Phiên bản & canonical

> Source of truth cho policy refund: file nào / version nào?
