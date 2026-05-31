# AWS PrivateLink

Dễ hiểu nhất:

```text
AWS PrivateLink = đường kết nối private giữa VPC của bạn và một service,
không đi qua Internet public.
```

Nói kiểu đời thường:

```text
Không dùng PrivateLink:
EC2 → Internet → AWS service / Partner service

Dùng PrivateLink:
EC2 → Private IP trong VPC → AWS PrivateLink → Service
```

Giống như thay vì đi đường công cộng, bạn đi bằng **đường nội bộ riêng trong AWS network**.

---

## 1. AWS PrivateLink dùng để làm gì?

PrivateLink giúp bạn truy cập service một cách **private**, an toàn hơn, không cần public IP, không cần Internet Gateway, không cần NAT Gateway trong nhiều trường hợp.

Ví dụ:

```text
EC2 private subnet
  ↓
Interface VPC Endpoint
  ↓
AWS PrivateLink
  ↓
Secrets Manager / SSM / ECR / CloudWatch Logs / Partner SaaS
```

Điểm chính:

| Lợi ích | Giải thích |
|---|---|
| Không đi Internet | Traffic nằm trong AWS network |
| Không cần public IP | EC2 private vẫn gọi được AWS service |
| Tăng security | Giảm exposure ra Internet |
| Giảm phụ thuộc NAT Gateway | Một số case có thể giảm NAT cost |
| Kết nối private tới SaaS/partner | Ví dụ service bên thứ ba expose qua PrivateLink |

---

## 2. PrivateLink khác gì VPC Endpoint?

Nhiều người hay nhầm chỗ này.

**VPC Endpoint** là “cổng” trong VPC của bạn.

**PrivateLink** là công nghệ phía sau giúp kết nối private.

Có 2 loại VPC Endpoint phổ biến:

| Loại endpoint | Dùng cho | Ví dụ |
|---|---|---|
| Gateway Endpoint | S3, DynamoDB | EC2 private access S3 không qua NAT |
| Interface Endpoint | Hầu hết AWS service khác / PrivateLink service | SSM, Secrets Manager, ECR, CloudWatch Logs |

PrivateLink chủ yếu liên quan tới **Interface VPC Endpoint**.

---

## 3. Interface Endpoint là gì?

**Interface Endpoint = một Elastic Network Interface có private IP nằm trong subnet của bạn.**

Ví dụ bạn tạo endpoint cho Secrets Manager:

```text
VPC: 10.0.0.0/16

Private Subnet:
  EC2: 10.0.10.20
  Secrets Manager Endpoint ENI: 10.0.10.50
```

Khi EC2 gọi Secrets Manager:

```text
EC2 10.0.10.20
  ↓ private IP
Endpoint ENI 10.0.10.50
  ↓ AWS PrivateLink
Secrets Manager
```

Không cần đi ra Internet.

---

## 4. Ví dụ dễ hiểu: EC2 private cần dùng SSM Session Manager

Nếu EC2 nằm private subnet, không có public IP, bạn vẫn muốn dùng Session Manager.

Không có PrivateLink:

```text
EC2 private → NAT Gateway → Internet → SSM service
```

Có PrivateLink:

```text
EC2 private → Interface Endpoint → SSM service
```

Bạn cần tạo các endpoint thường gặp:

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
```

Nếu EC2 cần gửi log:

```text
com.amazonaws.ap-northeast-1.logs
```

Nếu cần lấy image từ ECR:

```text
com.amazonaws.ap-northeast-1.ecr.api
com.amazonaws.ap-northeast-1.ecr.dkr
```

---

## 5. Ví dụ: Laravel trên EC2 private lấy secret

Mô hình tốt:

```text
Laravel EC2 - Private Subnet
  ↓
Secrets Manager Interface Endpoint
  ↓
Secrets Manager
```

Không cần:

```text
Public IP ❌
Internet Gateway ❌
NAT Gateway cho Secrets Manager ❌
Access Key trong app ❌
```

Kết hợp best practice:

```text
EC2 dùng IAM Role
Secrets lưu trong Secrets Manager
Traffic đi qua Interface Endpoint
Security Group chỉ allow HTTPS 443 nội bộ
```

---

## 6. PrivateLink dùng trong những case nào?

### Case 1: Private subnet access AWS service

Ví dụ:

```text
EC2 / ECS / Lambda trong private subnet
cần gọi:
- SSM
- Secrets Manager
- ECR
- CloudWatch Logs
- STS
- KMS
- SQS
- SNS
```

Dùng Interface Endpoint để không cần đi Internet.

### Case 2: Truy cập S3/DynamoDB private

Với S3 và DynamoDB, thường dùng **Gateway Endpoint**, không phải Interface Endpoint.

```text
EC2 private → Gateway Endpoint → S3
```

Use case rất phổ biến:

```text
EC2 backup MySQL → S3
```

Thay vì đi qua NAT Gateway:

```text
EC2 private → NAT Gateway → S3
```

Bạn dùng:

```text
EC2 private → S3 Gateway Endpoint → S3
```

Vừa private hơn, vừa thường tối ưu cost hơn.

### Case 3: Expose service của bạn cho account khác

Ví dụ công ty A có service API nội bộ chạy sau NLB:

```text
Provider Account
NLB
 ↓
Private service
```

Công ty B muốn truy cập nhưng không muốn public Internet.

Dùng PrivateLink:

```text
Consumer VPC
  ↓ Interface Endpoint
AWS PrivateLink
  ↓
Provider NLB
  ↓
Private service
```

Lợi ích: hai bên không cần peering VPC, không cần route CIDR qua nhau, không expose public.

### Case 4: Truy cập SaaS/private partner service

Một số SaaS hỗ trợ PrivateLink.

Ví dụ:

```text
Your VPC
  ↓
Interface Endpoint
  ↓
Partner SaaS service
```

Use case: security/compliance yêu cầu traffic không đi public Internet.

---

## 7. PrivateLink khác gì VPC Peering?

| Tiêu chí | PrivateLink | VPC Peering |
|---|---|---|
| Mục đích | Access một service cụ thể | Kết nối network giữa 2 VPC |
| Routing | Không cần route toàn VPC | Cần route table |
| CIDR overlap | Có thể xử lý tốt hơn | Không hỗ trợ overlap CIDR |
| Exposure | Chỉ expose service qua endpoint | Hai VPC có thể route tới nhau |
| Dùng cho SaaS | Rất phù hợp | Không phổ biến |
| Độ kiểm soát | Hẹp, an toàn hơn | Rộng hơn |

Dễ nhớ:

```text
VPC Peering = nối hai mạng với nhau
PrivateLink = chỉ mở một cửa private tới một service
```

---

## 8. PrivateLink khác gì NAT Gateway?

| Tiêu chí | NAT Gateway | PrivateLink / VPC Endpoint |
|---|---|---|
| Mục đích | Private subnet đi Internet | Private subnet gọi service private |
| Traffic | Có thể ra public Internet | Không cần public Internet |
| Security | Rộng hơn | Hẹp hơn, theo service |
| Cost | Tính theo giờ + data processed | Interface endpoint cũng có phí, Gateway endpoint thường rẻ hơn cho S3/DynamoDB |
| Use case | Update package, gọi API ngoài | Gọi AWS service/private service |

Ví dụ:

```text
Cần apt/yum update từ Internet → NAT Gateway
Cần gọi Secrets Manager → Interface Endpoint
Cần upload S3 → Gateway Endpoint
```

---

## 9. Pricing model

PrivateLink / Interface Endpoint thường tính theo:

```text
Số giờ endpoint chạy
+ lượng data processed
```

Gateway Endpoint cho S3/DynamoDB thường không có hourly charge, nên rất hay dùng để tối ưu cost khi EC2 private access S3/DynamoDB.

NAT Gateway cũng tính theo:

```text
Số giờ NAT Gateway chạy
+ data processed
```

Vì vậy nếu workload private gọi nhiều AWS service, cần so sánh:

```text
NAT Gateway cost
vs
Interface Endpoint cost
```

Không phải lúc nào endpoint cũng rẻ hơn NAT, nhưng endpoint thường tốt hơn về security và kiểm soát traffic.

---

## 10. Security Group cho Interface Endpoint

Interface Endpoint có Security Group riêng.

Ví dụ endpoint cho Secrets Manager:

```text
Endpoint SG:
  Inbound HTTPS 443 from App EC2 SG
```

App EC2 SG outbound:

```text
Outbound HTTPS 443 to Endpoint SG
```

Best practice:

```text
Chỉ cho resource cần thiết gọi endpoint
Không mở quá rộng nếu không cần
Bật Private DNS nếu muốn app dùng endpoint tự nhiên
```

---

## 11. Private DNS là gì?

Khi bật **Private DNS**, app vẫn gọi domain service bình thường:

```text
secretsmanager.ap-northeast-1.amazonaws.com
```

Nhưng DNS trong VPC sẽ resolve về **private IP của Interface Endpoint**.

Tức là app không cần đổi code.

```text
Laravel gọi Secrets Manager endpoint bình thường
DNS tự trỏ về private endpoint trong VPC
```

Rất tiện.

---

## 12. Architecture mẫu

### Private EC2 access AWS services

```text
VPC
├── Private Subnet
│   ├── EC2 Laravel
│   ├── Interface Endpoint: Secrets Manager
│   ├── Interface Endpoint: SSM
│   ├── Interface Endpoint: CloudWatch Logs
│   └── Gateway Endpoint: S3
│
└── Public Subnet
    └── NAT Gateway, chỉ dùng nếu thật sự cần Internet
```

Flow:

```text
Laravel → S3 backup/upload       → Gateway Endpoint
Laravel → Secrets Manager        → Interface Endpoint
EC2     → SSM Session Manager    → Interface Endpoint
EC2     → CloudWatch Logs        → Interface Endpoint
EC2     → external API Internet  → NAT Gateway
```

---

## 13. Best practice

| Best practice | Giải thích |
|---|---|
| Dùng Gateway Endpoint cho S3/DynamoDB | Tối ưu và đơn giản |
| Dùng Interface Endpoint cho SSM, Secrets, ECR, Logs | Private access AWS services |
| Bật Private DNS | Không cần sửa code/app config |
| Endpoint đặt multi-AZ | Tăng availability |
| Security Group giới hạn 443 | Chỉ allow từ app/server cần thiết |
| Dùng IAM Role | Không dùng Access Key trong app |
| Dùng endpoint policy nếu service hỗ trợ | Giới hạn service/resource được access |
| Monitor cost | Interface endpoint nhiều service/AZ có thể tăng phí |
| Không thay thế NAT 100% | NAT vẫn cần nếu private server gọi Internet public |

---

## 14. Câu nhớ nhanh nhất

```text
PrivateLink = đường private tới một service.
Interface Endpoint = cổng private có private IP trong subnet.
Gateway Endpoint = đường private tới S3/DynamoDB.
NAT Gateway = private subnet đi ra Internet.
VPC Peering = nối hai VPC với nhau.
```

Câu chốt:

**AWS PrivateLink giúp resource trong VPC truy cập AWS service, SaaS service hoặc service ở VPC khác bằng private IP, không cần đi qua Internet public. Với production, nên dùng PrivateLink/VPC Endpoint cho các service nội bộ quan trọng như SSM, Secrets Manager, ECR, CloudWatch Logs và dùng Gateway Endpoint cho S3/DynamoDB để tăng security và có thể tối ưu cost.**
