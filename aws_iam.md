# AWS Identity and Access Management (IAM)

> **Mục tiêu bài học:** Sau bài này, bạn sẽ hiểu IAM là gì, IAM kiểm soát quyền trong AWS như thế nào, và có thể thực hành tạo **IAM Role cho EC2 truy cập S3 mà không cần hardcode Access Key**.

---

# 1. Khái niệm và phép ẩn dụ

## AWS IAM là gì?

**AWS IAM — Identity and Access Management** — là dịch vụ dùng để quản lý:

```text
Ai được truy cập AWS?
Được làm hành động gì?
Trên tài nguyên nào?
Trong điều kiện nào?
```

Nói ngắn gọn:

> **IAM là hệ thống phân quyền trung tâm của AWS.**

IAM không trực tiếp chạy server, không lưu file, không xử lý database. Nhưng gần như mọi thao tác với AWS đều phải đi qua IAM để kiểm tra quyền.

Ví dụ:

| Câu hỏi                                          | IAM kiểm soát                  |
| ------------------------------------------------ | ------------------------------ |
| Ai được login AWS Console?                       | IAM User / IAM Identity Center |
| Ai được start EC2?                               | IAM Policy                     |
| Lambda có được đọc S3 không?                     | IAM Role                       |
| EC2 có được lấy secret từ Secrets Manager không? | IAM Role + Policy              |
| User có bắt buộc dùng MFA không?                 | IAM Condition                  |
| Có được tạo resource ở region khác không?        | IAM Policy / SCP               |

---

## Phép ẩn dụ: IAM giống hệ thống quản lý nhân viên trong công ty

Hãy tưởng tượng AWS Account là một **tòa nhà công ty**.

| Trong công ty                | Trong AWS IAM  |
| ---------------------------- | -------------- |
| Nhân viên                    | IAM User       |
| Phòng ban                    | IAM Group      |
| Thẻ ra vào                   | IAM Credential |
| Chìa khóa phòng              | IAM Permission |
| Nội quy ra vào               | IAM Policy     |
| Người được ủy quyền tạm thời | IAM Role       |
| Camera audit                 | CloudTrail     |
| Xác thực 2 lớp               | MFA            |

Ví dụ:

Một nhân viên phòng Dev có thể:

```text
Vào phòng Dev
Không được vào phòng Kế toán
Không được mở két sắt
Chỉ được thao tác trong giờ làm việc
```

Trong AWS, tương tự:

```text
Developer được start/stop EC2 ở môi trường Dev
Không được xóa RDS Production
Không được tạo user IAM mới
Chỉ được thao tác nếu có MFA
```

---

## Vì sao IAM là nền tảng bảo mật quan trọng nhất trong AWS?

Vì nếu IAM cấu hình sai, hậu quả có thể rất lớn:

```text
Sai quyền IAM
→ user/service có quyền quá rộng
→ xóa nhầm tài nguyên
→ lộ data S3
→ lộ secret
→ bị tạo resource tốn tiền
→ bị truy cập cross-account trái phép
```

Trong thực tế, nhiều sự cố AWS không bắt đầu từ lỗi server, mà bắt đầu từ:

```text
Access Key bị lộ
Policy quá rộng
User có AdministratorAccess
S3 bucket public
Role trust policy sai
Không bật MFA
```

IAM là “cửa chính” của AWS. Cửa chính mà mở toang thì các service phía sau dù cấu hình tốt vẫn rất nguy hiểm.

---

## IAM kiểm soát theo mô hình nào?

IAM trả lời 4 câu hỏi chính:

```text
Who?       → Principal
Can do?    → Action
On what?   → Resource
When/how?  → Condition
```

Ví dụ:

```text
User Alice được phép đọc object trong bucket my-app-logs,
nhưng chỉ khi truy cập từ IP công ty và đã bật MFA.
```

Biểu diễn theo IAM:

| Thành phần | Ý nghĩa                      |
| ---------- | ---------------------------- |
| Principal  | Alice                        |
| Action     | `s3:GetObject`               |
| Resource   | `arn:aws:s3:::my-app-logs/*` |
| Condition  | IP công ty + MFA             |

---

# 2. Các thành phần then chốt của IAM

## Tổng quan nhanh

| Thành phần | Dễ hiểu là                 | Dùng để làm gì                        |
| ---------- | -------------------------- | ------------------------------------- |
| IAM User   | Một tài khoản cụ thể       | Cho người hoặc app truy cập AWS       |
| IAM Group  | Nhóm user                  | Gán quyền chung cho nhiều user        |
| IAM Role   | Bộ quyền tạm thời          | Cho EC2, Lambda, account khác sử dụng |
| IAM Policy | File JSON định nghĩa quyền | Allow/Deny hành động                  |
| Permission | Quyền cụ thể               | Ví dụ đọc S3, start EC2               |
| Principal  | Ai thực hiện request       | User, Role, Service                   |
| Action     | Hành động AWS API          | `s3:GetObject`, `ec2:StartInstances`  |
| Resource   | Tài nguyên bị tác động     | S3 bucket, EC2 instance, secret       |
| Condition  | Điều kiện áp dụng quyền    | IP, MFA, Region, tag                  |
| MFA        | Xác thực nhiều lớp         | Tăng bảo mật khi đăng nhập/thao tác   |

---

## 2.1 IAM User

**IAM User** là identity đại diện cho một người dùng hoặc một ứng dụng cần truy cập AWS.

Ví dụ:

```text
alice
bob
deploy-user
s3-upload-user
```

IAM User có thể có:

| Loại truy cập           | Dùng cho                                |
| ----------------------- | --------------------------------------- |
| Console password        | Đăng nhập AWS Console                   |
| Access Key / Secret Key | Truy cập bằng AWS CLI, SDK, source code |
| MFA                     | Bảo mật thêm khi login                  |

Ví dụ thực tế:

```text
Developer A cần login AWS Console để xem EC2
→ tạo IAM User cho Developer A
```

Tuy nhiên, với ứng dụng chạy trên EC2/Lambda, **không nên dùng IAM User + Access Key**. Nên dùng **IAM Role**.

---

## 2.2 IAM Group

**IAM Group** là nhóm chứa nhiều IAM User.

Ví dụ:

```text
Developers
Operators
Auditors
BillingViewers
```

Bạn attach policy vào group, user trong group sẽ có quyền đó.

Ví dụ:

```text
Group Developers
→ được quyền thao tác EC2/S3 môi trường Dev

Group Auditors
→ chỉ được xem log, không được chỉnh sửa
```

Điểm cần nhớ:

```text
Group chỉ chứa User
Group không chứa Group khác
Một User có thể thuộc nhiều Group
```

Ví dụ:

```text
Alice thuộc group Developers
Alice cũng thuộc group BillingViewers
→ Alice có tổng quyền từ cả hai group
```

---

## 2.3 IAM Role

**IAM Role** là bộ quyền có thể được “assume” tạm thời bởi:

```text
AWS Service
IAM User
IAM Role khác
Account khác
Federated user
```

Nói dễ hiểu:

> IAM Role giống như “thẻ quyền tạm thời” được cấp khi cần làm một việc cụ thể.

Ví dụ:

```text
EC2 cần đọc file từ S3
→ gắn IAM Role vào EC2
→ EC2 tự nhận temporary credentials
→ không cần lưu Access Key trong source code
```

### Vì sao Role tốt hơn Access Key?

| Access Key hardcode        | IAM Role                               |
| -------------------------- | -------------------------------------- |
| Dễ bị lộ trong source code | Không cần lưu key                      |
| Key sống lâu               | Credential tạm thời                    |
| Phải tự rotate             | AWS tự cấp/rotate temporary credential |
| Rủi ro cao khi leak        | An toàn hơn nhiều                      |
| Khó audit theo workload    | Gắn trực tiếp vào service              |

Best practice:

```text
App chạy trên EC2/Lambda/ECS
→ dùng IAM Role
Không hardcode Access Key
```

---

## 2.4 IAM Policy

**IAM Policy** là tài liệu JSON định nghĩa quyền.

Policy trả lời:

```text
Allow hay Deny?
Cho hành động nào?
Trên resource nào?
Với điều kiện gì?
```

Ví dụ policy đơn giản:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    }
  ]
}
```

Policy trên nghĩa là:

```text
Cho phép đọc object trong bucket my-app-bucket
```

---

## 2.5 Permission

**Permission** là quyền thực hiện một hành động cụ thể.

Ví dụ:

| Permission                      | Ý nghĩa              |
| ------------------------------- | -------------------- |
| `s3:GetObject`                  | Đọc object trong S3  |
| `s3:PutObject`                  | Upload object vào S3 |
| `ec2:StartInstances`            | Start EC2            |
| `ec2:StopInstances`             | Stop EC2             |
| `secretsmanager:GetSecretValue` | Đọc secret           |
| `rds:DescribeDBInstances`       | Xem thông tin RDS    |

Permission không tự tồn tại riêng lẻ, thường nằm trong **IAM Policy**.

---

## 2.6 Principal

**Principal** là đối tượng thực hiện hành động.

Ví dụ Principal:

```text
IAM User
IAM Role
AWS Service
AWS Account
Federated User
```

Ví dụ trong resource-based policy như S3 bucket policy:

```json
"Principal": {
  "AWS": "arn:aws:iam::123456789012:role/ReadLogRole"
}
```

Nghĩa là:

```text
Role ReadLogRole từ account 123456789012 được phép truy cập.
```

---

## 2.7 Action

**Action** là hành động AWS API.

Ví dụ:

```text
s3:GetObject
s3:PutObject
s3:ListBucket
ec2:StartInstances
ec2:TerminateInstances
iam:CreateUser
```

Có thể dùng wildcard, nhưng phải cẩn thận:

```json
"Action": "s3:*"
```

Nghĩa là:

```text
Cho phép tất cả hành động S3
```

Không nên dùng nếu không thật sự cần.

---

## 2.8 Resource

**Resource** là tài nguyên AWS bị tác động.

Ví dụ ARN của S3 bucket:

```text
arn:aws:s3:::my-app-bucket
```

ARN của object trong bucket:

```text
arn:aws:s3:::my-app-bucket/*
```

Khác biệt rất quan trọng:

| Resource                       | Ý nghĩa                 |
| ------------------------------ | ----------------------- |
| `arn:aws:s3:::my-app-bucket`   | Chính bucket            |
| `arn:aws:s3:::my-app-bucket/*` | Object bên trong bucket |

Ví dụ:

```text
s3:ListBucket cần resource là bucket
s3:GetObject cần resource là object /*
```

---

## 2.9 Condition

**Condition** là điều kiện để policy có hiệu lực.

Ví dụ điều kiện theo MFA:

```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  }
}
```

Ví dụ điều kiện theo IP:

```json
"Condition": {
  "IpAddress": {
    "aws:SourceIp": "203.0.113.0/24"
  }
}
```

Ví dụ điều kiện theo Region:

```json
"Condition": {
  "StringEquals": {
    "aws:RequestedRegion": "ap-northeast-1"
  }
}
```

Ví dụ điều kiện theo tag:

```json
"Condition": {
  "StringEquals": {
    "aws:ResourceTag/Environment": "dev"
  }
}
```

Condition giúp IAM policy từ “cho phép chung chung” thành “cho phép có kiểm soát”.

---

## 2.10 MFA

**MFA — Multi-Factor Authentication** — là xác thực nhiều lớp.

Thay vì chỉ cần password:

```text
Password
```

MFA yêu cầu thêm thiết bị xác thực:

```text
Password + MFA code
```

Ví dụ MFA device:

```text
Google Authenticator
Authy
Microsoft Authenticator
YubiKey
Hardware MFA
```

Best practice:

```text
Root account: bắt buộc bật MFA
Admin user: bắt buộc bật MFA
User có quyền production: nên bắt buộc MFA
```

---

# 3. Ví dụ thực hành: Tạo IAM Role cho EC2 truy cập S3 không cần hardcode Access Key

## Kịch bản

Bạn có một app Laravel chạy trên EC2.

App cần đọc file từ S3 bucket:

```text
my-app-private-bucket
```

Cách sai thường gặp:

```env
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
```

Vấn đề:

```text
Key có thể bị leak qua Git
Key có thể bị lộ trong .env
Key sống lâu
Khó rotate
Nếu bị lộ có thể gây sự cố bảo mật và chi phí
```

Cách đúng:

```text
Tạo IAM Role
Attach Role vào EC2
App/CLI trên EC2 tự lấy temporary credential
Không cần hardcode Access Key
```

---

## Kiến trúc đơn giản

```text
EC2 Instance
   |
   | Assume IAM Role tự động
   v
IAM Role: EC2S3ReadOnlyRole
   |
   | Policy cho phép s3:GetObject
   v
S3 Bucket: my-app-private-bucket
```

---

## Bước 1: Tạo IAM Policy cho phép đọc object từ S3 bucket cụ thể

Policy theo nguyên tắc **Least Privilege**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListSpecificBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::my-app-private-bucket"
    },
    {
      "Sid": "ReadObjectsFromSpecificBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-app-private-bucket/*"
    }
  ]
}
```

Policy này cho phép:

```text
List bucket my-app-private-bucket
Read object trong bucket đó
Không cho upload
Không cho delete
Không cho truy cập bucket khác
```

---

## Bước 2: Tạo IAM Role cho EC2

Vào AWS Console:

```text
IAM
→ Roles
→ Create role
→ Trusted entity type: AWS service
→ Use case: EC2
→ Next
```

Chọn policy vừa tạo:

```text
EC2ReadSpecificS3BucketPolicy
```

Đặt tên role:

```text
EC2S3ReadOnlyRole
```

---

## Bước 3: Attach Policy vào Role

Nếu khi tạo Role bạn đã chọn policy thì xong.

Nếu chưa:

```text
IAM
→ Roles
→ EC2S3ReadOnlyRole
→ Add permissions
→ Attach policies
→ chọn EC2ReadSpecificS3BucketPolicy
```

---

## Bước 4: Gắn Role vào EC2 Instance

Vào EC2 Console:

```text
EC2
→ Instances
→ Chọn instance
→ Actions
→ Security
→ Modify IAM role
→ Chọn EC2S3ReadOnlyRole
→ Update IAM role
```

Không cần restart EC2.

---

## Bước 5: SSH vào EC2 và test bằng AWS CLI

SSH vào EC2:

```bash
ssh ec2-user@<EC2_PUBLIC_IP>
```

Kiểm tra identity hiện tại:

```bash
aws sts get-caller-identity
```

Kết quả mong muốn sẽ thấy Role dạng:

```json
{
  "UserId": "AROA...",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/EC2S3ReadOnlyRole/i-xxxxxxxx"
}
```

Test list bucket:

```bash
aws s3 ls s3://my-app-private-bucket
```

Test đọc file:

```bash
aws s3 cp s3://my-app-private-bucket/test.txt .
```

Nếu thành công:

```text
download: s3://my-app-private-bucket/test.txt to ./test.txt
```

---

## Lưu ý cho Laravel / SDK

Nếu dùng AWS SDK trong Laravel, khi chạy trên EC2 có IAM Role, bạn **không cần set Access Key** trong `.env`.

Thay vì:

```env
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=yyy
AWS_DEFAULT_REGION=ap-northeast-1
AWS_BUCKET=my-app-private-bucket
```

Nên để:

```env
AWS_DEFAULT_REGION=ap-northeast-1
AWS_BUCKET=my-app-private-bucket
```

AWS SDK sẽ tự tìm credential theo thứ tự, trong đó có:

```text
EC2 Instance Metadata Service
→ IAM Role attached to EC2
→ Temporary credentials
```

Đây là cách chuẩn hơn nhiều cho production.

---

# 4. AWS Best Practices cho IAM

## 4.1 Không dùng root account cho công việc hằng ngày

Root account có quyền tối cao trong AWS Account.

Root nên chỉ dùng cho việc đặc biệt như:

```text
Thiết lập account ban đầu
Đổi thông tin billing quan trọng
Đóng account
Một số thao tác chỉ root mới làm được
```

Không nên dùng root để:

```text
Tạo EC2
Tạo S3
Deploy app
Tạo IAM user hằng ngày
```

Best practice:

```text
Root account
→ bật MFA
→ cất credential an toàn
→ không dùng hằng ngày
```

---

## 4.2 Bật MFA cho root account và user quan trọng

Các user nên bật MFA:

```text
Root account
Admin user
User có quyền production
User có quyền billing
User có quyền IAM
User có quyền delete resource
```

MFA giúp giảm rủi ro khi password bị lộ.

---

## 4.3 Áp dụng Least Privilege

**Least Privilege** nghĩa là:

```text
Chỉ cấp đúng quyền cần thiết
Không cấp dư
Không cấp quyền admin nếu không cần
```

Ví dụ xấu:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

Ví dụ tốt hơn:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::my-app-private-bucket/*"
}
```

Tư duy đúng:

```text
Cần đọc S3?
→ chỉ cấp s3:GetObject

Cần upload?
→ thêm s3:PutObject

Không cần delete?
→ không cấp s3:DeleteObject
```

---

## 4.4 Dùng IAM Role thay vì hardcode Access Key

Với workload chạy trên AWS:

| Workload       | Nên dùng              |
| -------------- | --------------------- |
| EC2            | IAM Role for EC2      |
| Lambda         | Lambda Execution Role |
| ECS Task       | ECS Task Role         |
| EKS Pod        | IRSA / Pod Identity   |
| CloudFormation | Service Role          |

Không nên hardcode:

```text
Access Key trong source code
Access Key trong .env
Access Key trong GitHub Actions nếu có thể dùng OIDC
Access Key trong Docker image
```

---

## 4.5 Xoay vòng Access Key nếu bắt buộc phải dùng

Nếu bắt buộc dùng Access Key, ví dụ server ngoài AWS cần gọi AWS API, thì nên:

```text
Rotate định kỳ
Không share key qua chat/email
Không commit vào Git
Không dùng chung key cho nhiều app
Gắn quyền tối thiểu
Theo dõi last used
Xóa key không dùng
```

Quy trình rotate cơ bản:

```text
Tạo key mới
Update app dùng key mới
Test app
Disable key cũ
Theo dõi lỗi
Delete key cũ
```

---

## 4.6 Không dùng AdministratorAccess nếu không thật sự cần

`AdministratorAccess` rất tiện, nhưng rất nguy hiểm.

Policy này gần như cho phép toàn bộ hành động:

```text
Tạo/xóa EC2
Tạo/xóa S3
Tạo IAM User
Gắn quyền admin cho user khác
Đọc secret
Tạo resource tốn tiền
```

Chỉ nên cấp admin cho rất ít người, và phải có:

```text
MFA
CloudTrail audit
Quy trình approval
Review định kỳ
```

---

## 4.7 Dùng IAM Access Analyzer

IAM Access Analyzer giúp phát hiện quyền có thể truy cập từ bên ngoài, ví dụ:

```text
S3 bucket cho cross-account
KMS key cho account khác
IAM Role cho external principal assume
SQS queue public/cross-account
Secrets Manager resource policy mở rộng
```

Dùng để kiểm tra:

```text
Có resource nào public không?
Có resource nào cho account khác truy cập không?
Policy có quá rộng không?
```

---

## 4.8 Dùng tag, naming convention và group để quản lý quyền

Ví dụ naming convention:

```text
dev-developer-readonly
prod-operator-limited
ec2-s3-readonly-role
lambda-secretsmanager-read-role
crossaccount-log-read-role
```

Ví dụ tag:

```text
Environment = dev
Environment = stg
Environment = prod
Project = video-app
Owner = backend-team
```

Khi có tag tốt, bạn có thể viết policy theo tag.

Ví dụ:

```text
Developer chỉ được thao tác EC2 có tag Environment=dev
```

---

## 4.9 Review IAM permission định kỳ

Nên review:

```text
User nào không còn dùng?
Access Key nào lâu không dùng?
User nào có quyền admin?
Role nào có trust policy quá rộng?
Policy nào có Action "*"?
Policy nào có Resource "*"?
```

AWS cung cấp các công cụ audit như:

```text
IAM Credential Report
IAM Access Advisor
IAM Access Analyzer
CloudTrail
```

---

## 4.10 Sử dụng CloudTrail để audit

CloudTrail giúp trả lời:

```text
Ai đã làm gì?
Làm lúc nào?
Từ IP nào?
Dùng user/role nào?
Gọi API gì?
Thành công hay thất bại?
```

Ví dụ cần điều tra:

```text
Ai đã xóa S3 object?
Ai đã tạo EC2 instance lớn?
Ai đã tạo Access Key?
Ai đã thay đổi Security Group?
Ai đã disable CloudTrail?
```

IAM quyết định quyền, còn CloudTrail giúp audit hành động.

---

# 5. Các use case thực tế trong doanh nghiệp

## Kịch bản 1: Developer chỉ được deploy EC2/S3 ở môi trường Development

### Bài toán

Công ty có 3 môi trường:

```text
Development
Staging
Production
```

Developer chỉ được thao tác ở Development, không được đụng vào Production.

### Cách thiết kế

Gắn tag cho resource:

```text
Environment = dev
Environment = prod
```

Tạo group:

```text
Developers
```

Gắn policy cho group Developers:

```text
Cho phép thao tác EC2/S3 với resource có tag Environment=dev
Không cho delete hoặc thay đổi production
```

### Ví dụ policy ý tưởng

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2ActionsOnDevResources",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Environment": "dev"
        }
      }
    }
  ]
}
```

### Giá trị thực tế

```text
Developer làm việc được ở môi trường dev
Giảm rủi ro ảnh hưởng production
Dễ audit quyền
Dễ mở rộng khi team lớn hơn
```

---

## Kịch bản 2: EC2 hoặc Lambda truy cập S3/RDS/Secrets Manager bằng IAM Role

### Bài toán

Ứng dụng cần:

```text
Đọc file từ S3
Lấy DB password từ Secrets Manager
Ghi log ra CloudWatch
```

Cách sai:

```text
Hardcode Access Key trong source code
Hardcode password trong .env
Dùng chung một IAM User cho nhiều app
```

Cách đúng:

```text
EC2 gắn IAM Role
Lambda dùng Execution Role
Role chỉ có quyền cần thiết
Secret lưu trong Secrets Manager
```

### Ví dụ Role cho Lambda đọc secret

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSpecificSecret",
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:prod/db/password-*"
    }
  ]
}
```

### Giá trị thực tế

```text
Không lộ credential
Dễ rotate secret
Dễ audit app nào đọc secret nào
Giảm thiểu blast radius nếu app bị compromise
```

---

## Kịch bản 3: Cross-account để account quản trị đọc log hoặc backup từ production

### Bài toán

Doanh nghiệp có nhiều AWS Account:

```text
Management Account
Production Account
Security/Log Archive Account
Backup Account
```

Team Security cần đọc log từ Production, nhưng không nên có full quyền production.

### Cách thiết kế

Trong Production Account:

```text
Tạo IAM Role: CrossAccountLogReadRole
Trust account quản trị assume role này
Gắn policy chỉ cho đọc log/S3 backup cụ thể
```

Trong Management/Security Account:

```text
User hoặc Role được quyền sts:AssumeRole sang Production
```

### Trust policy ví dụ ở Production Account

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSecurityAccountAssumeRole",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Permission policy ví dụ cho role đọc log

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadProductionLogBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::prod-cloudtrail-logs"
    },
    {
      "Sid": "ReadProductionLogObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::prod-cloudtrail-logs/*"
    }
  ]
}
```

### Giá trị thực tế

```text
Tách biệt production và security account
Không cần tạo IAM User trong production
Quyền tạm thời qua AssumeRole
Dễ audit bằng CloudTrail
Giảm rủi ro truy cập quá quyền
```

---

# 6. Cách IAM ra quyết định Allow/Deny

Đây là phần cực kỳ quan trọng.

Khi một request gửi đến AWS, IAM sẽ đánh giá:

```text
Có explicit Deny không?
Nếu có → Deny luôn

Nếu không có Deny:
Có Allow phù hợp không?
Nếu có → Allow

Nếu không có Allow:
Mặc định → Deny
```

Tóm tắt:

| Trường hợp                           | Kết quả |
| ------------------------------------ | ------- |
| Không có policy allow                | Deny    |
| Có Allow                             | Allow   |
| Có Allow nhưng cũng có explicit Deny | Deny    |
| Có Deny ở bất kỳ policy liên quan    | Deny    |

Câu thần chú dễ nhớ:

```text
Deny thắng Allow.
Không Allow thì mặc định Deny.
```

Ví dụ:

```text
Group Developers cho phép Start EC2
Nhưng user Alice có policy Deny Start EC2 ở production
→ Alice không start được EC2 production
```

---

# 7. IAM User vs IAM Role: Khi nào dùng cái nào?

| Tiêu chí                       | IAM User                            | IAM Role                        |
| ------------------------------ | ----------------------------------- | ------------------------------- |
| Đại diện cho                   | Người dùng hoặc app legacy          | Quyền tạm thời                  |
| Credential                     | Password / Access Key               | Temporary credential            |
| Phù hợp cho                    | Human login, một số external system | EC2, Lambda, ECS, cross-account |
| Có nên hardcode?               | Không                               | Không cần                       |
| Rotation                       | Phải tự quản lý                     | AWS tự cấp temporary credential |
| Best practice cho AWS workload | Không ưu tiên                       | Nên dùng                        |

Kết luận thực tế:

```text
Người login Console → IAM User hoặc IAM Identity Center
App chạy trên AWS → IAM Role
Account khác truy cập → Cross-account Role
CI/CD hiện đại → OIDC + Role
```

---

# 8. IAM Policy: Identity-based vs Resource-based

## Identity-based policy

Gắn vào:

```text
IAM User
IAM Group
IAM Role
```

Ví dụ:

```text
User Alice được phép đọc S3
EC2 Role được phép đọc Secrets Manager
Developer Group được phép describe EC2
```

---

## Resource-based policy

Gắn trực tiếp vào resource.

Ví dụ:

```text
S3 Bucket Policy
KMS Key Policy
SQS Queue Policy
SNS Topic Policy
Secrets Manager Resource Policy
Lambda Resource Policy
```

Ví dụ S3 bucket policy cho account khác đọc object:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOtherAccountReadObjects",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:role/ExternalReadRole"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-shared-bucket/*"
    }
  ]
}
```

---

# 9. Những lỗi IAM phổ biến cần tránh

| Lỗi                                        | Rủi ro                                |
| ------------------------------------------ | ------------------------------------- |
| Dùng root account hằng ngày                | Mất kiểm soát toàn account            |
| Không bật MFA                              | Dễ bị chiếm tài khoản                 |
| Hardcode Access Key                        | Dễ leak qua Git/source                |
| Dùng `AdministratorAccess` quá rộng        | User/app có quyền quá mức             |
| Policy có `Action: "*"` và `Resource: "*"` | Blast radius cực lớn                  |
| Không rotate Access Key                    | Key cũ bị leak vẫn dùng được          |
| Không kiểm tra Access Advisor              | Không biết quyền nào không dùng       |
| Không bật CloudTrail                       | Khó điều tra sự cố                    |
| Trust policy role quá rộng                 | Account/service lạ có thể assume role |
| Không tách dev/stg/prod                    | Dễ thao tác nhầm production           |

---

# 10. Checklist thực hành IAM cho Developer mới học AWS

## Khi tạo user

```text
Không dùng root
Một người = một user riêng
Bật MFA
Không share user
Gán user vào group
Không attach quyền trực tiếp nếu không cần
```

## Khi viết policy

```text
Bắt đầu từ quyền nhỏ nhất
Tránh Action "*"
Tránh Resource "*"
Giới hạn theo bucket/resource cụ thể
Dùng Condition nếu cần
Test bằng IAM Policy Simulator
```

## Khi app cần gọi AWS

```text
Chạy trên EC2 → dùng IAM Role
Chạy trên Lambda → dùng Execution Role
Chạy trên ECS → dùng Task Role
Chạy ngoài AWS → cân nhắc IAM User key hoặc federation/OIDC
Không hardcode Access Key
```

## Khi vận hành production

```text
Bật CloudTrail
Bật IAM Access Analyzer
Review IAM định kỳ
Xóa user/key không dùng
Giới hạn quyền production
Dùng MFA cho thao tác nhạy cảm
```

---

# 11. Tổng kết dễ nhớ

IAM là dịch vụ kiểm soát truy cập trung tâm trong AWS.

Công thức quan trọng nhất:

```text
Principal + Action + Resource + Condition = Access Decision
```

Nói cách khác:

```text
Ai được làm gì, trên tài nguyên nào, trong điều kiện nào.
```

Những điểm phải nhớ:

```text
Root account không dùng hằng ngày
MFA là bắt buộc cho tài khoản quan trọng
Policy là JSON định nghĩa Allow/Deny
Deny luôn thắng Allow
Không có Allow thì mặc định Deny
IAM Role tốt hơn Access Key cho workload trên AWS
Least Privilege là nguyên tắc sống còn
CloudTrail dùng để audit ai đã làm gì
Access Analyzer giúp phát hiện quyền public/cross-account
```

Ví dụ thực tế quan trọng nhất cho developer:

```text
Laravel app chạy trên EC2 cần đọc S3
→ Không hardcode Access Key
→ Tạo IAM Role cho EC2
→ Attach policy chỉ cho s3:GetObject bucket cần thiết
→ Gắn role vào EC2
→ AWS SDK tự lấy temporary credentials
```

Đây là một trong những best practice IAM quan trọng nhất khi bắt đầu làm AWS production.
