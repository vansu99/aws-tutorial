# Giải thích Auto Scaling trong AWS

## 1. Auto Scaling là gì?

**Auto Scaling = AWS tự động tăng hoặc giảm số lượng server theo tải thực tế.**

Thay vì bạn phải ngồi canh hệ thống:

> “Hôm nay đông user quá, tạo thêm EC2 đi.”  
> “Tối ít user rồi, tắt bớt EC2 đi cho đỡ tốn tiền.”

AWS Auto Scaling sẽ làm việc đó tự động.

Mục tiêu chính:

- Đủ tài nguyên để hệ thống chạy ổn định
- Tự phục hồi khi instance bị lỗi
- Không chạy dư server gây tốn chi phí

---

## 2. Auto Scaling giải quyết vấn đề gì?

Ví dụ bạn có website bán hàng chạy trên EC2.

Bình thường chỉ cần **2 EC2** là đủ.

Nhưng vào ngày sale lớn:

```text
Bình thường: 1.000 user/ngày
Ngày sale: 50.000 user/ngày
```

Nếu vẫn chỉ có 2 EC2, website có thể gặp lỗi:

```text
Chậm
Timeout
502 / 503
Server quá tải
Khách không mua hàng được
```

Auto Scaling sẽ tự động thêm server:

```text
2 EC2  →  4 EC2  →  6 EC2
```

Khi hết cao điểm, AWS tự giảm lại:

```text
6 EC2  →  3 EC2  →  2 EC2
```

Nói đơn giản: **Auto Scaling giúp hệ thống tự co giãn theo nhu cầu thực tế.**

---

## 3. Mô hình hoạt động cơ bản

Kiến trúc thường gặp:

```text
Users
  ↓
CloudFront / Route 53
  ↓
Application Load Balancer
  ↓
Auto Scaling Group
  ↓
EC2 Instances
```

Trong đó:

| Thành phần | Vai trò |
|---|---|
| Users | Người dùng truy cập hệ thống |
| CloudFront / Route 53 | Phân phối nội dung hoặc định tuyến DNS |
| Application Load Balancer | Nhận request và chia tải cho các EC2 |
| Auto Scaling Group | Quản lý số lượng EC2 |
| EC2 Instances | Server chạy ứng dụng |
| CloudWatch | Theo dõi CPU, request, network, health check |
| Scaling Policy | Quyết định khi nào thêm hoặc bớt EC2 |

---

## 4. Ví dụ cấu hình Auto Scaling Group

Giả sử bạn cấu hình Auto Scaling Group như sau:

```text
Minimum capacity: 2
Desired capacity: 3
Maximum capacity: 6
```

Ý nghĩa:

| Cấu hình | Ý nghĩa |
|---|---|
| Minimum = 2 | Luôn giữ ít nhất 2 EC2 để hệ thống không thiếu server |
| Desired = 3 | Bình thường AWS cố gắng duy trì 3 EC2 |
| Maximum = 6 | Khi tải tăng, AWS được phép tăng tối đa lên 6 EC2 |

Ví dụ rule:

```text
Nếu CPU > 70% trong 5 phút → thêm 1 EC2
Nếu CPU < 30% trong 10 phút → giảm 1 EC2
```

Khi traffic tăng:

```text
CPU 80%
↓
CloudWatch Alarm báo tải cao
↓
Auto Scaling Group tạo thêm EC2
↓
Load Balancer chia traffic sang EC2 mới
↓
Website chạy ổn hơn
```

Khi traffic giảm:

```text
CPU 20%
↓
CloudWatch Alarm báo tải thấp
↓
Auto Scaling Group terminate bớt EC2
↓
Giảm chi phí
```

---

## 5. Auto Scaling không chỉ để chịu tải

Nhiều người hiểu nhầm Auto Scaling chỉ dùng để tăng server khi đông user. Thực tế nó có 3 lợi ích lớn.

### 5.1. Tăng hiệu năng

Khi user đông, hệ thống có thêm EC2 để xử lý request.

Ví dụ:

```text
1 EC2 xử lý 1.000 request/phút
3 EC2 xử lý khoảng 3.000 request/phút
```

### 5.2. Tăng độ sẵn sàng

Nếu 1 EC2 bị lỗi, Auto Scaling có thể tự thay thế bằng instance mới.

Ví dụ:

```text
EC2-1 Healthy
EC2-2 Unhealthy
↓
Auto Scaling terminate EC2-2
↓
Launch EC2-3 mới
```

Điểm này rất quan trọng trong production vì hệ thống không phụ thuộc vào một server duy nhất.

### 5.3. Tối ưu chi phí

Không cần chạy 6 server cả ngày nếu ban đêm chỉ cần 2 server.

```text
Ban ngày: 5 EC2
Ban đêm: 2 EC2
```

Bạn chỉ trả tiền cho EC2 đang chạy.

---

## 6. Scale Out, Scale In, Scale Up, Scale Down

| Thuật ngữ | Ý nghĩa |
|---|---|
| Scale Out | Tăng thêm EC2 |
| Scale In | Giảm bớt EC2 |
| Scale Up | Tăng cấu hình máy, ví dụ t3.medium → t3.large |
| Scale Down | Giảm cấu hình máy |

Auto Scaling Group chủ yếu dùng cho **Scale Out / Scale In**, tức là **horizontal scaling**.

Ví dụ:

```text
Scale Out:
2 EC2 → 4 EC2

Scale In:
4 EC2 → 2 EC2
```

---

## 7. Target Tracking Scaling là gì?

Đây là kiểu cấu hình dễ dùng nhất.

Bạn chỉ cần nói với AWS:

```text
Hãy giữ CPU trung bình khoảng 50%
```

AWS sẽ tự tính nên thêm hay bớt EC2.

Ví dụ:

```text
Target CPU = 50%
Hiện tại CPU = 80%
→ AWS thêm EC2

Target CPU = 50%
Hiện tại CPU = 20%
→ AWS giảm EC2
```

Cách này phù hợp cho hầu hết web application.

---

## 8. Step Scaling là gì?

Step Scaling là cấu hình chi tiết hơn.

Ví dụ:

```text
CPU từ 60% đến 70% → thêm 1 EC2
CPU từ 70% đến 85% → thêm 2 EC2
CPU trên 85% → thêm 3 EC2
```

Nó giống như chia mức độ quá tải:

```text
Tải hơi cao  → thêm ít
Tải rất cao → thêm nhiều
```

Dùng khi hệ thống có traffic tăng mạnh hoặc cần kiểm soát kỹ cách scale.

---

## 9. Ví dụ thực tế: Website Laravel chạy trên EC2

Giả sử bạn có hệ thống Laravel:

```text
Users
  ↓
Application Load Balancer
  ↓
EC2 Laravel App trong Auto Scaling Group
  ↓
RDS MySQL
  ↓
S3 lưu ảnh/file
  ↓
SQS xử lý queue
```

Khi user upload ảnh hoặc đặt hàng nhiều:

```text
Request tăng
CPU EC2 tăng
Queue nhiều hơn
Response chậm
```

Bạn có thể cấu hình:

```text
Nếu CPU Laravel EC2 > 70% → thêm EC2 app
Nếu request count trên ALB cao → thêm EC2 app
Nếu SQS queue tăng → thêm EC2 worker
```

Nên tách riêng 2 nhóm Auto Scaling:

```text
ASG Web:
Scale theo CPU hoặc ALB Request Count

ASG Worker:
Scale theo số lượng message trong SQS
```

Thiết kế này tốt hơn vì web request và background job có tải khác nhau.

---

## 10. Best Practice khi dùng Auto Scaling trong Production

Auto Scaling không phải cứ bật là xong. Cần thiết kế đúng để tránh lỗi.

| Hạng mục | Best practice |
|---|---|
| EC2 nên stateless | Không lưu file quan trọng local trên EC2 |
| File upload | Đưa lên S3 |
| Session | Dùng ElastiCache Redis hoặc database |
| Database | Không để DB chạy trong ASG web |
| Health check | Dùng ALB health check |
| Multi-AZ | Chạy EC2 ở ít nhất 2 Availability Zones |
| Logs | Đẩy log về CloudWatch Logs |
| Deploy | Dùng Launch Template / AMI / User Data / CI/CD |
| Security | Dùng IAM Role, Security Group chặt chẽ, không hard-code key |
| Backup | Backup database, EBS, cấu hình quan trọng |
| Monitoring | Theo dõi CPU, memory, disk, request count, error rate |

Quan trọng nhất:

> EC2 trong Auto Scaling có thể bị terminate bất cứ lúc nào, nên không nên lưu dữ liệu quan trọng trực tiếp trong máy EC2.

---

## 11. Ví dụ dễ nhớ

Auto Scaling giống như **quản lý nhân viên tự động cho nhà hàng**:

```text
Đông khách  → gọi thêm nhân viên
Vắng khách → cho bớt nhân viên nghỉ
Nhân viên bị bệnh → thay người khác
```

Trong AWS:

```text
Đông traffic → thêm EC2
Ít traffic → giảm EC2
EC2 lỗi → thay EC2 mới
```

---

## 12. Tóm tắt ngắn gọn

Auto Scaling giúp hệ thống đạt 3 mục tiêu chính:

```text
Ổn định hơn
Chịu tải tốt hơn
Tối ưu chi phí hơn
```

Nói ngắn gọn:

> Auto Scaling là nền tảng của hệ thống cloud có tính đàn hồi.  
> Thay vì cố định số lượng server, hệ thống sẽ tự điều chỉnh theo nhu cầu thực tế.

---

## 13. Ghi nhớ cho AWS Solutions Architect

Khi thiết kế hệ thống production trên AWS, Auto Scaling nên đi cùng:

```text
Application Load Balancer
Multi-AZ
CloudWatch Alarm
Launch Template
IAM Role
Security Group
CloudWatch Logs
S3 / RDS / ElastiCache để tách state khỏi EC2
```

Thiết kế tốt không phải là chạy thật nhiều server.

Thiết kế tốt là:

> Chạy đủ tài nguyên khi cần, tự phục hồi khi lỗi, và tự giảm tài nguyên khi không còn cần thiết.
