# DATA CONTRACT: KB_CHUNK_EXPORT

## 1. Thông tin chung (Overview)

- **Dataset:** `kb_chunk_export`
- **Phiên bản:** `1.0`
- **Đội ngũ quản lý (Owner):** `AI-Data-Engineering`
- **Kênh thông báo cảnh báo:** `#alerts-data-quality`

---

## 1. Nguồn dữ liệu (source map)

| Nguồn                  | Phương thức ingest            | Failure mode chính                                       | Metric / alert                                                |
| ---------------------- | ----------------------------- | -------------------------------------------------------- | ------------------------------------------------------------- |
| policy_refund_v4       | CSV export (manual/scheduled) | Duplicate record, stale version, wrong refund window     | duplicate_count, stale_chunk_detected, refund_window_mismatch |
| hr_leave_policy        | CSV export (manual/scheduled) | Version conflict (10 vs 12 days), missing effective_date | version_conflict_count, missing_date_count                    |
| sla_p1_2026            | CSV export (manual/scheduled) | Missing/invalid SLA, wrong date format                   | missing_sla_count, invalid_date_count                         |
| it_helpdesk_faq        | CSV export (manual/scheduled) | Missing chunk_text, invalid effective_date               | missing_text_count, invalid_date_count                        |
| legacy_catalog_xyz_zzz | CSV export (manual/scheduled) | Unknown doc_id, fails expectation (length, schema)       | unknown_docid_count, expectation_fail_count                   |

## 2. Đặc tả dữ liệu (Schema Specification)

Dữ liệu sau khi làm sạch (Cleaned Layer) phải tuân thủ cấu trúc sau:

| Trường dữ liệu   | Kiểu dữ liệu | Bắt buộc? | Mô tả                                                                  |
| :--------------- | :----------- | :-------: | :--------------------------------------------------------------------- |
| `chunk_id`       | String       |    Yes    | ID duy nhất định danh mẩu tin (thường là Hash).                        |
| `doc_id`         | String       |    Yes    | Mã tài liệu nguồn (Allowlist: policy_refund_v4, hr_leave_policy, ...). |
| `chunk_text`     | String       |    Yes    | Nội dung văn bản (Tối thiểu 8 ký tự).                                  |
| `effective_date` | Date         |    Yes    | Ngày hiệu lực của chính sách (Định dạng YYYY-MM-DD).                   |
| `exported_at`    | DateTime     |    Yes    | Thời điểm dữ liệu được xuất khỏi hệ thống nguồn.                       |

---

## 3. Quy tắc chất lượng (Quality Rules/Expectations)

Hệ thống ETL sẽ tự động từ chối (Halt) hoặc cảnh báo (Warn) dựa trên các quy tắc:

- **R01 (HALT):** Không được chứa "14 ngày" trong văn bản của `policy_refund_v4` (Đảm bảo tính chính xác của luật mới).
- **R02 (HALT):** Ngày hiệu lực phải đúng định dạng ISO.
- **R03 (WARN):** Cảnh báo nếu văn bản chứa các ký tự đặc biệt nghi vấn ($ % ^ \* ~).
- **R04 (WARN):** Cảnh báo nếu văn bản quá ngắn (dưới 8 ký tự).

---

## 4. Cam kết độ tươi (Freshness SLA)

- **Điểm đo lường:** Tại thời điểm xuất bản (Publish).
- **Thời gian cam kết (SLA):** **24 giờ**.
- **Hành động khi vi phạm:** Bắn alert tới kênh quản lý và đánh dấu `FAIL` trong freshness check.

---

## 5. Nguồn dữ liệu chuẩn (Canonical Sources)

Hệ thống AI Assistant sẽ chỉ tiêu thụ dữ liệu từ các nguồn đã được phê duyệt sau:

1. `policy_refund_v4`: Chính sách hoàn tiền phiên bản 4.
2. `sla_p1_2026`: Quy định xử lý sự cố P1 năm 2026.
3. `it_helpdesk_faq`: Câu hỏi thường gặp bộ phận IT.
4. `hr_leave_policy`: Chính sách nghỉ phép và làm việc từ xa.
5. `access_control_sop`: Quy trình kiểm soát truy cập hệ thống.
