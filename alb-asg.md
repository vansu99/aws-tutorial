# Application Load Balancer & Auto Scaling Group

Dễ hiểu nhất:

```text
Application Load Balancer = người phân phối traffic
Auto Scaling Group       = đội tự tăng/giảm số lượng EC2
```

Hai service này thường đi chung trong production web/app:

```text
Users
  ↓
Route 53 / CloudFront
  ↓
Application Load Balancer
  ↓
Auto Scaling Group
  ↓
EC2 Instances
  ↓
RDS / ElastiCache / S3
```

---

## 1. Application Load Balancer là gì?

**Application Load Balancer — ALB** là bộ cân bằng tải tầng application, dùng cho HTTP/HTTPS.

Nói dễ hiểu:

```text
User gửi request vào website
ALB nhận request
ALB chuyển request đến EC2 khỏe mạnh phía sau
```

Ví dụ bạn có 3 EC2:

```text
User request 1 → EC2 A
User request 2 → EC2 B
User request 3 → EC2 C
```

Nếu EC2 B bị lỗi:

```text
ALB phát hiện EC2 B unhealthy
ALB ngừng gửi traffic tới EC2 B
Traffic chỉ đi tới EC2 A và EC2 C
```

---

## 2. ALB dùng khi nào?

Dùng ALB khi bạn có web/app cần nhận traffic HTTP/HTTPS:

| Use case | Ví dụ |
|---|---|
| Website | Laravel, WordPress, Node.js, React SSR |
| API | `/api/users`, `/api/orders` |
| Microservices | Route theo path `/service-a`, `/service-b` |
| HTTPS termination | Gắn SSL certificate từ ACM |
| Blue/Green deployment | Chia traffic sang version mới |
| Multi-AZ high availability | Phân phối traffic tới nhiều AZ |

Ví dụ Laravel:

```text
User
  ↓ HTTPS
ALB
  ↓ HTTP/HTTPS
EC2 Laravel
```

---

## 3. ALB có những thành phần nào?

### Listener

**Listener = cổng ALB lắng nghe request.**

Ví dụ:

```text
HTTP  : port 80
HTTPS : port 443
```

Best practice:

```text
Port 80  → redirect sang 443
Port 443 → forward tới Target Group
```

### Target Group

**Target Group = nhóm server/app phía sau ALB.**

Ví dụ:

```text
Target Group: prod-laravel-tg
Targets:
  - EC2 A
  - EC2 B
  - EC2 C
```

ALB không gửi trực tiếp tới ASG. Thực tế là:

```text
ALB → Target Group → EC2 instances trong ASG
```

### Health Check

**Health Check = ALB kiểm tra server còn sống không.**

Ví dụ:

```text
Path: /health
Port: traffic port
Success code: 200
```

Nếu EC2 trả `200 OK`:

```text
healthy
```

Nếu timeout hoặc trả `500`:

```text
unhealthy
```

Best practice: app nên có endpoint riêng:

```text
/health
```

Không nên dùng homepage `/` nếu homepage phụ thuộc DB/cache nặng, vì dễ báo unhealthy sai.

### Listener Rule

ALB có thể route theo domain/path.

Ví dụ:

```text
api.example.com      → API Target Group
admin.example.com    → Admin Target Group
example.com          → Web Target Group
```

Hoặc theo path:

```text
/api/*     → API Service
/admin/*   → Admin Service
/images/*  → Image Service
```

---

## 4. Auto Scaling Group là gì?

**Auto Scaling Group — ASG** là nhóm EC2 có thể tự tăng hoặc giảm số lượng instance.

Nói dễ hiểu:

```text
Ít traffic  → chạy 2 EC2
Nhiều traffic → tự tăng lên 4, 6, 8 EC2
Hết cao điểm → tự giảm về 2 EC2
```

ASG giúp hệ thống:

```text
Không thiếu server khi traffic tăng
Không lãng phí server khi traffic giảm
Tự thay EC2 bị lỗi
```

---

## 5. ASG có những thành phần nào?

### Launch Template

**Launch Template = bản thiết kế để tạo EC2.**

Trong đó có:

```text
AMI
Instance type
Key pair
Security Group
IAM Role
User data
EBS volume
```

Ví dụ:

```text
Launch Template: prod-laravel-template
AMI: Laravel image
Instance type: t3.medium
Security Group: app-sg
IAM Role: prod-laravel-ec2-role
```

ASG dùng Launch Template để tạo EC2 mới.

### Desired / Min / Max Capacity

ASG có 3 thông số rất quan trọng:

| Thông số | Ý nghĩa |
|---|---|
| Min | Số EC2 tối thiểu |
| Desired | Số EC2 mong muốn hiện tại |
| Max | Số EC2 tối đa |

Ví dụ:

```text
Min     = 2
Desired = 2
Max     = 6
```

Nghĩa là bình thường chạy 2 EC2, khi tải tăng có thể scale tối đa lên 6 EC2.

### Scaling Policy

**Scaling Policy = điều kiện để tăng/giảm EC2.**

Ví dụ:

```text
Nếu CPU > 70% trong 5 phút → tăng EC2
Nếu CPU < 30% trong 15 phút → giảm EC2
```

Loại phổ biến nhất là **Target Tracking**:

```text
Giữ CPU trung bình khoảng 50%
```

ASG sẽ tự tính cần thêm/bớt EC2.

---

## 6. ALB và ASG hoạt động cùng nhau như thế nào?

Mô hình chuẩn:

```text
Users
  ↓
ALB
  ↓
Target Group
  ↓
ASG
  ↓
EC2 A
EC2 B
EC2 C
```

Flow:

```text
1. User gửi request tới ALB
2. ALB kiểm tra target nào healthy
3. ALB chuyển request tới EC2 healthy
4. ASG theo dõi số lượng EC2
5. Nếu EC2 lỗi, ASG tạo EC2 mới
6. Nếu traffic tăng, ASG scale out thêm EC2
7. Nếu traffic giảm, ASG scale in bớt EC2
```

---

## 7. Ví dụ thực tế: Laravel Production

```text
Route 53
  ↓
CloudFront
  ↓
ALB HTTPS 443
  ↓
Target Group
  ↓
ASG: Laravel EC2
  ↓
RDS MySQL
  ↓
S3
```

Thiết kế subnet:

```text
VPC
├── Public Subnet AZ-a
│   └── ALB
├── Public Subnet AZ-c
│   └── ALB
│
├── Private App Subnet AZ-a
│   └── EC2 Laravel
├── Private App Subnet AZ-c
│   └── EC2 Laravel
│
└── Private DB Subnet
    └── RDS
```

Best practice:

```text
ALB nằm public subnet
EC2 app nằm private subnet
RDS nằm private DB subnet
```

---

## 8. Security Group nên set thế nào?

### ALB Security Group

```text
Inbound:
  80  from 0.0.0.0/0
  443 from 0.0.0.0/0

Outbound:
  App port tới App EC2 SG
```

Nếu dùng HTTPS:

```text
80  → redirect 443
443 → forward target group
```

### EC2 App Security Group

```text
Inbound:
  80 hoặc 8080 from ALB Security Group only

Outbound:
  3306 to RDS SG
  443 to S3/Secrets Manager/Internet nếu cần
```

Không nên mở:

```text
App EC2 inbound 80 from 0.0.0.0/0 ❌
App EC2 SSH 22 from 0.0.0.0/0 ❌
```

Đúng hơn:

```text
Chỉ ALB được gọi EC2 app
Admin access dùng SSM Session Manager
```

### RDS Security Group

```text
Inbound:
  3306 from App EC2 Security Group only
```

Không nên:

```text
3306 from 0.0.0.0/0 ❌
```

---

## 9. Health Check nên cấu hình thế nào?

Ví dụ cho Laravel:

```text
Health check path: /health
Protocol: HTTP
Port: traffic port
Success code: 200
Healthy threshold: 2
Unhealthy threshold: 3
Timeout: 5 seconds
Interval: 30 seconds
```

Laravel route mẫu:

```php
Route::get('/health', function () {
    return response('OK', 200);
});
```

Nếu muốn check DB:

```php
Route::get('/health', function () {
    try {
        DB::select('select 1');
        return response('OK', 200);
    } catch (Exception $e) {
        return response('DB Error', 500);
    }
});
```

Lưu ý: nếu health check phụ thuộc DB, khi DB có vấn đề toàn bộ EC2 có thể bị đánh unhealthy. Production nên cân nhắc tách:

```text
/health       = app còn sống
/ready        = app sẵn sàng phục vụ đầy đủ
```

---

## 10. Auto Scaling example

Ví dụ cấu hình đơn giản:

```text
Min capacity: 2
Desired capacity: 2
Max capacity: 6

Scaling policy:
  Target tracking CPU = 50%
```

Ý nghĩa:

```text
Bình thường chạy 2 EC2
Nếu CPU trung bình cao hơn 50%, ASG thêm EC2
Nếu CPU thấp hơn 50%, ASG giảm EC2
Luôn giữ ít nhất 2 EC2
Không vượt quá 6 EC2
```

---

## 11. Pricing model

### ALB tính phí theo

```text
Thời gian ALB chạy
+ LCU usage
```

LCU liên quan đến:

```text
Số connection mới
Số active connection
Số GB processed
Số rule evaluations
```

Nói dễ hiểu: ALB càng nhiều traffic, xử lý càng nhiều request/data thì cost càng tăng.

### ASG có tính phí không?

**Auto Scaling Group bản thân không tính phí riêng.**

Bạn trả tiền cho resource mà ASG tạo ra:

```text
EC2 instances
EBS volumes
Data transfer
CloudWatch alarm nếu có
```

Ví dụ ASG scale từ 2 EC2 lên 6 EC2 thì bạn trả phí cho 6 EC2 trong thời gian chúng chạy.

---

## 12. ALB vs NLB khác nhau nhanh

| Tiêu chí | ALB | NLB |
|---|---|---|
| Layer | Layer 7 | Layer 4 |
| Protocol | HTTP/HTTPS | TCP/UDP/TLS |
| Route theo path/domain | Có | Không như ALB |
| Dùng cho web/API | Rất phù hợp | Phù hợp TCP/UDP performance cao |
| Static IP | Không trực tiếp | Có thể dùng Elastic IP |
| WebSocket | Có hỗ trợ | Có |

Dùng đơn giản:

```text
Web/API HTTP/HTTPS → ALB
TCP/UDP/game/low latency/static IP → NLB
```

---

## 13. Best practice production

| Hạng mục | Khuyến nghị |
|---|---|
| Multi-AZ | ALB và ASG chạy ít nhất 2 AZ |
| Private EC2 | EC2 app không cần public IP |
| Health check | Dùng `/health` endpoint |
| HTTPS | Gắn ACM certificate vào ALB |
| HTTP redirect | Port 80 redirect sang 443 |
| Security Group | EC2 chỉ allow từ ALB SG |
| IAM Role | EC2 dùng IAM Role, không dùng Access Key |
| Logging | Bật ALB access logs nếu cần audit |
| Monitoring | CloudWatch metrics + alarm |
| Scaling | Target Tracking hoặc request count |
| Deployment | Dùng Launch Template version rõ ràng |
| Backup | App stateless, data lưu RDS/S3/EFS |
| Session | Không lưu session local EC2; dùng Redis/DynamoDB/DB |
| Cost | Rightsize EC2, dùng Savings Plans nếu ổn định |

---

## 14. Lỗi thường gặp

### EC2 healthy trong console nhưng ALB báo unhealthy

Nguyên nhân thường gặp:

```text
Security Group EC2 không allow từ ALB
Health check path sai
App không listen đúng port
Nginx/PHP-FPM lỗi
HTTP code không phải 200
Route table/NACL chặn traffic
```

### Scale out thêm EC2 nhưng app lỗi

Nguyên nhân:

```text
AMI chưa có code mới
User data lỗi
.env thiếu
Permission S3/Secrets Manager thiếu
EC2 không join Target Group
Security Group sai
```

### User bị logout khi request qua nhiều EC2

Nguyên nhân:

```text
Session lưu local trên từng EC2
```

Cách xử lý:

```text
Lưu session vào Redis/ElastiCache
Hoặc database
Hoặc DynamoDB
```

Không nên phụ thuộc sticky session nếu có thể tránh.

### Upload file bị mất khi scale EC2

Nguyên nhân:

```text
File upload lưu local EBS của từng EC2
```

Cách xử lý:

```text
Upload file lên S3
Hoặc dùng EFS nếu cần shared filesystem
```

---

## 15. Mô hình Laravel tốt hơn với ALB + ASG

```text
Users
  ↓
Route 53
  ↓
CloudFront
  ↓
ALB + ACM HTTPS
  ↓
ASG Laravel EC2 private subnets
  ↓
RDS MySQL Multi-AZ
  ↓
ElastiCache Redis for session/cache
  ↓
S3 for uploads
```

Nên tránh:

```text
Session local EC2 ❌
Upload local EC2 ❌
Access Key trong .env ❌
Single EC2 production ❌
RDS public access ❌
```

---

## 16. Câu nhớ nhanh nhất

```text
ALB = phân phối request tới EC2 khỏe mạnh
ASG = tự giữ đúng số lượng EC2 và tự scale theo tải

ALB xử lý traffic.
ASG xử lý số lượng server.
```

Câu chốt:

**Trong production, ALB và ASG thường đi cùng nhau: ALB nhận và phân phối traffic, ASG tự tăng/giảm/thay thế EC2. EC2 app nên nằm trong private subnet, chỉ nhận traffic từ ALB, data nên lưu ở RDS/S3/EFS/ElastiCache thay vì lưu local trên từng EC2.**
