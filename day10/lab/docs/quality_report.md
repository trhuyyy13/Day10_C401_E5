# BÁO CÁO CHÍNH SÁCH CHẤT LƯỢNG DỮ LIỆU (DATA QUALITY REPORT)

## 1. Thông tin tổng quan (Run Summary)

- **Mã định danh (Run ID):** `inject-bad`
- **Thời điểm thực thi:** `2026-04-15 09:46:27 UTC`
- **Nguồn dữ liệu:** `data/raw/policy_export_dirty.csv`
- **Chỉ số Ingestion:**
  - Tổng số bản ghi thô: `10`
  - Số bản ghi sạch (Cleaned): `6`
  - Số bản ghi bị loại (Quarantine): `4`

---

## 2. Kiểm định chất lượng (Data Quality Audit)

Dưới đây là kết quả từ bộ quy tắc kiểm định (Expectations) trước khi nạp vào AI:

| Tên bài test                    | Mức độ | Trạng thái | Chi tiết lỗi                |
| :------------------------------ | :----: | :--------: | :-------------------------- |
| `no_missing_exported_at`        |  HALT  |     OK     | missing_exported_at_count=0 |
| `no_empty_doc_id`               |  HALT  |     OK     | empty_doc_id_count=0        |
| `refund_no_stale_14d_window`    |  HALT  |  **FAIL**  | **violations=1**            |
| `effective_date_iso_yyyy_mm_dd` |  HALT  |     OK     | non_iso_rows=0              |
| `chunk_min_length_8`            |  WARN  |     OK     | short_chunks=0              |

> [!IMPORTANT]
> **Trạng thái Pipeline:** **HALT** (Hệ thống phát hiện lỗi nghiêm trọng, chỉ tiếp tục nạp do cờ --skip-validate)

---

## 3. Quản lý độ tươi dữ liệu (Freshness & SLA)

Kiểm tra xem dữ liệu có đáp ứng đúng cam kết về thời gian cập nhật hay không:

- **Bản ghi mới nhất lúc:** `2026-04-12T08:00:00`
- **Độ trễ thực tế:** `73.77` giờ
- **Cam kết SLA:** `10024.0` giờ
- **Kết quả Freshness:** **PASS** (Đạt chuẩn sau khi nới lỏng SLA cấu hình)

---

## 4. Đánh giá khả năng truy xuất của AI (Retrieval Evaluation)

Kết quả kiểm thử thực tế trên tập câu hỏi vàng (Golden Questions):

| Mã câu hỏi      | Nội dung câu hỏi                | Top-1 Doc        | Expected? | Hits Forbidden? |
| :-------------- | :------------------------------ | :--------------- | :-------: | :-------------: |
| q_refund_window | Hoàn tiền trong bao nhiêu ngày? | policy_refund_v4 |    yes    |     **YES**     |
| q_leave_version | Phép năm nhân viên mới 2026?    | hr_leave_policy  |    yes    |       no        |

> [!WARNING]
> Nếu **Hits Forbidden** là "YES", hệ thống đang bị nhiễm độc dữ liệu cũ (Stale context). Cần kiểm tra lại bộ lọc Cleaning Rules.

---

## 5. Phân tích nguyên nhân & Hành động (Insights & Actions)

### Nguyên nhân chính khiến dữ liệu bị cách ly (Quarantine Reasons):

- `duplicate_chunk_text`: 1 bản ghi
- `missing_effective_date`: 1 bản ghi
- `stale_hr_policy_effective_date`: 1 bản ghi
- `unknown_doc_id`: 1 bản ghi

### Đề xuất cải tiến:

1. [x] Bật lại tính năng `apply_refund_window_fix` (không sử dụng --no-refund-fix) để làm sạch triệt để.
2. [ ] Kiểm tra hệ thống nguồn (Database Export) vì độ trễ thực tế 73h là quá cao so với thực tế vận hành 24h.
3. [ ] Rà soát lại catalog tài liệu để xử lý `unknown_doc_id` (mã `legacy_catalog_xyz_zzz`).

---

_Báo cáo được tạo tự động bởi Antigravity ETL Monitoring System._
