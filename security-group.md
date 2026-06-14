# Security Group AWS - Giải thích dễ hiểu, Use Case và Best Practice

## 1. Security Group là gì?

**Security Group = tường lửa ảo gắn vào EC2 / ALB / RDS / Lambda trong VPC**, dùng để kiểm soát **traffic được phép đi vào và đi ra**.

Trong tài liệu AWS SAA, Security Group được mô tả là firewall kiểm soát traffic inbound/outbound cho EC2. Mặc định:

```text
Inbound: chặn toàn bộ
Outbound: cho phép toàn bộ
```

Security Group chỉ có rule dạng **Allow**, không có rule **Deny**.

---

## 2. Phép ẩn dụ dễ hiểu

Hãy tưởng tượng EC2 là một căn nhà.

**Security Group giống như bảo vệ ở cổng.**

Bảo vệ sẽ kiểm tra:

```text
Ai được vào?
Được vào bằng cổng nào?
Được đi ra đâu?
```

Ví dụ với Web Server:

```text
User Internet → EC2
```

Bạn cần cho phép user truy cập website qua:

```text
HTTP  port 80
HTTPS port 443
```

Nhưng không nên cho toàn Internet SSH vào server qua:

```text
SSH port 22
```

---

## 3. Security Group kiểm soát cái gì?

Security Group có 2 phần chính:

### 3.1. Inbound Rules

Kiểm soát traffic **đi vào** resource.

Ví dụ:

```text
Internet → EC2
```

Nếu không mở inbound port 80, user không thể truy cập website.

### 3.2. Outbound Rules

Kiểm soát traffic **đi ra** từ resource.

Ví dụ:

```text
EC2 → Internet
EC2 → RDS
EC2 → S3
```

Mặc định outbound thường được allow all.

---

## 4. Đặc điểm quan trọng của Security Group

### 4.1. Mặc định chặn inbound

Khi mới tạo Security Group:

```text
Inbound: không cho gì vào
Outbound: cho đi ra
```

Nghĩa là EC2 không tự public ra Internet nếu bạn chưa mở rule.

---

### 4.2. Security Group chỉ có Allow, không có Deny

Bạn không viết rule kiểu:

```text
Deny IP A
```

Bạn chỉ viết:

```text
Allow IP B
Allow Security Group C
```

Nếu không được allow thì mặc định bị chặn.

---

### 4.3. Security Group là Stateful

**Stateful** nghĩa là: nếu request được phép đi vào, response tự động được phép đi ra.

Ví dụ bạn mở inbound HTTP 80:

```text
User → EC2 port 80: allowed
EC2 → User response: automatically allowed
```

Bạn không cần mở outbound riêng cho response HTTP.

---

### 4.4. Có thể gắn nhiều Security Group vào một resource

Một EC2 có thể gắn nhiều Security Group.

Ví dụ:

```text
EC2 Web Server
 ├── sg-web-http
 └── sg-admin-ssh
```

Tổng rule được áp dụng là cộng dồn.

---

### 4.5. Có thể reference Security Group khác

Đây là best practice rất quan trọng.

Thay vì allow IP cố định, bạn có thể allow từ một Security Group khác.

Ví dụ:

```text
RDS Security Group:
Allow MySQL 3306 from WebServer Security Group
```

Nghĩa là chỉ EC2 thuộc WebServer SG mới vào được RDS.

Không cần quan tâm IP private của EC2 là gì.

---

# 5. Ví dụ use case thực tế

## Use case 1: EC2 Web Server public Internet

Bạn có một EC2 chạy Apache hoặc Nginx.

User cần truy cập website.

Security Group nên setup:

```text
Inbound:
- HTTP  80   from 0.0.0.0/0
- HTTPS 443  from 0.0.0.0/0
- SSH   22   from YOUR_OFFICE_IP/32

Outbound:
- All traffic hoặc giới hạn theo nhu cầu
```

Không nên:

```text
SSH 22 from 0.0.0.0/0
```

Vì ai trên Internet cũng có thể thử brute-force SSH.

---

## Use case 2: Web App + RDS MySQL

Kiến trúc:

```text
User
  ↓
ALB
  ↓
EC2 Web/App
  ↓
RDS MySQL
```

### ALB Security Group

```text
Inbound:
- HTTP 80 from 0.0.0.0/0
- HTTPS 443 from 0.0.0.0/0

Outbound:
- HTTP 80 to Web EC2 Security Group
```

### EC2 Web Security Group

```text
Inbound:
- HTTP 80 from ALB Security Group
- SSH 22 from Bastion SG hoặc VPN IP

Outbound:
- MySQL 3306 to RDS Security Group
- HTTPS 443 to Internet nếu cần gọi API ngoài
```

### RDS Security Group

```text
Inbound:
- MySQL 3306 from EC2 Web Security Group

Outbound:
- Có thể để default hoặc giới hạn theo nhu cầu
```

Điểm tốt của mô hình này:

```text
User không thể truy cập thẳng EC2
User không thể truy cập thẳng RDS
RDS chỉ nhận kết nối từ EC2 App
```

Đây là mô hình production phổ biến.

---

## Use case 3: Bastion Host SSH vào server private

Kiến trúc:

```text
Admin
  ↓ SSH
Bastion Host public subnet
  ↓ SSH
EC2 App private subnet
```

### Bastion Security Group

```text
Inbound:
- SSH 22 from Office IP / Your IP

Outbound:
- SSH 22 to Private EC2 SG
```

### Private EC2 Security Group

```text
Inbound:
- SSH 22 from Bastion Security Group
- App port from ALB Security Group

Outbound:
- Theo nhu cầu
```

Không nên mở SSH trực tiếp từ Internet vào private app server.

---

## Use case 4: Laravel EC2 upload file lên S3

Security Group không kiểm soát IAM permission, nhưng kiểm soát network.

Nếu EC2 cần gọi S3:

```text
EC2 → S3 HTTPS 443
```

Outbound cần cho phép:

```text
HTTPS 443
```

Best practice hơn:

- Dùng **IAM Role** cho EC2 để cấp quyền S3.
- Dùng **VPC Endpoint for S3** nếu muốn traffic đi nội bộ trong AWS.
- Không lưu Access Key trong `.env`.

---

# 6. AWS Best Practice khi setup Security Group

## 6.1. Chỉ mở port cần thiết

Không mở kiểu này trong production:

```text
All traffic from 0.0.0.0/0
```

Nên mở đúng port:

```text
HTTP 80
HTTPS 443
SSH 22 chỉ từ IP quản trị
MySQL 3306 chỉ từ App SG
Redis 6379 chỉ từ App SG
```

---

## 6.2. Không mở SSH/RDP toàn Internet

Tránh:

```text
SSH 22 from 0.0.0.0/0
RDP 3389 from 0.0.0.0/0
```

Nên dùng một trong các cách:

```text
SSH from office IP /32
SSH through Bastion Host
AWS Systems Manager Session Manager
VPN
```

Với production, nên ưu tiên **SSM Session Manager** để giảm nhu cầu mở port 22.

---

## 6.3. Dùng Security Group reference thay vì IP

Không nên hardcode private IP EC2:

```text
Allow 10.0.1.25/32 to MySQL 3306
```

Nên:

```text
Allow WebApp-SG to MySQL 3306
```

Vì EC2 trong Auto Scaling có thể thay đổi IP liên tục.

---

## 6.4. Tách Security Group theo vai trò

Không nên dùng một SG khổng lồ cho tất cả.

Nên chia:

```text
sg-prod-alb
sg-prod-web
sg-prod-rds
sg-prod-bastion
sg-prod-redis
sg-prod-lambda
```

Mỗi SG đại diện cho một vai trò rõ ràng.

---

## 6.5. Đặt tên và mô tả rule rõ ràng

Ví dụ tốt:

```text
Name: sg-prod-web
Description: Allow HTTP from ALB and SSH from Bastion only
```

Rule description:

```text
Allow HTTP from prod ALB
Allow SSH from bastion host
Allow outbound HTTPS for external API
```

Sau này review sẽ dễ hơn rất nhiều.

---

## 6.6. Không để rule “tạm thời” mãi mãi

Ví dụ lúc debug bạn mở:

```text
SSH 22 from 0.0.0.0/0
```

Sau khi debug xong phải xóa ngay.

Nên có rule description kèm thời hạn:

```text
TEMP SSH for issue INC-1234, remove after 2026-06-20
```

---

## 6.7. Hạn chế outbound nếu môi trường yêu cầu bảo mật cao

Mặc định outbound allow all là tiện, nhưng với hệ thống quan trọng có thể giới hạn.

Ví dụ:

```text
EC2 App outbound:
- 3306 to RDS SG
- 443 to S3 VPC Endpoint
- 443 to Secrets Manager VPC Endpoint
- 443 to CloudWatch Logs endpoint
```

Tuy nhiên, giới hạn outbound cần test kỹ vì app có thể cần gọi package repo, external API, license server, monitoring agent.

---

## 6.8. Không dùng Security Group để thay IAM

Security Group kiểm soát network.

IAM kiểm soát quyền AWS API.

Ví dụ:

```text
Security Group cho EC2 đi ra HTTPS 443
```

Không có nghĩa EC2 được quyền upload S3.

Muốn upload S3 cần:

```text
IAM Role with s3:PutObject
```

---

# 7. Cách quản lý Security Group tốt

## 7.1. Chuẩn hóa naming convention

Gợi ý format:

```text
sg-{env}-{system}-{role}
```

Ví dụ:

```text
sg-prod-videoapp-alb
sg-prod-videoapp-web
sg-prod-videoapp-rds
sg-stg-coupon-web
sg-dev-internal-bastion
```

Nhìn tên là biết:

```text
Môi trường nào?
Dự án nào?
Vai trò gì?
```

---

## 7.2. Dùng tag đầy đủ

Tag nên có:

```text
Environment = production
Project     = video-app
Owner       = backend-team
ManagedBy   = terraform/cloudformation/manual
Purpose     = web-server
```

Tag giúp lọc, audit và tự động hóa.

---

## 7.3. Mỗi Security Group chỉ nên có một mục đích

Không nên:

```text
sg-common-all
```

Vừa mở HTTP, SSH, MySQL, Redis, internal API, external API.

Nên:

```text
sg-prod-web
sg-prod-db
sg-prod-cache
sg-prod-admin
```

Rule càng rõ, rủi ro càng thấp.

---

## 7.4. Review định kỳ

Ít nhất mỗi tháng hoặc mỗi quý nên review:

```text
Có rule 0.0.0.0/0 không?
Có port nhạy cảm đang public không?
Có SG không gắn resource nào không?
Có rule temporary chưa xóa không?
Có description rõ ràng không?
Có resource production dùng chung SG với dev không?
```

Port nhạy cảm cần đặc biệt chú ý:

```text
22    SSH
3389  RDP
3306  MySQL
5432  PostgreSQL
6379  Redis
9200  Elasticsearch/OpenSearch
27017 MongoDB
```

---

## 7.5. Dùng AWS Config hoặc Security Hub để audit

Nên bật các rule kiểm tra như:

```text
Security group không được mở SSH 0.0.0.0/0
Security group không được mở RDP 0.0.0.0/0
RDS không được public
EC2 không được public nếu không cần
```

Dịch vụ nên dùng:

```text
AWS Config
AWS Security Hub
Amazon GuardDuty
IAM Access Analyzer
VPC Reachability Analyzer
```

---

## 7.6. Quản lý bằng Infrastructure as Code

Với production, nên quản lý Security Group bằng:

```text
Terraform
CloudFormation
AWS CDK
```

Lợi ích:

```text
Có version control
Có review Pull Request
Tránh sửa tay nhầm
Rollback dễ hơn
Chuẩn hóa rule
```

Ví dụ thay vì sửa SG trực tiếp trên Console, team tạo Pull Request:

```text
Add rule: allow ALB SG to Web SG on port 80
Reason: expose new web service
Ticket: INFRA-123
```

---

## 7.7. Xóa Security Group không dùng

Security Group cũ dễ gây nhầm lẫn.

Nên định kỳ kiểm tra:

```text
SG nào không attached vào resource nào?
SG nào thuộc project đã ngưng?
SG nào có rule nguy hiểm?
```

Nếu không dùng thì xóa.

---

# 8. Mẫu setup Security Group chuẩn cho Web App Production

Giả sử hệ thống:

```text
Internet
  ↓
ALB
  ↓
EC2 Laravel App
  ↓
RDS MySQL
```

## 8.1. sg-prod-app-alb

Inbound:

```text
HTTP  80   from 0.0.0.0/0
HTTPS 443  from 0.0.0.0/0
```

Outbound:

```text
HTTP 80 to sg-prod-app-web
```

---

## 8.2. sg-prod-app-web

Inbound:

```text
HTTP 80 from sg-prod-app-alb
SSH 22 from sg-prod-bastion hoặc VPN IP
```

Outbound:

```text
MySQL 3306 to sg-prod-app-rds
HTTPS 443 to Internet hoặc VPC Endpoints
```

---

## 8.3. sg-prod-app-rds

Inbound:

```text
MySQL 3306 from sg-prod-app-web
```

Outbound:

```text
Default hoặc giới hạn theo policy nội bộ
```

---

## 8.4. sg-prod-bastion

Inbound:

```text
SSH 22 from Office IP /32
```

Outbound:

```text
SSH 22 to sg-prod-app-web
```

---

# 9. Checklist nhanh khi tạo Security Group

Trước khi lưu Security Group, hỏi 7 câu này:

```text
1. Resource này là public hay private?
2. Có thật sự cần mở port này không?
3. Source có thể dùng Security Group thay vì IP không?
4. Có port nào đang mở 0.0.0.0/0 không?
5. SSH/RDP đã giới hạn IP chưa?
6. Rule có description rõ ràng chưa?
7. SG có tag Owner/Environment/Project chưa?
```

---

# 10. Tóm tắt ngắn gọn

```text
Security Group = firewall ảo cho AWS resource trong VPC.
Inbound = traffic đi vào.
Outbound = traffic đi ra.
Mặc định inbound bị chặn.
Security Group là stateful.
Chỉ có Allow rule, không có Deny rule.
Best practice là chỉ mở đúng port, đúng source, đúng mục đích.
```

Cách nhớ dễ nhất:

```text
ALB được public.
EC2 chỉ nhận traffic từ ALB.
RDS chỉ nhận traffic từ EC2.
SSH chỉ từ Bastion/VPN/SSM.
```

Đây là tư duy setup Security Group rất chuẩn cho production.
