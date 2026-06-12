# Hướng dẫn chuyên sâu để làm chủ AWS Secrets Manager

## Ý chính của bài

Bài viết nói về cách quản lý **mật khẩu, API key, database credential** một cách an toàn bằng **AWS Secrets Manager**.

Thay vì hardcode kiểu này trong code:

```env
DB_PASSWORD=my-secret-password
AWS_ACCESS_KEY=xxx
```

ta nên lưu các thông tin nhạy cảm đó trong **Secrets Manager**, rồi application sẽ lấy ra khi cần.

AWS Secrets Manager giúp:

- Lưu secret tập trung.
- Mã hóa secret bằng AWS KMS.
- Kiểm soát ai được đọc secret bằng IAM.
- Tự động xoay vòng mật khẩu.
- Ghi log/audit bằng CloudTrail.
- Tích hợp với Lambda, ECS, RDS, ứng dụng backend.
- Chia sẻ secret cross-account khi cần.

---

## Phần 1: Lưu secret với Customer-Managed KMS Key

### Dịch dễ hiểu

Tác giả không muốn dùng key mặc định của AWS để mã hóa secret, mà muốn tự tạo một **KMS key riêng**.

Quy trình:

1. Vào **AWS KMS**.
2. Tạo một symmetric key mới.
3. Đặt alias là `SecretsManagerKey`.
4. Cấp quyền admin/user cho chính mình.
5. Khi tạo secret trong AWS Secrets Manager, chọn key này thay vì AWS managed key mặc định.

### Giải thích dễ hiểu

Secrets Manager luôn mã hóa secret. Nhưng có 2 kiểu phổ biến:

| Loại key | Ý nghĩa |
|---|---|
| AWS managed key | AWS quản lý key giúp bạn |
| Customer-managed KMS key | Bạn tự quản lý key, policy, audit, rotation |

Dùng **Customer-managed KMS key** giúp kiểm soát chặt hơn:

- Ai được decrypt secret.
- Ai được quản lý key.
- Theo dõi được usage của key qua CloudTrail.
- Có thể dùng key policy riêng.
- Phù hợp production/compliance hơn.

Ví dụ thực tế:

```text
Secret: prod/db/password
KMS Key: alias/SecretsManagerKey
```

Muốn đọc được password thì IAM role không chỉ cần:

```text
secretsmanager:GetSecretValue
```

mà còn cần:

```text
kms:Decrypt
```

với đúng KMS key đó.

---

## Phần 2: Tự build Lambda để rotate secret

### Dịch dễ hiểu

Tác giả tạo một Lambda tên `SecretsManagerRotation` để tự động đổi mật khẩu database theo chu kỳ.

Secrets Manager gọi Lambda theo 4 bước:

| Bước | Ý nghĩa |
|---|---|
| `createSecret` | Tạo password mới và lưu vào version pending |
| `setSecret` | Đổi password thật trong database |
| `testSecret` | Test login database bằng password mới |
| `finishSecret` | Đưa password mới thành current, password cũ bị thay thế |

### Giải thích dễ hiểu

Secret rotation nghĩa là **đổi mật khẩu tự động định kỳ**, ví dụ 30 ngày/lần.

Flow đơn giản:

```text
Secrets Manager
      ↓
Rotation Lambda
      ↓
Generate password mới
      ↓
Update password trong Database
      ↓
Test kết nối DB
      ↓
Promote secret mới thành active
```

Ví dụ:

```text
Ngày 1:
DB password = abc123

Sau 30 ngày:
Secrets Manager tạo password mới = xyz789
Lambda cập nhật password DB thành xyz789
App lấy secret mới từ Secrets Manager
```

Điểm quan trọng: rotation không chỉ đổi secret trong Secrets Manager. Nó phải đổi luôn password thật ở database. Nếu chỉ đổi trong Secrets Manager mà không đổi trong DB thì application sẽ lỗi login.

---

## Phần 3: Kiểm soát quyền bằng tag-based IAM / ABAC

### Dịch dễ hiểu

Tác giả muốn chỉ những application thuộc:

```text
Environment = Production
Department = Engineering
```

mới được đọc production secret.

Cách làm:

1. Gắn tag cho secret:

```text
Environment: Production
Department: Engineering
```

2. Gắn tag tương tự cho IAM role:

```text
Environment: Production
Department: Engineering
```

3. Viết IAM policy cho phép `GetSecretValue` chỉ khi tag của principal khớp với tag của secret.

### Giải thích dễ hiểu

Đây gọi là **ABAC — Attribute-Based Access Control**.

Thay vì cấp quyền theo từng ARN rất dài dòng:

```text
Role A được đọc Secret A
Role B được đọc Secret B
Role C được đọc Secret C
```

ta cấp quyền theo attribute/tag:

```text
Role nào có tag Environment=Production
chỉ được đọc secret có tag Environment=Production
```

Ví dụ:

| IAM Role | Tag | Secret | Tag | Kết quả |
|---|---|---|---|---|
| app-prod-role | Production | prod-db-secret | Production | Được đọc |
| app-dev-role | Development | prod-db-secret | Production | Không được đọc |
| finance-prod-role | Finance | engineering-prod-secret | Engineering | Không được đọc |

Cách này rất mạnh cho công ty có nhiều team, nhiều môi trường dev/staging/prod.

Best practice: production secret nên tách rõ bằng tag, naming convention và IAM condition.

Ví dụ naming:

```text
/prod/payment/db
/staging/payment/db
/dev/payment/db
```

---

## Phần 4: Tích hợp Secrets Manager với Lambda và ECS

### Lambda

#### Dịch dễ hiểu

Trong Lambda, tác giả dùng `boto3` để gọi Secrets Manager và lấy secret. Tên secret được truyền qua environment variable, không hardcode trong code.

Ví dụ ý tưởng:

```text
Lambda Environment Variable:
SECRET_NAME=prod/db/password
```

Lambda code sẽ dùng `SECRET_NAME` để gọi Secrets Manager.

#### Giải thích dễ hiểu

Không nên viết thẳng secret ARN hoặc password trong source code.

Nên làm:

```text
Code → đọc SECRET_NAME từ env → gọi Secrets Manager → lấy password
```

IAM role của Lambda cần tối thiểu:

```text
secretsmanager:GetSecretValue
kms:Decrypt
```

với đúng secret và đúng KMS key.

### ECS

#### Dịch dễ hiểu

Với ECS thì dễ hơn. Trong task definition, chỉ cần khai báo secret ARN và map vào environment variable của container.

Ví dụ:

```text
DB_PASSWORD ← ARN của secret trong Secrets Manager
```

ECS agent sẽ tự lấy secret và inject vào container.

#### Giải thích dễ hiểu

Flow:

```text
ECS Task Definition
      ↓
secrets section
      ↓
Secrets Manager ARN
      ↓
Container nhận DB_PASSWORD
```

Application trong container chỉ cần đọc:

```env
DB_PASSWORD
```

Không cần tự gọi AWS SDK.

Tuy nhiên cần chú ý: **ECS Task Execution Role** phải có quyền đọc secret.

---

## Phần 5: Cross-account access

### Dịch dễ hiểu

Tác giả muốn cho một AWS account khác đọc secret.

Cách làm:

1. Thêm **resource policy** vào secret.
2. Cho phép account đích gọi `secretsmanager:GetSecretValue`.
3. Ở account đích, IAM user/role cũng phải có IAM policy cho phép gọi `GetSecretValue`.

### Giải thích dễ hiểu

Cross-account access cần 2 phía đồng ý.

Ví dụ:

```text
Account A: chứa secret
Account B: muốn đọc secret
```

Cần:

```text
Account A:
Secret resource policy cho phép Account B đọc

Account B:
IAM role/user có policy cho phép gọi GetSecretValue
```

Nếu thiếu 1 trong 2 bên thì vẫn bị deny.

Ngoài ra, nếu secret dùng Customer-managed KMS key, phải cấp thêm quyền trên KMS key cho account/role bên kia:

```text
kms:Decrypt
```

Đây là lỗi rất hay gặp: đã cho `GetSecretValue` nhưng quên `kms:Decrypt`.

---

## Kiến trúc tổng thể dễ hiểu

```text
Application / ECS / Lambda
        ↓
IAM Role
        ↓
Secrets Manager
        ↓
KMS decrypt
        ↓
Secret value
        ↓
Database / API / Third-party service
```

Khi có rotation:

```text
Secrets Manager
        ↓
Rotation Lambda
        ↓
Generate password mới
        ↓
Update database password
        ↓
Test password mới
        ↓
Promote secret mới
```

---

## Kết luận chính của bài

### 1. Không hardcode credentials

Sai lầm lớn nhất là để password/API key trong:

```text
source code
.env commit lên Git
config file trên server
CI/CD variable không kiểm soát
```

Nên dùng Secrets Manager hoặc Parameter Store tùy nhu cầu.

### 2. Dùng KMS key riêng cho production secret

Với môi trường production, nên dùng Customer-managed KMS key để kiểm soát tốt hơn.

Ví dụ:

```text
alias/prod-secrets-key
```

### 3. Rotation giúp giảm rủi ro

Nếu password bị lộ, rotation định kỳ giúp giảm thời gian credential có thể bị lợi dụng.

Nhưng rotation phải được test kỹ, vì nếu Lambda rotation sai logic thì app có thể mất kết nối database.

### 4. IAM phải theo least privilege

Không nên cấp kiểu:

```json
"Resource": "*"
```

Nên giới hạn theo secret ARN, tag, environment, KMS key.

### 5. ECS và Lambda nên dùng IAM Role, không dùng access key

Best practice production:

```text
Lambda Execution Role
ECS Task Role
EC2 Instance Profile
```

Không nên để access key trong source code hoặc `.env`.

---

## Ví dụ thực tế dễ hiểu

Giả sử bạn có Laravel app chạy trên ECS hoặc EC2 cần connect RDS MySQL.

### Cách chưa tốt

```env
DB_HOST=prod-rds.xxxxx.ap-northeast-1.rds.amazonaws.com
DB_USERNAME=admin
DB_PASSWORD=hardcoded_password
```

Rủi ro:

- Dev thấy password.
- Password bị commit lên Git.
- Server bị leak file `.env`.
- Khó rotate password.
- Không audit rõ ai đã đọc password.

### Cách tốt hơn

```text
Secrets Manager:
Secret name: /prod/videoapp/mysql
Value:
{
  "username": "admin",
  "password": "xxxxx",
  "host": "prod-rds.xxxxx.ap-northeast-1.rds.amazonaws.com"
}
```

Application sẽ lấy secret bằng IAM Role.

IAM role chỉ được đọc:

```text
/prod/videoapp/mysql
```

và chỉ được decrypt bằng:

```text
alias/prod-secrets-key
```

---

## Best practice production

Checklist ngắn gọn:

| Hạng mục | Best practice |
|---|---|
| Secret storage | Dùng AWS Secrets Manager |
| Encryption | Dùng Customer-managed KMS key cho production |
| Access control | IAM least privilege |
| Environment separation | Tách dev/staging/prod secret |
| Rotation | Bật rotation cho database credentials nếu phù hợp |
| Audit | Bật CloudTrail |
| App integration | ECS Task Role / Lambda Execution Role |
| Cross-account | Dùng resource policy + KMS key policy |
| Naming | Dùng format rõ ràng như `/prod/app/db` |
| Tagging | Gắn tag `Environment`, `Application`, `Owner`, `Department` |

---

## Tóm tắt cực ngắn

AWS Secrets Manager là nơi lưu password/API key an toàn.  
KMS dùng để mã hóa secret.  
IAM quyết định ai được đọc secret.  
Lambda rotation giúp tự động đổi mật khẩu.  
ECS/Lambda có thể lấy secret mà không cần hardcode.  
Cross-account access cần cả resource policy và IAM policy, nếu dùng KMS riêng thì cần thêm quyền `kms:Decrypt`.

Nói đơn giản:

```text
Đừng để password trong code.
Hãy để password trong Secrets Manager.
App lấy password bằng IAM Role.
Secret được mã hóa bằng KMS.
Mọi truy cập đều có thể audit.
```
