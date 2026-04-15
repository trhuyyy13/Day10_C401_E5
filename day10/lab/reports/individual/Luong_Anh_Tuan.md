# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lương Anh Tuấn

**Vai trò:** Rule Owner

**Ngày nộp:** 15/04/2025

**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- transform/cleaning_rules.py
- etl_pipeline.py
- quality/expectations.py
- docs/data_contract.md

**Kết nối với thành viên khác:**

Tôi phối hợp với các bạn Docs Owner để đảm bảo các rule về cleaning, quarantine, expectation được áp dụng đúng, đồng thời cập nhật tài liệu hướng dẫn và báo cáo nhóm. Tôi cũng trao đổi với Monitoring Owner để kiểm tra các guardrail mới và xác nhận kết quả trên log.

**Bằng chứng (commit / comment trong code):**

- Commit hash: 0cfd4e2b96270b4ecc11aceeaf8a84819863f0fb
- Có thêm 3 comment rule mới trong cleaning_rules.py
- Có thêm 2 expectation mới trong expectations.py

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Tôi quyết định bổ sung ba rule mới trong cleaning_rules.py: quarantine nếu thiếu exported_at, quarantine nếu chunk_text chứa ký tự đặc biệt bất thường, và chuẩn hóa viết hoa đầu câu trong chunk_text. Đồng thời, tôi thêm hai expectation mới trong expectations.py để kiểm soát các trường hợp này: không dòng nào thiếu exported_at (halt) và không dòng nào chứa ký tự đặc biệt bất thường (warn). Các bổ sung này giúp pipeline phát hiện và loại bỏ dữ liệu lỗi hoặc nhiễu, đồng thời nâng cao chất lượng dữ liệu đầu ra. Tôi cũng phối hợp với các thành viên khác để đảm bảo các rule và expectation mới được kiểm thử, ghi nhận metric_impact và cập nhật tài liệu hướng dẫn.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Khi kiểm thử với run inject-bad, tôi phát hiện expectation `refund_no_stale_14d_window` bị FAIL do sử dụng tham số `--no-refund-fix`, khiến chunk 14 ngày vẫn xuất hiện trong collection. Ngoài ra, tôi cũng kiểm tra các rule và expectation mới: intentionally thêm dòng thiếu exported_at và dòng có ký tự đặc biệt vào dữ liệu, kết quả là các dòng này bị quarantine đúng theo rule mới và expectation tương ứng cũng FAIL như mong đợi. Sau khi sửa dữ liệu và rerun pipeline, các lỗi này không còn xuất hiện. Bằng chứng: artifacts/eval/before_after_eval.csv, artifacts/eval/after_inject_bad.csv, log và quarantine CSV.

---


## 4. Bằng chứng trước / sau (80–120 từ)

Tôi đã kiểm thử và ghi nhận số liệu thực tế từ file kết quả:

**Trước khi clean (before_after_eval.csv):**
```
gq_d10_01,...,policy_refund_v4,...,yes,no,,3
```
Nghĩa là: câu hỏi về policy_refund_v4 có contains_expected=yes, hits_forbidden=no (không phát hiện lỗi stale 14 ngày).

**Sau khi inject lỗi (after_inject_bad.csv):**
```
gq_d10_01,...,policy_refund_v4,...,yes,yes,,3
```
Nghĩa là: cùng câu hỏi, contains_expected=yes nhưng hits_forbidden=yes (đã phát hiện lỗi stale 14 ngày do intentionally inject).

Các dòng này chứng minh rõ ràng rule và expectation mới đã hoạt động đúng, giúp quarantine và cảnh báo các trường hợp dữ liệu lỗi, nâng cao chất lượng pipeline.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ bổ sung thêm unit test tự động cho từng rule và expectation, đồng thời mở rộng bộ dữ liệu kiểm thử để kiểm tra các trường hợp edge case, đảm bảo mọi lỗi tiềm ẩn đều được phát hiện sớm và báo cáo rõ ràng trong log/quarantine.
