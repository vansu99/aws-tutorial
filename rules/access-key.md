# Nên dùng IAM Role thay vì Access Key cho workload chạy trên AWS.

Lý do: Access Key là long-term credential, nếu bị lộ thì attacker có thể dùng cho tới khi bạn disable/delete key. Còn IAM Role dùng temporary credential, tự động được cấp và xoay vòng, hết hạn thì không dùng lại được. AWS cũng khuyến nghị dùng IAM roles cho human users và workloads để sử dụng temporary credentials.

```
Access Key = chìa khóa cố định, lộ là nguy hiểm
IAM Role   = thẻ tạm thời, tự hết hạn, an toàn hơn
```

## Cách làm chuẩn theo từng case

### Case 1: EC2 cần access S3

Không nên:

```
EC2 lưu access key trong .env
EC2 dùng key đó để upload backup lên S3
```

Nên làm:

```
EC2
  ↓ Assume Role tự động qua Instance Profile
IAM Role
  ↓
S3 Bucket
```

Ví dụ policy cho EC2 chỉ được upload backup vào đúng bucket/path:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUploadBackupToSpecificPath",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-backup-bucket",
        "arn:aws:s3:::my-backup-bucket/mysql-backup/*"
      ]
    }
  ]
}
```

Best practice hơn nữa: thêm bucket policy chỉ cho role này access.

### Case 2: Lambda cần access DynamoDB/S3/Secrets Manager

Không dùng access key trong Lambda environment variable.

Nên dùng:

```
Lambda Execution Role
  ↓
Permission tới DynamoDB/S3/Secrets Manager
```

Ví dụ:

```
Lambda Role:
  - dynamodb:GetItem
  - dynamodb:PutItem
  - secretsmanager:GetSecretValue
  - logs:CreateLogStream
  - logs:PutLogEvents
```

### Case 3: ECS/EKS container cần access AWS service

Không bake access key vào Docker image.

Nên dùng:

```
ECS Task Role
EKS IRSA / Pod Identity
```

Ví dụ:


```
Container app → Task Role → SQS/S3/DynamoDB
```

### Case 4: User/operator cần CLI access

Không nên tạo IAM User access key lâu dài cho từng người.

```
IAM Identity Center / Federation
  ↓
Assume Role
  ↓
Temporary credential for CLI
```

AWS khuyến nghị human users nên dùng federation với identity provider để access AWS bằng temporary credentials thay vì IAM user với long-term credentials.

## Khi nào Access Key vẫn có thể dùng?

Access Key chỉ nên dùng khi bắt buộc, ví dụ:

| Trường hợp                        | Ghi chú                                        |
| --------------------------------- | ---------------------------------------------- |
| App ngoài AWS cần gọi AWS API     | Ví dụ server on-prem không dùng OIDC/role được |
| Third-party tool chưa hỗ trợ role | Cần giới hạn quyền rất chặt                    |
| Legacy system                     | Cần kế hoạch migration sang role/federation    |
| Emergency/break-glass             | Lưu rất an toàn, audit chặt                    |

Nhưng kể cả phải dùng Access Key thì phải áp dụng:

```
Least privilege
Rotate định kỳ
Không hardcode
Lưu trong Secrets Manager/secure vault
Bật CloudTrail audit
Set alert khi key được dùng bất thường
Xóa key không dùng
```


## Best practice triển khai trong production

### Principle of Least Privilege

Không cấp quyền kiểu:

```
{
  "Action": "*",
  "Resource": "*",
  "Effect": "Allow"
}
```

Nên giới hạn:

```
Chỉ action cần thiết
Chỉ resource cần thiết
Chỉ region/account cần thiết nếu có thể
Có condition nếu phù hợp
```

Ví dụ app chỉ upload backup thì không cần `s3:*`.

Nên cấp:

```
s3:PutObject
s3:GetObject
s3:ListBucket
```

### Không dùng chung một role cho quá nhiều app

Không nên:

```
prod-common-role
  - S3 full access
  - RDS full access
  - SQS full access
  - CloudWatch full access
```

Nên tách role theo workload:

```
prod-web-ec2-role
prod-backup-ec2-role
prod-lambda-report-role
prod-ecs-worker-task-role
```

## Migration plan: chuyển từ Access Key sang IAM Role

### Bước 1: Inventory key hiện tại

Kiểm tra IAM User nào có access key:

```
aws iam list-users
aws iam list-access-keys --user-name USER_NAME
```

Xem lần cuối key được dùng:

```
aws iam get-access-key-last-used --access-key-id AKIAxxxxxxxx
```

### Bước 2: Xác định app/service đang dùng key

### Bước 3: Tạo IAM Role thay thế

Ví dụ EC2 backup MySQL lên S3:

```
Role name: prod-mysql-backup-role
Trusted entity: EC2
Permission: PutObject vào s3://my-backup-bucket/mysql-backup/*
```

### Bước 4: Attach role vào EC2/service

Với EC2:

```
EC2 Console
→ Instance
→ Actions
→ Security
→ Modify IAM role
→ chọn role mới
```

Không cần restart EC2 trong đa số trường hợp.

### Bước 5: Update app bỏ access key

Nếu app dùng AWS SDK/CLI chuẩn, thường chỉ cần xóa env này:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
```

Sau đó SDK/CLI sẽ tự lấy credential từ IAM Role.

```
aws sts get-caller-identity
```

### Bước 6: Disable key cũ trước, chưa delete ngay

Monitor app 24–72 giờ. Nếu không lỗi thì delete.

### Monitoring & alert nên có

Nên bật:

| Service                | Mục đích                              |
| ---------------------- | ------------------------------------- |
| CloudTrail             | Log API call                          |
| GuardDuty              | Detect credential misuse              |
| IAM Access Analyzer    | Phát hiện access cross-account/public |
| AWS Config             | Check IAM/security compliance         |
| CloudWatch/EventBridge | Alert khi có event nhạy cảm           |

Alert nên có:

```
CreateAccessKey
DeleteAccessKey
UpdateAccessKey
AttachUserPolicy
PutUserPolicy
CreateUser
CreateRole
AssumeRole bất thường
ConsoleLogin without MFA
Root account activity
```

***Câu chốt: Access Key chỉ nên dùng khi không còn lựa chọn tốt hơn. Với workload chạy trên AWS, IAM Role gần như luôn là lựa chọn đúng hơn, an toàn hơn và dễ vận hành hơn.***