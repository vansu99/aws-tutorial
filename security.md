# AWS Security - Giải thích dễ hiểu

Dễ hiểu nhất: **AWS Security = bảo vệ tài khoản, network, server, data, application và log/audit**.

Cứ nhớ theo 5 lớp này là không bị rối:

```text
1. Identity Security    = Ai được quyền làm gì?
2. Network Security     = Traffic được đi đâu?
3. Data Security        = Dữ liệu có được mã hóa/bảo vệ không?
4. Application Security = Web/API có bị attack không?
5. Detection & Audit    = Có ai làm gì bất thường không?
```

---

## 1. Identity Security — IAM

**IAM = quản lý user, role, permission.**

Hiểu đơn giản:

```text
IAM User   = người dùng cố định
IAM Group  = nhóm user
IAM Role   = quyền tạm thời cho EC2/Lambda/service/user assume
Policy     = rule cho phép hoặc từ chối hành động
```

Ví dụ:

```text
User A chỉ được xem S3
User B được quản lý EC2
EC2 Role được phép upload backup lên S3
Lambda Role được phép ghi log vào CloudWatch
```

Best practice:

| Việc nên làm | Giải thích |
|---|---|
| Bật MFA | Bảo vệ login |
| Dùng IAM Role thay vì Access Key | Giảm rủi ro lộ key |
| Least Privilege | Chỉ cấp quyền cần thiết |
| Không dùng root account hằng ngày | Root chỉ dùng việc đặc biệt |
| Rotate access key | Đổi key định kỳ |
| Dùng IAM Identity Center nếu nhiều user | Quản lý login tập trung |

Ví dụ policy sai nguy hiểm:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

Nghĩa là **full access toàn bộ AWS**. Production không nên cấp bừa như vậy.

---

## 2. Network Security — SG, NACL, WAF, Network Firewall

### Security Group

**Security Group = firewall gắn vào resource**, ví dụ EC2, RDS, ALB.

Ví dụ chuẩn:

```text
ALB SG:
  Allow 443 from Internet

App EC2 SG:
  Allow 80/8080 from ALB SG only

RDS SG:
  Allow 3306 from App EC2 SG only
```

Không nên:

```text
RDS allow 3306 from 0.0.0.0/0 ❌
```

### NACL

**NACL = firewall ở subnet level.**

Dùng khi muốn chặn hoặc allow traffic ở cấp subnet.

```text
Security Group = khóa cửa từng server
NACL           = cổng bảo vệ của cả khu subnet
```

### AWS WAF

**WAF = bảo vệ web/app/API khỏi request xấu.**

Dùng với:

```text
CloudFront
ALB
API Gateway
AppSync
```

Dùng để chặn:

| Loại attack | Ví dụ |
|---|---|
| SQL Injection | `' OR 1=1` |
| XSS | Script độc hại |
| Bad bot | Bot spam request |
| Rate limit | 1 IP request quá nhiều |
| Geo block | Chặn quốc gia cụ thể |

Ví dụ:

```text
User → CloudFront + WAF → ALB → EC2
```

### AWS Network Firewall

**Network Firewall = firewall nâng cao cho VPC.**

Dùng khi cần kiểm soát traffic kỹ hơn Security Group/NACL, ví dụ:

```text
Inspect traffic VPC-to-VPC
Filter outbound traffic
Filter Internet traffic
Protect VPN / Direct Connect traffic
Block known bad IP/domain
IDS/IPS
```

---

## 3. Data Security — KMS, Encryption, Secrets Manager

### KMS

**KMS = dịch vụ quản lý encryption key.**

Dùng để mã hóa:

```text
S3
EBS
EFS
RDS
Secrets Manager
CloudWatch Logs
Snapshot
Backup
```

Ví dụ:

```text
S3 bucket bật SSE-KMS
EBS volume bật encryption
RDS bật encryption at rest
```

Best practice:

| Việc nên làm | Lý do |
|---|---|
| Encrypt at rest | Dữ liệu lưu trữ được mã hóa |
| Encrypt in transit | Dùng HTTPS/TLS |
| Dùng KMS key có kiểm soát | Quản lý ai được decrypt |
| Không hardcode secret trong code | Tránh lộ password/API key |

### Secrets Manager

**Secrets Manager = nơi lưu password/API key/database credential an toàn.**

Ví dụ không nên làm:

```env
DB_PASSWORD=plain-text-password
```

Nên làm:

```text
App → Secrets Manager → lấy DB password
```

Secrets Manager có thể:

| Chức năng | Ý nghĩa |
|---|---|
| Store secret | Lưu password/API key |
| Encrypt bằng KMS | Mã hóa secret |
| Rotate secret | Tự đổi password định kỳ |
| Audit access | Biết ai lấy secret |
| Cross-region replication | Copy secret sang region khác |

---

## 4. Application Security

Ứng dụng thường cần các lớp bảo vệ sau:

```text
Route 53
  ↓
CloudFront
  ↓
AWS WAF
  ↓
ALB
  ↓
EC2/ECS/Lambda
  ↓
RDS
```

Nên có:

| Layer | Service |
|---|---|
| HTTPS certificate | ACM |
| CDN + edge protection | CloudFront |
| Web firewall | WAF |
| DDoS basic protection | Shield Standard |
| DDoS advanced protection | Shield Advanced |
| Secret management | Secrets Manager |
| Vulnerability scan | Inspector |
| Central findings | Security Hub |

---

## 5. Detection & Audit — phát hiện và điều tra

### CloudTrail

**CloudTrail = log ai đã làm gì trong AWS account.**

Ví dụ log:

```text
Ai tạo EC2?
Ai xóa S3 bucket?
Ai tạo access key?
Ai gọi Bedrock API?
Ai sửa Security Group?
```

Dùng CloudTrail để điều tra sự cố bảo mật/billing là rất quan trọng.

### GuardDuty

**GuardDuty = dịch vụ phát hiện threat tự động.**

Nó phân tích log để phát hiện:

```text
API call bất thường
Credential bị lộ
EC2 giao tiếp với IP độc hại
Crypto mining behavior
S3 access bất thường
Malware/suspicious activity
```

### Security Hub

**Security Hub = dashboard trung tâm gom cảnh báo security.**

Nó gom findings từ:

```text
GuardDuty
Inspector
Macie
IAM Access Analyzer
Firewall Manager
Partner tools
```

Dùng để xem security posture toàn account/multi-account.

### Inspector

**Inspector = scan lỗ hổng bảo mật.**

Dùng để check:

```text
EC2 có package lỗi bảo mật không?
Container image trong ECR có CVE không?
Lambda dependency có vulnerability không?
```

### Macie

**Macie = tìm dữ liệu nhạy cảm trong S3.**

Dùng để phát hiện:

```text
PII
Email
Phone number
Credential
Financial data
Sensitive document
```

### Detective

**Detective = điều tra nguyên nhân security incident.**

Dễ hiểu:

```text
GuardDuty báo có dấu hiệu bất thường
↓
Detective giúp phân tích resource, user, IP, API call liên quan
↓
Tìm root cause nhanh hơn
```

---

## 6. Compliance & Governance

### AWS Config

**AWS Config = theo dõi cấu hình resource có đúng chuẩn không.**

Ví dụ rule:

```text
S3 bucket không được public
EBS phải bật encryption
RDS phải bật backup
Security Group không được mở SSH 0.0.0.0/0
CloudTrail phải enabled
```

Nếu resource sai chuẩn, AWS Config báo non-compliant.

### Organizations SCP

**SCP = guardrail cấp account/OU.**

Ví dụ:

```text
Deny dùng Bedrock
Deny tạo resource ngoài ap-northeast-1
Deny tạo EC2 instance quá lớn
Deny disable CloudTrail
```

Rất hữu ích để chống phát sinh cost/security incident.

### IAM Access Analyzer

**Access Analyzer = phát hiện resource bị share public/cross-account.**

Ví dụ:

```text
S3 bucket public
KMS key cho account khác dùng
IAM role cho external account assume
```

---

## 7. Pricing model — service nào tốn phí?

### Thường miễn phí hoặc không tính phí trực tiếp

| Service / Feature | Pricing |
|---|---|
| IAM User/Role/Policy | Không tính phí |
| Security Group | Không tính phí |
| NACL | Không tính phí |
| IAM Access Analyzer basic | Có phần miễn phí cho external access analyzer |
| Shield Standard | Miễn phí mặc định |
| ACM public certificate | Miễn phí khi dùng với AWS integrated services |

### Có tính phí

| Service | Tính phí theo |
|---|---|
| KMS | Key/month + API request |
| Secrets Manager | Số secret/month + API calls |
| CloudTrail | Management event trail thường có free copy; data events/extra trails/storage có phí |
| Config | Configuration items + rule evaluations |
| GuardDuty | Lượng log/event/data analyzed |
| Security Hub | Số security checks + findings ingested |
| Inspector | Số EC2/ECR image/Lambda scanned |
| Macie | S3 bucket/object monitoring + sensitive data discovery |
| WAF | Web ACL + rules + request inspected |
| Shield Advanced | Monthly fee + data transfer/usage liên quan |
| Network Firewall | Firewall endpoint/hour + data processed |
| Firewall Manager | Policy/resource usage, phụ thuộc service được quản lý |
| CloudWatch Logs | Ingestion + storage + query |
| VPC Flow Logs | Phí lưu log trong CloudWatch/S3/Athena query |

Điểm cần nhớ: **bật security service rất tốt, nhưng phải biết pricing**. Đặc biệt các service như GuardDuty, Macie, Config, CloudTrail data events, Network Firewall, WAF có thể tăng cost theo volume.

---

## 8. Security architecture mẫu cho production

```text
Users
  ↓
Route 53
  ↓
CloudFront
  ↓
AWS WAF
  ↓
Application Load Balancer - Public Subnet
  ↓
EC2/ECS App - Private Subnet
  ↓
RDS - Private DB Subnet
```

Security layers:

```text
Identity:
  IAM least privilege + MFA + role-based access

Network:
  VPC private subnet + SG + NACL + VPC Endpoint

Data:
  KMS encryption + Secrets Manager + S3 block public access

Application:
  WAF + HTTPS + ACM + secure headers

Detection:
  CloudTrail + GuardDuty + Security Hub + Config

Response:
  CloudWatch Alarm + EventBridge + SNS/Lambda remediation
```

---

## 9. Checklist security cơ bản cho AWS account

| Mục | Nên làm |
|---|---|
| Root account | Bật MFA, không dùng hằng ngày |
| IAM users | Không cấp AdministratorAccess nếu không cần |
| Access keys | Rotate, xóa key không dùng |
| CloudTrail | Enable all regions |
| GuardDuty | Enable |
| Security Hub | Enable nếu cần central dashboard |
| S3 | Block Public Access |
| EBS/RDS/S3/EFS | Enable encryption |
| Security Group | Không mở SSH/RDP/DB ra 0.0.0.0/0 |
| Secrets | Dùng Secrets Manager hoặc SSM Parameter Store |
| Budget | Tạo budget + cost anomaly detection |
| Config | Check compliance resource |
| Backup | AWS Backup / snapshot / S3 lifecycle |
| VPC Flow Logs | Bật cho VPC quan trọng |

---

## 11. Câu nhớ nhanh nhất

```text
IAM             = ai được quyền làm gì
Security Group  = firewall của resource
NACL            = firewall của subnet
KMS             = quản lý key mã hóa
Secrets Manager = lưu password/API key an toàn
WAF             = bảo vệ web/API
Shield          = chống DDoS
CloudTrail      = log ai làm gì
GuardDuty       = phát hiện hành vi bất thường
Security Hub    = dashboard security trung tâm
Inspector       = scan lỗ hổng
Macie           = tìm dữ liệu nhạy cảm trong S3
Config          = kiểm tra resource có đúng chuẩn không
```

Với production AWS, tối thiểu nên có:

```text
MFA + IAM least privilege
CloudTrail all regions
GuardDuty
S3 Block Public Access
Encryption bằng KMS
Security Group chặt
Secrets Manager/SSM Parameter Store
AWS Budgets + Cost Anomaly Detection
```
