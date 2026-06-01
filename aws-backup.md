# AWS Backup

Dễ hiểu nhất:

```text
AWS Backup = dịch vụ trung tâm để tự động backup và restore resource AWS.
```

Thay vì từng service tự backup riêng lẻ, ví dụ EC2 snapshot riêng, RDS snapshot riêng, EFS backup riêng, bạn gom về một nơi:

```text
AWS Backup
  ├── EC2 / EBS
  ├── RDS / Aurora
  ├── EFS
  ├── DynamoDB
  ├── S3
  ├── FSx
  └── nhiều service khác
```

---

## 1. AWS Backup dùng để làm gì?

AWS Backup dùng để:

| Mục đích | Giải thích |
|---|---|
| Tự động backup | Lên lịch backup hằng ngày/tuần/tháng |
| Quản lý tập trung | Xem backup nhiều service ở một nơi |
| Retention | Tự xóa backup cũ sau X ngày/tháng |
| Restore | Khôi phục resource khi lỗi |
| Copy cross-region | Copy backup sang region khác để DR |
| Copy cross-account | Copy sang account khác để chống xóa nhầm/ransomware |
| Compliance | Có Backup Audit Manager, Vault Lock |

Ví dụ production:

```text
EC2 Laravel
  ↓
EBS volume backup daily

RDS MySQL
  ↓
Backup daily + retention 35 days

EFS shared upload
  ↓
Backup daily

Backup copy
  ↓
Another AWS account / another region
```

---

## 2. Thành phần chính của AWS Backup

### Backup Plan

**Backup Plan = chính sách backup.**

Nó định nghĩa:

```text
Backup lúc mấy giờ?
Backup mỗi ngày hay mỗi tuần?
Giữ backup bao lâu?
Có chuyển sang cold storage không?
Có copy sang region/account khác không?
```

Ví dụ:

```text
Plan name: prod-daily-backup
Schedule: every day 01:00
Retention: 35 days
Copy to another region: yes
```

Với resource hỗ trợ incremental backup, bản đầu tiên là full backup, các bản sau chỉ backup phần thay đổi, giúp giảm storage cost.

---

### Backup Vault

**Backup Vault = két chứa backup.**

Ví dụ:

```text
Vault: prod-backup-vault
  ├── EC2 recovery point
  ├── RDS recovery point
  └── EFS recovery point
```

---

### Recovery Point

**Recovery Point = một bản backup tại một thời điểm.**

Ví dụ:

```text
RDS backup lúc 2026-06-01 01:00
EC2 EBS snapshot lúc 2026-06-01 01:10
EFS backup lúc 2026-06-01 01:20
```

Khi cần restore, bạn chọn đúng recovery point để khôi phục.

---

### Resource Assignment

**Resource Assignment = chọn resource nào sẽ được backup.**

Có 2 cách phổ biến:

#### Cách 1: Chọn resource trực tiếp

```text
Chọn EC2 instance A
Chọn RDS database B
```

#### Cách 2: Chọn theo tag

Ví dụ tag resource:

```text
Backup = Daily
Environment = Production
```

Backup Plan chọn tất cả resource có tag:

```text
Backup = Daily
```

Đây là cách tốt cho production vì dễ tự động hóa.

---

## 3. AWS Backup hỗ trợ service nào?

Các service hay gặp:

| Service | Backup cái gì? |
|---|---|
| EC2 | AMI/snapshot các EBS volume |
| EBS | Volume snapshot |
| RDS | DB snapshot |
| Aurora | Cluster snapshot |
| EFS | File system backup |
| FSx | File system backup |
| DynamoDB | Table backup |
| S3 | Object backup/version backup tùy cấu hình |
| Storage Gateway | Volume backup |
| VMware/on-prem | Hybrid backup |
| EKS | Cluster state và persistent storage |

Danh sách service hỗ trợ phụ thuộc vào Region và feature availability.

---

## 4. Ví dụ dễ hiểu: backup EC2

Nếu bạn backup EC2 bằng AWS Backup:

```text
EC2 Instance
  ├── Root EBS volume
  └── Data EBS volume
      ↓
AWS Backup tạo recovery point
```

Khi restore, AWS có thể tạo lại:

```text
AMI / EBS volume / instance mới tùy loại restore
```

Lưu ý: AWS Backup không backup “RAM/session đang chạy”. Nó backup storage/resource state.

Với EC2, quan trọng là data phải nằm trên EBS. Nếu dữ liệu nằm trong instance store, dừng/mất instance có thể mất dữ liệu.

---

## 5. Ví dụ: backup RDS

```text
RDS MySQL
  ↓
AWS Backup daily snapshot
  ↓
Retention 35 days
```

Khi restore:

```text
Recovery Point
  ↓
Restore thành RDS instance mới
```

Thông thường restore RDS không ghi đè DB cũ trực tiếp, mà tạo DB mới từ snapshot. Sau đó bạn đổi endpoint/app config nếu cần.

---

## 6. Ví dụ: backup EFS

```text
EFS file system
  ↓
AWS Backup
  ↓
Recovery Point
```

Phù hợp cho:

```text
WordPress uploads
Shared folder
Application shared files
```

Khi restore, có thể restore về file system mới hoặc recovery path tùy service/option.

---

## 7. Pricing model

AWS Backup không tính kiểu “bật service là mất tiền cố định lớn”. Bạn trả chủ yếu theo usage.

Dễ nhớ:

```text
AWS Backup cost =
  dung lượng backup lưu trữ
+ dung lượng restore
+ copy cross-region nếu có
+ restore testing nếu dùng
+ Audit Manager nếu dùng
```

Các cost driver hay gặp:

| Cost driver | Giải thích |
|---|---|
| Backup storage | Bạn lưu càng nhiều backup càng tốn |
| Retention quá dài | Giữ 1 năm sẽ tốn hơn giữ 35 ngày |
| Full backup nhiều | Một số resource không incremental có thể tốn hơn |
| Cross-region copy | Có data transfer + storage ở region đích |
| Cross-account copy | Tăng storage ở account đích |
| Restore | Restore data có thể phát sinh phí |
| Audit Manager | Tính theo control evaluations |
| Warm/cold storage | Cold rẻ hơn nhưng restore có thể chậm/phí khác |

AWS Backup storage pricing thường dựa trên lượng storage mà backup data tiêu thụ, tính theo GB-month trung bình trong tháng.

---

## 8. Warm storage vs Cold storage

Dễ hiểu:

```text
Warm storage = backup cần restore nhanh
Cold storage = backup ít dùng, lưu lâu dài rẻ hơn
```

Ví dụ:

| Loại | Dùng khi nào |
|---|---|
| Warm | Backup 7–35 ngày gần nhất |
| Cold | Backup 3 tháng, 1 năm, 7 năm |

Không phải resource nào cũng hỗ trợ cold storage.

---

## 9. Backup Vault Lock là gì?

**Backup Vault Lock = khóa két backup để chống xóa/sửa retention.**

Dùng cho compliance hoặc chống ransomware.

Có 2 mode thường gặp:

| Mode | Ý nghĩa |
|---|---|
| Governance mode | Người có quyền đặc biệt vẫn có thể thay đổi |
| Compliance mode | Sau grace period, kể cả account owner/AWS cũng không thể xóa/sửa lock cho tới hết retention |

Rất nên dùng cho backup quan trọng, nhưng phải cẩn thận vì lock sai retention có thể gây giữ backup lâu và phát sinh cost.

---

## 10. Backup strategy đề xuất cho production

Ví dụ hệ thống Laravel:

```text
ALB
 ↓
EC2 Laravel trong ASG
 ↓
RDS MySQL
 ↓
S3 uploads
 ↓
EFS nếu có shared folder
```

Đề xuất:

| Resource | Backup strategy |
|---|---|
| EC2 app | Không phụ thuộc backup EC2 nếu app stateless; dùng AMI/Launch Template/IaC |
| EBS data volume | AWS Backup daily, retention 14–35 ngày |
| RDS | Automated backup + AWS Backup nếu cần central governance |
| S3 uploads | Versioning + lifecycle + replication nếu quan trọng |
| EFS | AWS Backup daily |
| Config/IaC | Git repo + Terraform/CDK/CloudFormation |
| Secrets | Secrets Manager replication/versioning strategy |
| Logs | CloudWatch Logs/S3 retention riêng |

Điểm quan trọng: **AWS Backup không thay thế thiết kế HA**.

Backup dùng để khôi phục sau lỗi/xóa nhầm/ransomware. Còn high availability cần:

```text
Multi-AZ
Auto Scaling
RDS Multi-AZ
ALB health check
Monitoring
```

---

## 11. Backup policy mẫu

### Daily backup cho production

```text
Backup frequency: Daily
Backup window: 01:00–03:00
Retention: 35 days
Vault: prod-backup-vault
Resources: tag Backup=Daily
```

### Weekly backup dài hơn

```text
Backup frequency: Weekly
Day: Sunday
Retention: 12 weeks
Resources: tag Backup=Weekly
```

### Monthly backup cho compliance

```text
Backup frequency: Monthly
Retention: 12 months / 7 years tùy yêu cầu
Cold storage: enabled nếu resource hỗ trợ
Vault Lock: cân nhắc dùng
```

---

## 12. Tagging best practice

Nên dùng tag để AWS Backup tự chọn resource.

Ví dụ:

```text
Environment = Production
Backup = Daily
BackupRetention = 35Days
Application = Laravel
Owner = TeamA
```

Resource không cần backup:

```text
Backup = None
```

Cách này giúp tránh quên backup resource mới.

---

## 13. IAM permission cần chú ý

AWS Backup cần IAM role để backup/restore resource.

Khi lỗi kiểu:

```text
AWS Backup does not have permission to describe resource...
```

Thường do:

```text
Service role thiếu permission
Role bị sửa policy
Resource nằm service/region chưa opt-in
SCP/Permission Boundary chặn
KMS key policy không cho AWS Backup dùng
```

Cần kiểm tra:

```text
IAM Role của AWS Backup
AWSBackupServiceRolePolicyForBackup
AWSBackupServiceRolePolicyForRestores
KMS key policy
AWS Organizations SCP
Region/resource opt-in
```

---

## 14. Restore test rất quan trọng

Backup mà chưa test restore thì chưa chắc dùng được.

Nên định kỳ test:

```text
Restore EC2/EBS sang instance test
Restore RDS snapshot thành DB test
Restore EFS sang path/file system test
Kiểm tra app có chạy được không
Ghi lại RTO/RPO thực tế
```

---

## 15. Checklist AWS Backup

```text
[ ] Xác định resource cần backup
[ ] Xác định RPO/RTO
[ ] Tạo Backup Vault
[ ] Bật encryption KMS nếu cần
[ ] Tạo Backup Plan
[ ] Chọn schedule phù hợp
[ ] Set retention hợp lý
[ ] Assign resource bằng tag
[ ] Bật cross-region/account copy nếu cần DR
[ ] Cân nhắc Vault Lock cho backup quan trọng
[ ] Kiểm tra IAM role permission
[ ] Monitor backup job failed
[ ] Test restore định kỳ
[ ] Review cost hằng tháng
```

---

## 16. Câu nhớ nhanh nhất

```text
Backup Plan   = lịch và chính sách backup
Backup Vault  = két chứa backup
Recovery Point= bản backup tại một thời điểm
Retention     = giữ backup bao lâu
Vault Lock    = khóa backup chống xóa/sửa
Restore Test  = kiểm tra backup có khôi phục được không
```

Câu chốt:

**AWS Backup giúp tự động hóa và quản lý tập trung backup cho nhiều AWS service. Production nên dùng backup theo tag, retention rõ ràng, cross-account/cross-region cho dữ liệu quan trọng, bật Vault Lock nếu cần chống xóa/ransomware, và bắt buộc phải test restore định kỳ.**
