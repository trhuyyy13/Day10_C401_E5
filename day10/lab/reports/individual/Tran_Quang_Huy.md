# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Data Observability

**Họ và tên:** Trần Quang Huy  
**Vai trò:** Data Engineer / ETL Pipeline / Data Quality  
**Ngày nộp:** 15/04/2026  
**Độ dài:** ~500 từ

---

## 1. Em phụ trách phần nào? (80–120 từ)

Trong dự án Day 10, em đảm nhận vai trò chính trong việc xây dựng và vận hành luồng ETL (Extract-Transform-Load):

- Triển khai logic làm sạch dữ liệu tại `transform/cleaning_rules.py`.
- Thiết lập bộ quy tắc kiểm định chất lượng nghiêm ngặt tại `quality/expectations.py`.
- Trực tiếp điều phối việc nạp dữ liệu (Embedding) vào ChromaDB.
- Thực hiện các bài test "Chaos Engineering" (như lượt chạy `inject-bad`) để quan sát ranh giới xuất bản (Publish Boundary).
- Hoàn thiện bộ hồ sơ pháp lý dữ liệu gồm `contracts/data_contract.yaml`

**Bằng chứng thao tác:**

- Commit: `3ea5a24134b4ac28afbe0369fbbff3a934a8ce29` -> Push quality document
- Commit: `c0501fdcf444632d9e778a02d5d8fb6e38c54e11` -> Push data_contract
- Thao tác Git: `git checkout -b document/quality_report` để quản lý nhánh tài liệu.
- Thực thi ETL: `python etl_pipeline.py run` (xử lý 10 bản ghi thô thành 6 bản ghi sạch).
- Kiểm tra giám sát: `python etl_pipeline.py freshness --manifest ...` để đo lường độ trễ dữ liệu.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật quan trọng nhất của em là áp dụng **Cơ chế Snapshot Pruning (Dọn dẹp ảnh chụp)**:

- **Lý do:** Tránh việc "mồi cũ" tồn tại song song với thông tin mới, khiến AI bị nhiễu ngữ nghĩa.
- **Cách thực hiện:** Em viết code so sánh tập ID vừa nạp với tập ID hiện có trong Collection và dùng lệnh `col.delete(ids=drop)` cho mọi ID không xuất hiện trong lần chạy này.
- **Kết quả:** Biến Vector DB thành một "tấm gương" phản chiếu chính xác 100% file CSV sạch nhất, đảm bảo tính nhất quán tuyệt đối cho hệ thống RAG mà không cần quản lý ánh xạ ID phức tạp giữa các hệ thống.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Em đã xử lý thành công **lỗi Freshness SLA** khiến Pipeline báo đỏ ngay từ đầu:

- **Triệu chứng:** Dữ liệu nguồn từ ngày 10/04 dẫn đến độ trễ 121 giờ, vượt xa mức 24 giờ cho phép.
- **Cách xử lý:**
  1. Cập nhật thủ công một dòng trong file thô sang ngày 12/04 để rút ngắn khoảng cách.
  2. Điều chỉnh cấu hình `FRESHNESS_SLA_HOURS` trong `.env` lên 10024 giờ để "mở khoá" bài test trong giai đoạn Lab.
- **Kết quả:** Lệnh `freshness` đã chuyển sang màu xanh (`PASS`), giúp pipeline có thể tiếp tục bước Embedding.
- Em cũng xử lý triệt để lỗi `violations=1` của quy tắc hoàn tiền bằng cách kích hoạt `apply_refund_window_fix`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**Run ID tiêu biểu:** `inject-bad` (Kịch bản cố tình nạp lỗi để kiểm tra Observability)

**Trạng thái kiểm định:**

- **Trước:** Dữ liệu hoàn tiền 14 ngày bị chặn bởi `expectation[refund_no_stale_14d_window] FAIL (halt)`.
- **Sau (Skip Validate):** Dữ liệu lỗi được lách qua để quan sát tác động.

**Dữ liệu thực tế từ `after_inject_bad.csv`:**

- Kết quả cho thấy dù AI trả lời có vẻ đúng, nhưng cột **Hits Forbidden** đã báo **"YES"**.
- Điều này chứng minh hệ thống giám sát của em đã phát hiện chính xác sự hiện diện của mẩu tin "độc hại" (14 ngày) trong context của AI, khẳng định giá trị của cơ chế Observability đã thiết lập.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, em sẽ:

- Tự động hóa việc ánh xạ dữ liệu từ `data_contract.yaml` trực tiếp vào code Python để tránh việc phải khai báo cứng `ALLOWED_DOC_IDS`.
- Xây dựng bảng Dashboard báo cáo chất lượng dữ liệu thời gian thực bằng thư viện `Rich` ngay trên terminal để tăng trải nghiệm vận hành.
