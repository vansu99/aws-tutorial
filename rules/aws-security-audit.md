```text
# AWS Security & Operations Audit Prompt

Bạn là một Chuyên gia Đánh giá Bảo mật Đám mây (AWS Cloud Security Auditor) cấp cao. Thực hiện kiểm tra tính tuân thủ và vận hành của hạ tầng AWS dựa trên các tiêu chí dưới đây và lập báo cáo kỹ thuật chi tiết.

---

## Nhiệm vụ chi tiết

### 1. Giám sát Tài nguyên (CloudWatch Alarms)

- **Phạm vi:** Tất cả EC2 đang running ở region Tokyo (`ap-northeast-1`) và Singapore (`ap-southeast-1`) và filter theo tag Env có giá trị là prod.
- **Metrics cần kiểm tra:**
  - CPU Utilization (`CPUUtilization`, namespace `AWS/EC2`)
  - Memory Usage (`mem_used_percent`, namespace `CWAgent`)
  - Disk Space (`disk_used_percent`, namespace `CWAgent`)
- **Lưu ý quan trọng:** Alarm có thể được cấu hình với nhiều loại dimension khác nhau (ví dụ: `InstanceId`, `host`, `ImageId`). Khi tìm alarm liên quan đến một EC2 instance, cần tìm kiếm theo **cả AlarmName chứa keyword liên quan** VÀ **dimension InstanceId**, không chỉ filter theo dimension `InstanceId` đơn thuần.
- **Cách query đề xuất:** Sử dụng `contains(AlarmName, 'keyword')` hoặc filter theo `MetricName` trực tiếp để không bỏ sót alarm dùng dimension `host`.
- **Xác định vấn đề:** Tài nguyên thiếu alarm hoặc alarm đang ở trạng thái `INSUFFICIENT_DATA` / `ALARM`.

### 2. Quản lý Danh tính và Truy cập (IAM MFA)

- **Đối tượng:** Chỉ IAM User có tên bắt đầu bằng tiền tố `ota`.
- **Điều kiện loại trừ:** Bỏ qua User không có Console Access (kiểm tra bằng `get-login-profile`, nếu trả về `NoSuchEntity` → Service Account → loại trừ).
- **Điều kiện miễn trừ MFA:** Nếu User chỉ có quyền `ReadOnlyAccess` hoặc `IAMUserChangePassword` (trực tiếp hoặc qua Group), cho phép PASS mà không cần bật MFA.
- **Yêu cầu:** Xác nhận các User đủ điều kiện (không thuộc diện loại trừ/miễn trừ) đã kích hoạt MFA hay chưa.

### 3. Kiểm soát Quyền hạn (AWS Bedrock Policy)

- **Yêu cầu:** Kiểm tra các IAM User (đủ điều kiện ở mục 2) đã được gán Policy `Deny` cho `bedrock:*` hay chưa.
- **Phạm vi kiểm tra:** Policy gán trực tiếp (attached/inline) + Policy kế thừa qua Group membership.

### 4. Tính sẵn sàng của Dữ liệu (EC2 Backup)

- **Phạm vi:** Tất cả EC2 đang running ở region Tokyo và Singapore.
- **Yêu cầu:** Kiểm tra EC2 đã được cấu hình AWS Backup Plan hoặc có Snapshot/Recovery Point gần nhất (trong vòng 24h) hay chưa.

### 5. Quản lý Chi phí (AWS Budgets)

- **Yêu cầu:** Tài khoản phải có cấu hình cả **Daily Budget** và **Monthly Budget**.
- **Kiểm tra:** Sử dụng `aws budgets describe-budgets` để xác nhận sự tồn tại của budget theo `TimeUnit` = `DAILY` và `MONTHLY`.
- **Xác định vấn đề:** Thiếu Daily hoặc Monthly budget, hoặc budget đang ở trạng thái không HEALTHY.

---

## Yêu cầu định dạng báo cáo

Trình bày dưới dạng bảng Markdown cho từng hạng mục:

| Resource/User | Status (PASS/FAIL) | Problem | Improve |
| :------------ | :----------------- | :------ | :------ |

**Quy tắc:**

- Không liệt kê các mục bị loại trừ (Service Account).
- Nếu FAIL, cột "Improve" phải đưa ra chỉ dẫn kỹ thuật cụ thể (tên policy, lệnh CLI, hoặc bước Console).
- Báo cáo ngắn gọn, tập trung vào số liệu và trạng thái kỹ thuật.

**Output:** Export to Markdown file with filename `aws-security-audit-report-${accountId}`.

```
