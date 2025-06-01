# Cloudwatch alarm

## Mục tiêu

Thiết lập một CloudWatch Alarm để gửi email hoặc SNS khi số lượng EC2 Instances vượt quá một giới hạn, ví dụ: 8 instance.

## Hướng dẫn thiết lập cảnh báo (Alarm) theo số lượng EC2 trong Auto Scaling Group

- Bước 1: Mở CloudWatch
  + Truy cập AWS Management Console
  + Vào dịch vụ CloudWatch
  + Chọn Alarms > Create Alarm
- Bước 2: Chọn Metric
  + Bấm Select metric
  + Chọn Browse > Auto Scaling > Group Metrics > GroupDesiredCapacity
  + Tìm Auto Scaling Group của bạn, tích vào metric: `GroupDesiredCapacity`
  + Nhấn Select metric
- Bước 3: Cấu hình Alarm
  + Statistic: Maximum (hoặc Average nếu bạn muốn trung bình)
  + Period: 1 minute hoặc 5 minutes (tuỳ mức phản ứng bạn cần)
  + Threshold type: Static
  + Whenever GroupDesiredCapacity is...
    + Greater than
    + 8 (hoặc giá trị bạn muốn cảnh báo)
- Bước 4: Hành động khi cảnh báo
  + Send notification to...
    + Chọn hoặc tạo một SNS topic để nhận email.
  + Nếu chưa có SNS topic:
    + Bấm Create new topic hoặc chọn service SNS để tiến hành tạo topic
    + Nhập tên, email nhận cảnh báo
    + Sau khi tạo xong, bạn phải xác nhận email (check email và bấm vào link xác nhận từ AWS)
- Bước 5: Đặt tên và tạo Alarm
  + Đặt tên ví dụ: High-EC2-Count-Alarm
  + Review và bấm Create Alarm

## Kết quả
- Khi số lượng EC2 instance vượt quá 8, bạn sẽ:
  + Nhận email cảnh báo
  + Có thể thêm hành động tự động như: chạy Lambda để log, scale-in lại, hoặc gửi báo cáo cho người chịu trách nhiệm giám sát.

## Best Practice

- Đặt thêm Alarm cho GroupInServiceInstances: Để biết số lượng instance thực sự đang phục vụ
- Tạo Alarm cho CPUUtilization trung bình của tất cả instance: Giúp bạn phát hiện trước khi scaling
- Kết hợp Alarm với AWS Budgets: Để kiểm soát chi phí nếu EC2 scale quá mức
  




