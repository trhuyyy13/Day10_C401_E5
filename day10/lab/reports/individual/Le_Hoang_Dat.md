# Báo Cáo Cá nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lê Hoàng Đạt  
**Vai trò:** Ingestion / Raw Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `etl_pipeline.py` — entry point chính, triển khai lớp **ingestion**: load raw CSV từ `data/raw/policy_export_dirty.csv`, khởi tạo run_id, logging metrics (raw_records, cleaned_records, quarantine_records).
- `transform/cleaning_rules.py` — hàm `load_raw_csv()` để đọc dữ liệu raw; hàm `clean_rows()` nhận rows từ ingestion, chuyển tiếp cho lớp transform.
- `contracts/data_contract.yaml` — định nghĩa schema và allowed_doc_ids allowlist để validate raw data.

**Kết nối với thành viên khác:**

Tôi chuyên giao dữ liệu raw sang cho **Cleaning & Quality Owner** qua function `clean_rows()` và CSV quarantine. Sau đó **Embed Owner** nhận cleaned CSV để upsert vào Chroma. **Monitoring Owner** đọc manifest và log mà tôi sinh ra để kiểm tra freshness.

**Bằng chứe (commit / comment trong code):**

Trong `etl_pipeline.py` dòng 50–75: khởi tạo run_id, tạo log_path, print metrics. Manifest được ghi tại dòng 105–113 (`manifest_written=...`).

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Vấn đề:** Raw CSV có thể chứa timestamp `exported_at` không nhất quán hoặc rỗng. Phải quyết định: quarantine dòng nào ở tầng ingestion vs để Transform xử lý?

**Quyết định:** Tôi thêm rule **"missing_exported_at"** ở hàm `load_raw_csv()` (trước đầy đủ transform). Lý do: `exported_at` là metadata sử dụng cho **freshness check** sau; nếu thiếu sẽ làm manifest không thể tính tuổi dữ liệu. Quarantine sớm ở ingestion sẽ giảm rủi ro truyền dữ liệu lỏng vào pipeline tiếp theo. Dòng log test: `"quarantine_records=4"` (run_id 2026-04-15T05-30Z) — 1 trong 4 quarantine là missing exported_at.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** Chạy pipeline lần đầu, thấy log có `quarantine_records=4` nhưng khi inspect quarantine CSV, một dòng có `reason=unknown_doc_id` với giá trị `legacy_catalog_xyz_zzz`. Đây là dòng raw không nằm trong `ALLOWED_DOC_IDS` được định nghĩa ở `cleaning_rules.py`.

**Phát hiện:** Đọc contract YAML lần thứ hai (`contracts/data_contract.yaml` — allowed_doc_ids list), nhận ra allowlist chỉ gồm 4 doc: `policy_refund_v4`, `sla_p1_2026`, `it_helpdesk_faq`, `hr_leave_policy`. Nhưng raw export từ nguồn cũ bao gồm cả `legacy_catalog_xyz_zzz` — export bị mix version cũ.

**Fix:** Thêm log validation ở ingestion để báo sớm nếu phát hiện doc_id ngoài allowlist; cập nhật contract để team biết nên hỏi data owner: bỏ doc lạ hay thêm vào allowlist chính thức? Khi đó commit sửa chữa vào `contracts/data_contract.yaml` và update `ALLOWED_DOC_IDS` trong `cleaning_rules.py`.

---

## 4. Bằng chứe trước / sau (80–120 từ)

**Run ID:** `2026-04-15T05-30Z`

**Trước (Raw data):**

- `raw_records=10` — từ `data/raw/policy_export_dirty.csv`
- Các dòng này chứa: corrupt doc_id, missing effective_date, stale HR version (2025-01-01 < 2026-01-01 cutoff), duplicate chunk.

**Sau (Sau Ingestion + Transform):**

```
cleaned_records=6 ✓
quarantine_records=4:
  - duplicate_chunk_text: 1 dòng
  - missing_effective_date: 1 dòng  
  - stale_hr_policy_effective_date: 1 dòng
  - unknown_doc_id: 1 dòng
```

Bằng chứng từ log: `artifacts/logs/run_2026-04-15T05-30Z.log` — các dòng metric_impact được ghi.

---

## 5. Cải tiến tiếp theo (40–80 từ)

**Nếu có thêm 2 giờ:** Thêm schema validation dùng pydantic vào ingestion — kiểm tra kiểu dữ liệu (exported_at phải datetime hợp lệ, doc_id phải là string, v.v.) trước khi pass cho Transform. Hiện giờ chỉ có hardcode check; validate schema sẽ giảm anomaly phát hiện muộn ở downstream. Commit: `mcp_server.py` hoặc module validation riêng nếu nhóm quyết định dùng Great Expectations.
