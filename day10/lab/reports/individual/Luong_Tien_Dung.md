# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lương Tiến Dũng 
**Vai trò:** Monitoring, Documentation, Quality verification  
**Ngày nộp:** 2026-04-15

---

## 1. Tôi phụ trách phần nào?

Tôi phụ trách kiểm tra vòng chạy end-to-end để hoàn thiện runbook và chứng cứ observability cho nhóm. Phần công việc chính của tôi là chạy lại pipeline theo hai kịch bản clean và inject, đối chiếu manifest, log expectation, quarantine và file eval trước/sau để đảm bảo phần mô tả trong tài liệu không bị viết chung chung. Tôi trực tiếp cập nhật file docs/runbook.md theo format incident gồm Symptom, Detection, Diagnosis, Mitigation, Prevention và đảm bảo các số liệu trong runbook khớp artifact thực tế.

**File / module:**

- docs/runbook.md
- artifacts/manifests/manifest_sprint1.json
- artifacts/manifests/manifest_inject-bad.json
- artifacts/eval/before_after_eval.csv
- artifacts/eval/after_inject_bad.csv

**Kết nối với thành viên khác:**

Tôi lấy output clean và expectation từ phần Cleaning/Quality, sau đó xác nhận trạng thái index và kết quả retrieval để phần báo cáo nhóm và runbook dùng cùng một bộ run_id.

**Bằng chứng:**

Pipeline đã được chạy với run_id sprint1 và inject-bad; runbook được cập nhật theo dữ liệu thật của các artifact nêu trên.

---

## 2. Một quyết định kỹ thuật

Quyết định kỹ thuật quan trọng tôi áp dụng là luôn tạo cặp before/after theo đúng thứ tự index state, thay vì chỉ chạy eval ngẫu nhiên. Cụ thể, tôi chạy lại pipeline clean với run_id sprint1 trước, sau đó mới xuất before_after_eval.csv; tiếp theo chạy inject-bad với --no-refund-fix và --skip-validate rồi mới xuất after_inject_bad.csv. Cách này giúp tránh sai lệch do eval đọc nhầm trạng thái collection cũ.

Ngoài ra, trong phần tài liệu runbook tôi tách rõ 3 lớp detection: freshness từ manifest, expectation từ log pipeline, và hits_forbidden từ eval CSV. Cách tách này giúp debug theo đúng thứ tự observability của lab: dữ liệu có mới không, dữ liệu có sạch không, và retrieval có còn dính context stale không.

---

## 3. Một lỗi hoặc anomaly đã xử lý

Anomaly tôi gặp trong quá trình làm là khi chạy nhiều lệnh gần nhau, file eval có thể phản ánh trạng thái chưa đúng nếu collection chưa được cập nhật theo run mong muốn. Triệu chứng là kết quả CSV không thể hiện rõ khác biệt expected giữa clean và inject. Tôi phát hiện vấn đề này bằng cách đối chiếu lại thứ tự timestamp và kiểm tra trực tiếp log expectation của run inject-bad, trong đó refund_no_stale_14d_window phải FAIL với violations=1.

Cách xử lý là chạy lại theo chuỗi tuần tự: clean run -> eval before -> inject run -> eval after. Sau khi làm lại, kết quả trở nên nhất quán: q_refund_window ở before có hits_forbidden=no và ở after có hits_forbidden=yes.

---

## 4. Bằng chứng trước / sau

**run_id clean:** sprint1  
**run_id inject:** inject-bad

Trích log:

- expectation[refund_no_stale_14d_window] OK (halt) trong run clean sprint1.
- expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1 trong run inject-bad.

Trích CSV:

- before_after_eval.csv: q_refund_window -> contains_expected=yes, hits_forbidden=no.
- after_inject_bad.csv: q_refund_window -> contains_expected=yes, hits_forbidden=yes.

Hai dòng trên là bằng chứng rõ nhất cho việc top-1 vẫn đúng nhưng top-k bị nhiễm chunk stale khi inject.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ thêm một script tự động tạo báo cáo so sánh before/after từ hai CSV eval và một manifest, xuất ra bảng markdown dùng thẳng cho runbook và group report. Mục tiêu là giảm thao tác thủ công, tránh ghi sai số liệu giữa các tài liệu, và giúp peer review nhanh hơn.