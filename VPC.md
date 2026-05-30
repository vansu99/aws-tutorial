# Giải thích dễ hiểu về VPC, Subnet, Security Group, NACL và AWS Networking

Dễ hiểu nhất: **VPC = khu đất riêng của bạn trong AWS**. Trong khu đất đó, bạn chia thành nhiều khu nhỏ gọi là **Subnet**, đặt server/database vào đó, rồi dùng **Security Group / NACL / Route Table / Gateway** để kiểm soát đường đi và bảo vệ hệ thống.

---

## 1. Hình dung đơn giản

Ví dụ bạn có một công ty:

```text
Internet / User
   ↓
CloudFront / Route 53
   ↓
Application Load Balancer
   ↓
Public Subnet
   ↓
EC2 Web Server
   ↓
Private Subnet
   ↓
RDS Database
```

Giải thích kiểu đời thường:

| Thành phần | Hiểu đơn giản |
|---|---|
| VPC | Khu đất / mạng riêng của công ty trong AWS |
| Subnet | Phòng / khu vực nhỏ bên trong khu đất |
| Public Subnet | Khu có cửa ra Internet |
| Private Subnet | Khu nội bộ, Internet không vào trực tiếp |
| Route Table | Bảng chỉ đường |
| Internet Gateway | Cổng ra/vào Internet |
| NAT Gateway | Cổng cho server private đi ra Internet, nhưng Internet không vào ngược lại |
| Security Group | Bảo vệ ở cửa từng server |
| NACL | Bảo vệ ở cửa từng subnet |
| VPC Endpoint | Đường riêng để đi tới AWS service như S3, DynamoDB mà không cần Internet |

---

## 2. VPC là gì?

**VPC — Virtual Private Cloud** là mạng riêng do bạn tự định nghĩa trong AWS. Bạn chọn dải IP, ví dụ:

```text
VPC CIDR: 10.0.0.0/16
```

Nghĩa là toàn bộ hệ thống trong VPC có thể dùng IP từ `10.0.x.x`.

AWS định nghĩa VPC là một phần tách biệt logic trong AWS Cloud, nơi bạn launch AWS resources trong virtual network do bạn định nghĩa.

Ví dụ thực tế:

```text
VPC: production-vpc
CIDR: 10.0.0.0/16
Region: ap-northeast-1
```

Trong VPC này bạn đặt EC2, RDS, ALB, ElastiCache, Lambda kết nối VPC, ECS service...

---

## 3. Subnet là gì?

**Subnet** là phần nhỏ hơn của VPC.

Ví dụ:

```text
VPC: 10.0.0.0/16

Public Subnet A:   10.0.1.0/24
Public Subnet C:   10.0.2.0/24

Private Subnet A:  10.0.11.0/24
Private Subnet C:  10.0.12.0/24
```

Best practice production là chia subnet theo **nhiều AZ** để tăng độ sẵn sàng:

```text
ap-northeast-1a:
  Public Subnet
  Private Subnet

ap-northeast-1c:
  Public Subnet
  Private Subnet
```

Region có nhiều Availability Zone tách biệt vật lý, được kết nối bằng network low latency, high throughput và redundant networking.

---

## 4. Public Subnet và Private Subnet khác nhau thế nào?

| Loại subnet | Dùng cho | Có route ra Internet Gateway? | Ví dụ |
|---|---|---:|---|
| Public Subnet | Thành phần cần nhận traffic từ Internet | Có | ALB, NAT Gateway, Bastion nếu có |
| Private Subnet | Thành phần nội bộ | Không trực tiếp | EC2 app, RDS, ElastiCache |

Quan trọng: **Subnet không tự public/private**. Nó public hay private là do **Route Table**.

Ví dụ:

```text
Public Subnet Route Table:
0.0.0.0/0 → Internet Gateway

Private Subnet Route Table:
0.0.0.0/0 → NAT Gateway
```

---

## 5. Route Table là gì?

**Route Table = bảng chỉ đường của subnet**.

Ví dụ:

```text
10.0.0.0/16  → local
0.0.0.0/0    → Internet Gateway
```

Nghĩa là:

| Destination | Target | Ý nghĩa |
|---|---|---|
| 10.0.0.0/16 | local | Traffic nội bộ trong VPC |
| 0.0.0.0/0 | IGW | Traffic ra Internet |

Với private subnet:

| Destination | Target | Ý nghĩa |
|---|---|---|
| 10.0.0.0/16 | local | Traffic nội bộ |
| 0.0.0.0/0 | NAT Gateway | Server private đi ra Internet |

---

## 6. Internet Gateway là gì?

**Internet Gateway — IGW** là cổng giúp tài nguyên trong public subnet giao tiếp với Internet.

Muốn EC2 public truy cập Internet cần đủ 3 điều kiện:

```text
EC2 có Public IP
Subnet route 0.0.0.0/0 → Internet Gateway
Security Group/NACL cho phép traffic
```

Không có một trong ba cái này thì Internet không vào được.

---

## 7. NAT Gateway là gì?

**NAT Gateway** cho phép EC2 trong private subnet **đi ra Internet**, ví dụ update package, gọi API bên ngoài, download patch.

Nhưng Internet **không thể chủ động đi vào EC2 private** qua NAT Gateway.

Ví dụ:

```text
Private EC2 → NAT Gateway → Internet
Internet    ❌ không vào ngược Private EC2
```

Dùng khi:

```text
EC2 app private cần yum update / apt update
EC2 cần gọi API bên thứ ba
Server cần pull package từ Internet
```

Không nên dùng NAT Gateway nếu chỉ để truy cập S3/DynamoDB, vì có thể dùng **Gateway VPC Endpoint** để tiết kiệm cost. Gateway endpoint cho S3 và DynamoDB giúp VPC kết nối tới hai service này mà không cần Internet Gateway hoặc NAT device.

---

## 8. Security Group là gì?

**Security Group = firewall gắn vào resource**, ví dụ EC2, RDS, ALB.

Nó hoạt động ở cấp resource/interface.

Ví dụ Security Group cho EC2 Web:

```text
Inbound:
HTTP  80   from ALB Security Group
HTTPS 443  from ALB Security Group
SSH   22   from office IP only

Outbound:
All traffic
```

Đặc điểm quan trọng:

| Security Group | Ý nghĩa |
|---|---|
| Stateful | Cho request vào thì response ra tự được phép |
| Chỉ có Allow | Không có Deny rule |
| Gắn vào resource | EC2, RDS, ALB, Lambda ENI... |
| Best practice | Chỉ mở port cần thiết |

Ví dụ dễ nhớ:

```text
Security Group giống bảo vệ đứng trước từng phòng.
Ai được phép vào phòng nào thì rule ở phòng đó quyết định.
```

---

## 9. NACL là gì?

**NACL — Network Access Control List** là firewall ở cấp **subnet**.

Nó kiểm soát traffic vào/ra cả subnet.

Đặc điểm:

| NACL | Ý nghĩa |
|---|---|
| Stateless | Inbound cho phép chưa chắc outbound được phép |
| Có Allow và Deny | Có thể chặn rõ ràng IP xấu |
| Gắn với subnet | Một NACL có thể gắn nhiều subnet |
| Rule theo số thứ tự | Số nhỏ được xử lý trước |

Ví dụ dễ nhớ:

```text
NACL giống cổng bảo vệ của cả tòa nhà/subnet.
Security Group giống khóa cửa từng phòng/server.
```

---

## 10. Security Group vs NACL

| Tiêu chí | Security Group | NACL |
|---|---|---|
| Gắn ở đâu | EC2/RDS/ALB/ENI | Subnet |
| Kiểu rule | Allow only | Allow và Deny |
| Stateful/Stateless | Stateful | Stateless |
| Thường dùng | Kiểm soát chính | Lớp bảo vệ phụ |
| Dễ quản lý | Dễ hơn | Cẩn thận hơn |
| Best practice | Dùng chính | Dùng để chặn subnet/IP đặc biệt |

Best practice: dùng **Security Group làm cơ chế chính** để kiểm soát network access, dùng NACL khi cần lớp kiểm soát stateless ở subnet hoặc chặn coarse-grained traffic.

---

## 11. Architecture mẫu chuẩn cho web app

```text
Users
  ↓
Route 53
  ↓
CloudFront + WAF
  ↓
Application Load Balancer
  ↓
Public Subnet
  ↓
EC2 / ECS App
  ↓
Private Subnet
  ↓
RDS / ElastiCache
```

Mô hình tốt hơn:

```text
VPC 10.0.0.0/16

AZ-1a
  Public Subnet:
    - ALB
    - NAT Gateway

  Private App Subnet:
    - EC2 / ECS App

  Private DB Subnet:
    - RDS

AZ-1c
  Public Subnet:
    - ALB
    - NAT Gateway

  Private App Subnet:
    - EC2 / ECS App

  Private DB Subnet:
    - RDS standby / replica
```

Đây là mô hình production tốt vì:

| Mục tiêu | Cách làm |
|---|---|
| High Availability | Dùng ít nhất 2 AZ |
| Security | App/DB nằm private subnet |
| Reliability | ALB phân phối traffic |
| Cost | Chỉ dùng NAT Gateway khi cần |
| Operation | Log bằng VPC Flow Logs, CloudWatch, CloudTrail |

---

## 12. Những service nào được đặt trong VPC?

Chia thành 3 nhóm cho dễ hiểu.

### Nhóm 1 — Service thật sự nằm trong VPC

| Service | Ghi chú |
|---|---|
| EC2 | Server trong subnet |
| RDS / Aurora | Database trong DB subnet group |
| ElastiCache | Redis/Memcached trong subnet |
| ECS trên EC2 | Container chạy trên EC2 trong VPC |
| ECS Fargate | Task có ENI trong subnet |
| EKS | Worker node/pod networking trong VPC |
| Lambda trong VPC | Lambda tạo ENI để truy cập private resource |
| EFS | Có mount target trong subnet |
| FSx | File system trong subnet |
| OpenSearch Service | Có thể chạy trong VPC |
| Redshift | Cluster/subnet group trong VPC |
| DMS Replication Instance | Đặt trong VPC |
| Directory Service | Đặt trong VPC |
| WorkSpaces | Dùng subnet trong VPC |
| SageMaker notebook/domain VPC mode | Có thể gắn VPC |

### Nhóm 2 — Service không nằm trong VPC nhưng có thể kết nối private qua VPC Endpoint

| Service | Cách kết nối private |
|---|---|
| S3 | Gateway Endpoint |
| DynamoDB | Gateway Endpoint |
| SQS | Interface Endpoint |
| SNS | Interface Endpoint |
| Secrets Manager | Interface Endpoint |
| Systems Manager / SSM | Interface Endpoint |
| CloudWatch Logs | Interface Endpoint |
| ECR | Interface Endpoint |
| STS | Interface Endpoint |
| KMS | Interface Endpoint |

AWS PrivateLink cho phép VPC kết nối private tới các AWS service được hỗ trợ, giữ traffic trong AWS network và tránh public internet.

### Nhóm 3 — Global/Public service, không “đặt trong VPC”

| Service | Ghi chú |
|---|---|
| IAM | Global identity service |
| Route 53 | DNS global service |
| CloudFront | CDN edge/global |
| S3 bucket | Không nằm trong VPC, nhưng truy cập private được qua endpoint |
| DynamoDB table | Không nằm trong VPC, nhưng truy cập private được qua endpoint |
| CloudWatch | Monitoring service, không đặt trong subnet |
| CloudTrail | Logging/audit service |
| AWS Budgets | Billing service |
| WAF | Gắn với CloudFront, ALB, API Gateway... |

---

## 13. Pricing model — cái nào tốn phí?

### Không tốn phí trực tiếp

| Thành phần | Có tính phí riêng không? |
|---|---:|
| VPC | Không |
| Subnet | Không |
| Route Table | Không |
| Internet Gateway | Không tính hourly riêng |
| Security Group | Không |
| NACL | Không |
| Private IPv4 | Không tính như public IPv4 |
| VPC Peering | Không hourly charge, nhưng có data transfer |

### Có thể tốn phí

| Thành phần | Cách tính phí |
|---|---|
| NAT Gateway | Tính theo giờ + GB data xử lý |
| VPC Endpoint Interface | Tính theo giờ + data processed |
| VPC Endpoint Gateway | Thường không tính hourly cho S3/DynamoDB endpoint |
| Transit Gateway | Tính attachment/hour + data processed |
| Site-to-Site VPN | Tính theo giờ connection + data transfer |
| Client VPN | Tính endpoint association/hour + client connection/hour |
| Public IPv4 | AWS có tính phí public IPv4 theo giờ |
| Elastic IP | Tính phí nếu allocate, kể cả đang attach hay idle tùy chính sách hiện tại |
| VPC Flow Logs | Tốn phí CloudWatch Logs/S3/Athena tùy nơi lưu và query |
| Network Firewall | Tính endpoint/hour + data processed |
| Data Transfer | Ra Internet, cross-AZ, cross-region có thể tốn phí |

NAT Gateway là cost driver rất hay gặp: AWS tính phí mỗi giờ NAT Gateway available và mỗi GB data NAT xử lý. Số chính xác nên check theo region đang dùng, ví dụ Tokyo có thể khác.

---

## 14. Ví dụ cost dễ hiểu

### Case 1: EC2 private tải update qua NAT Gateway

```text
Private EC2 → NAT Gateway → Internet
```

Bạn trả tiền:

```text
NAT Gateway hourly
+ NAT Gateway data processed
+ data transfer nếu có
```

Nếu có nhiều EC2 private đi Internet thường xuyên, NAT Gateway có thể tăng cost khá nhanh.

### Case 2: EC2 private upload backup lên S3

Cách chưa tối ưu:

```text
EC2 private → NAT Gateway → S3
```

Cách tốt hơn:

```text
EC2 private → Gateway VPC Endpoint → S3
```

Lợi ích:

| Cách | Cost | Security |
|---|---|---|
| Qua NAT Gateway | Tốn NAT hourly + data processed | Đi qua public endpoint |
| Qua S3 Gateway Endpoint | Tối ưu hơn | Private trong AWS network |

Đây là best practice rất đáng dùng cho backup từ EC2 lên S3.

---

## 15. Những thứ network liên quan cần biết

| Thành phần | Dùng khi nào |
|---|---|
| Elastic IP | Cần IP public cố định |
| ENI | Card mạng ảo gắn vào EC2/Lambda/ECS task |
| Public IP | Cho resource giao tiếp Internet trực tiếp |
| Private IP | Giao tiếp nội bộ trong VPC |
| VPC Peering | Nối 2 VPC đơn giản |
| Transit Gateway | Hub trung tâm nối nhiều VPC/on-prem |
| Site-to-Site VPN | Nối AWS với công ty/on-prem qua Internet mã hóa |
| Direct Connect | Kết nối private vật lý từ data center tới AWS |
| PrivateLink | Service private giữa VPC hoặc tới AWS service |
| VPC Flow Logs | Ghi log traffic network để điều tra |
| Network Firewall | Firewall layer nâng cao cho VPC |

---

## 16. Best practice ngắn gọn

Với production, nên thiết kế như sau:

```text
Public Subnet:
  - ALB
  - NAT Gateway nếu cần

Private App Subnet:
  - EC2 / ECS / EKS app

Private DB Subnet:
  - RDS / ElastiCache

Security:
  - SG chỉ mở port cần thiết
  - DB chỉ allow từ App SG
  - SSH hạn chế hoặc dùng SSM Session Manager
  - Bật VPC Flow Logs nếu cần audit
  - Dùng VPC Endpoint cho S3, DynamoDB, SSM, Secrets Manager
```

Ví dụ rule chuẩn:

```text
ALB SG:
  Inbound 443 from Internet

App SG:
  Inbound 80/8080 from ALB SG only

DB SG:
  Inbound 3306 from App SG only
```

Không nên làm:

```text
DB SG:
  Inbound 3306 from 0.0.0.0/0 ❌
```

---

## 17. Tóm tắt cực ngắn để nhớ

```text
VPC        = mạng riêng
Subnet     = khu nhỏ trong mạng
Route Table= bảng chỉ đường
IGW        = cổng Internet
NAT        = private server đi ra Internet
SG         = firewall của resource
NACL       = firewall của subnet
Endpoint   = đường private tới AWS service
```

Câu dễ nhớ nhất:

**User vào hệ thống qua Public Subnet, App và Database nên nằm trong Private Subnet. Security Group bảo vệ từng resource, NACL bảo vệ cả subnet. NAT Gateway chỉ dùng khi private server cần đi ra Internet.**
