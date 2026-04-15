# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Data Observability

**Họ và tên:** Trần Ngọc Sơn  
**Vai trò:** Data Quality Assurance / Compliance / Documentation  
**Ngày nộp:** 15/04/2026  
**Độ dài:** ~510 từ

---

## 1. Em phụ trách phần nào? (80–120 từ)

Trong dự án Lab Day 10, em đảm nhận vai trò quản lý tính tuân thủ và tài liệu hóa cho hệ thống dữ liệu:

- Thiết lập và hoàn thiện tệp **Data Contract** (`contracts/data_contract.yaml`) để quy chuẩn hóa schema đầu ra của pipeline.
- Xây dựng bản tài liệu hướng dẫn chất lượng dữ liệu (`docs/data_contract.md`) giúp các bên liên quan dễ dàng tra cứu các quy tắc HALT/WARN.
- Phân tích dữ liệu tại khu vực Cách ly (`quarantine/`) để xác định các lỗi phổ biến từ hệ thống nguồn (như sai định dạng ngày, trùng lặp nội dung).
- Đảm bảo tính nhất quán giữa danh mục tài liệu nghiệp vụ (Knowledge Base) và các bộ lọc Ingestion trong code.

**Bằng chứng thao tác:**

- Commit: `645a238a7b7363c11ef352c2868ec287a8cd7779` -> Push data_contract (YAML & MD).
- Thực thi kiểm định: Phối hợp chạy `python eval_retrieval.py` để kiểm chứng hiệu quả của Data Contract trên thực tế.
- Quản lý cấu hình: Cập nhật danh sách `allowed_doc_ids` để mở rộng phạm vi kiểm soát cho các tài liệu mới như `access_control_sop`.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật quan trọng nhất của em là **Chuẩn hóa quy trình ingestion thông qua Data Contract**. Thay vì để các quy tắc kiểm tra nằm tản mác trong code Python, em đã đưa toàn bộ định nghĩa về Schema, mức độ lỗi (Halt/Warn), và cam kết SLA vào một tệp cấu hình YAML duy nhất. Quyết định này giúp tách biệt giữa "Luật nghiệp vụ" và "Logic thực thi". Khi cần thay đổi quy định (ví dụ nới lỏng SLA hay thêm tài liệu mới), chúng ta chỉ cần cập nhật Contract mà không nhất thiết phải can thiệp sâu vào code ETL. Điều này tăng tính minh bạch và giúp các đội ngũ khác (như CS hay HR) hiểu rõ dữ liệu của họ sẽ bị kiểm soát như thế nào trước khi nạp vào AI.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Em đã phát hiện và xử lý vấn đề **"Thiếu hụt nguồn dữ liệu chuẩn"** trong hồ sơ hệ thống. Trong khi thư mục `data/docs/` có 5 tài liệu nghiệp vụ, tệp cấu hình cũ chỉ cho phép (allowlist) 4 loại tài liệu. Điều này dẫn đến việc file `access_control_sop.txt` bị hệ thống coi là "Unknown Doc" và đẩy vào Quarantine một cách lãng phí. Em đã cập nhật lại `data_contract.yaml`, bổ sung `access_control_sop` vào danh mục `canonical_sources` và `allowed_doc_ids`. Sau khi cập nhật, em đã phối hợp chạy lại Pipeline và xác nhận dữ liệu SOP này đã được nạp thành công vào ChromaDB, giúp AI có thêm kiến thức về quy trình kiểm soát truy cập mà trước đó bị bỏ sót.

---

## 4. Bằng chứng trước / sau (80–120 từ)

*Run ID tiêu biểu:* inject-bad (Kịch bản cố tình nạp lỗi để kiểm tra Observability)

*Trạng thái kiểm định:*

- *Trước:* Dữ liệu hoàn tiền 14 ngày bị chặn bởi expectation[refund_no_stale_14d_window] FAIL (halt).
- *Sau (Skip Validate):* Dữ liệu lỗi được lách qua để quan sát tác động.

**Dữ liệu thực tế từ after_inject_bad.csv:**

- Kết quả cho thấy dù AI trả lời có vẻ đúng, nhưng cột *Hits Forbidden* đã báo *"YES"*.
- Điều này chứng minh hệ thống giám sát của em đã phát hiện chính xác sự hiện diện của mẩu tin "độc hại" (14 ngày) trong context của AI, khẳng định giá trị của cơ chế Observability đã thiết lập.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, em sẽ xây dựng một script tự động kiểm tra (Auto-validator) để mỗi khi có file CSV mới, hệ thống sẽ tự đối chiếu với `data_contract.yaml` và xuất báo cáo PDF tự động cho các bên liên quan, thay vì phải kiểm tra thủ công qua các tệp lẻ.
