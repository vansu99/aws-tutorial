# Storage Types AWS: S3, EBS, EFS

Dễ nhớ nhất:

```text
S3  = Object Storage  → lưu file/object, backup, image, log, static website
EBS = Block Storage   → ổ cứng gắn vào EC2
EFS = File Storage    → shared folder cho nhiều EC2 cùng dùng
```

---

## 1. Amazon S3 là gì?

**S3 giống Google Drive / kho lưu trữ file khổng lồ**, nhưng dùng cho hệ thống/app.

Dùng để lưu:

| Use case | Ví dụ |
|---|---|
| Backup | Backup MySQL, EC2 logs, file export |
| Static files | Image, CSS, JS, video |
| Log storage | ALB logs, CloudTrail logs, app logs |
| Data lake | Lưu dữ liệu lớn để Athena/Glue đọc |
| Archive | Chuyển sang Glacier để lưu lâu dài giá rẻ |

Ví dụ:

```text
App upload ảnh user → S3 Bucket
EC2 backup MySQL    → S3 Bucket
CloudFront          → lấy file từ S3 để serve cho user
```

### Đặc điểm chính của S3

| Điểm | Ý nghĩa |
|---|---|
| Object storage | Lưu dạng object/file |
| Không gắn trực tiếp như ổ đĩa | App truy cập qua API/SDK/CLI |
| Scale rất lớn | Gần như không phải lo tăng dung lượng |
| Durable rất cao | Phù hợp backup/lưu trữ lâu dài |
| Có lifecycle | Tự chuyển file cũ sang IA/Glacier để giảm cost |

### Pricing model của S3

S3 tính phí chủ yếu theo:

```text
Dung lượng lưu trữ
+ số request PUT/GET
+ data transfer ra ngoài Internet
+ retrieval fee nếu dùng IA/Glacier
+ lifecycle/replication nếu bật
```

Các storage class thường gặp:

| S3 Class | Dùng khi nào |
|---|---|
| S3 Standard | File truy cập thường xuyên |
| S3 Standard-IA | File ít truy cập nhưng cần lấy nhanh |
| S3 One Zone-IA | Rẻ hơn, chỉ lưu 1 AZ, chấp nhận rủi ro AZ |
| S3 Intelligent-Tiering | AWS tự tối ưu tier theo tần suất truy cập |
| Glacier Instant Retrieval | Archive nhưng cần lấy rất nhanh |
| Glacier Flexible Retrieval | Archive, lấy chậm hơn |
| Glacier Deep Archive | Lưu rất lâu, rẻ nhất, restore chậm |

---

## 2. Amazon EBS là gì?

**EBS giống ổ cứng SSD/HDD gắn vào EC2.**

Ví dụ EC2 Linux:

```text
/dev/xvda  → root volume
/dev/xvdf  → data volume
```

Dùng cho:

| Use case | Ví dụ |
|---|---|
| OS disk | Ổ root của EC2 |
| App data | `/var/www`, `/data` |
| Database chạy trên EC2 | MySQL/PostgreSQL tự quản lý |
| Persistent storage | Stop EC2 data vẫn còn |
| Snapshot backup | Tạo backup volume |

### Đặc điểm chính của EBS

| Điểm | Ý nghĩa |
|---|---|
| Block storage | Hoạt động như ổ cứng |
| Gắn với EC2 | Thường gắn cho 1 EC2 trong cùng AZ |
| Persistent | EC2 stop/start data vẫn còn |
| Snapshot được | Backup volume sang S3 backend |
| Hiệu năng cao | Phù hợp database/workload cần latency thấp |

### EBS volume types

| Type | Dùng khi nào |
|---|---|
| gp3 | Best practice phổ biến, cân bằng cost/performance |
| gp2 | Đời cũ hơn gp3 |
| io1/io2 | Database cần IOPS cao, critical workload |
| st1 | HDD throughput cao, log/big data tuần tự |
| sc1 | HDD rẻ, dữ liệu ít truy cập |
| Magnetic | Legacy, không nên dùng mới |

Production hiện tại thường nên dùng **gp3** thay vì gp2 vì dễ chỉnh IOPS/throughput riêng và tối ưu cost hơn.

### Pricing model của EBS

EBS tính phí theo:

```text
Dung lượng provisioned GB/tháng
+ IOPS provisioned nếu dùng gp3/io1/io2
+ throughput provisioned nếu có
+ snapshot storage
```

Điểm quan trọng: **EBS tính theo dung lượng bạn provision**, không phải dung lượng thực tế đang dùng trong filesystem.

Ví dụ bạn tạo volume 500 GB nhưng mới dùng 50 GB:

```text
Bạn vẫn trả tiền cho 500 GB
```

---

## 3. Amazon EFS là gì?

**EFS giống shared folder/NFS dùng chung cho nhiều EC2.**

Ví dụ:

```text
EC2 A ┐
EC2 B ├── mount EFS → /mnt/shared
EC2 C ┘
```

Dùng cho:

| Use case | Ví dụ |
|---|---|
| Shared files | Nhiều EC2 cùng đọc/ghi file |
| WordPress shared upload | `/wp-content/uploads` dùng chung |
| Web app nhiều instance | File upload cần chia sẻ |
| Container shared storage | ECS/EKS cần file system chung |
| Linux NFS | App cần POSIX file system |

### Đặc điểm chính của EFS

| Điểm | Ý nghĩa |
|---|---|
| File storage | Dạng thư mục/file như Linux |
| Multi-AZ | Có thể mount từ nhiều AZ |
| Multi-client | Nhiều EC2 cùng truy cập |
| Elastic | Tự tăng/giảm theo dữ liệu thực tế |
| Dễ dùng | Không cần tự quản lý NFS server |

### Pricing model của EFS

EFS tính phí theo:

```text
Dung lượng thực tế lưu trữ
+ throughput mode/performance nếu chọn
+ storage class Standard / IA / Archive
+ data transfer nếu cross-AZ/cross-region tùy case
```

EFS thường **đắt hơn S3 và EBS** nếu lưu nhiều dữ liệu lâu dài, nhưng đổi lại là **nhiều server dùng chung rất tiện**.

---

## 4. So sánh nhanh S3 vs EBS vs EFS

| Tiêu chí | S3 | EBS | EFS |
|---|---|---|---|
| Loại storage | Object | Block | File |
| Hiểu đơn giản | Kho file | Ổ cứng EC2 | Shared folder |
| Gắn vào EC2? | Không mount native như disk | Có | Có, qua NFS |
| Dùng cho nhiều EC2? | Có qua API | Thường không | Có |
| Dữ liệu persistent? | Có | Có | Có |
| Scale dung lượng | Gần như unlimited | Theo volume size | Tự động |
| Latency | Không thấp như disk | Thấp nhất | Thấp, nhưng thường chậm hơn EBS |
| Phù hợp database? | Không dùng làm disk DB | Có | Không nên cho DB chính |
| Phù hợp backup? | Rất phù hợp | Snapshot backup | Có, nhưng không tối ưu bằng S3 |
| Cost | Rẻ nhất cho lưu trữ lớn | Trung bình | Thường cao hơn |

---

## 5. Ví dụ thực tế dễ hiểu

### Case 1: Backup MySQL trên EC2

Nên dùng:

```text
MySQL dump trên EC2
  ↓
Lưu tạm vào EBS
  ↓
Sync lên S3
  ↓
S3 Lifecycle chuyển backup cũ sang Glacier
```

Không nên lưu backup lâu dài chỉ trong EBS vì nếu volume lỗi hoặc bị xóa nhầm thì rủi ro cao hơn. S3 phù hợp hơn cho backup lâu dài.

---

### Case 2: Website WordPress chạy 2 EC2

```text
ALB
 ↓
EC2 WordPress A
EC2 WordPress B
 ↓
RDS MySQL
 ↓
EFS for wp-content/uploads
```

Dùng:

| Thành phần | Storage |
|---|---|
| Code WordPress | AMI/EBS hoặc deploy pipeline |
| Upload image | EFS hoặc S3 |
| Database | RDS |
| Backup | S3 |

Nếu 2 EC2 cùng chạy WordPress mà upload file chỉ nằm local EBS, user upload vào EC2 A thì EC2 B không thấy file. Lúc đó dùng **EFS** hoặc chuyển upload sang **S3**.

---

### Case 3: EC2 chạy app đơn giản

```text
EC2
 ↓
EBS root volume
 ↓
App logs sync lên S3
```

Dùng:

| Nhu cầu | Chọn |
|---|---|
| Ổ hệ điều hành | EBS |
| Log archive | S3 |
| File chia sẻ nhiều server | EFS |

---

## 6. Khi nào chọn cái nào?

### Chọn S3 khi:

```text
Cần lưu file, backup, log, image, video, archive
Không cần mount như ổ cứng
Muốn cost rẻ và durability cao
```

Ví dụ:

```text
backup/mysql/db_20260530.sql.gz → S3
```

---

### Chọn EBS khi:

```text
Cần ổ đĩa cho EC2
Cần latency thấp
Chạy database hoặc app cần filesystem local
```

Ví dụ:

```text
EC2 root volume
MySQL data directory /var/lib/mysql
Application folder /var/www
```

---

### Chọn EFS khi:

```text
Nhiều EC2/ECS/EKS cần dùng chung folder
Cần NFS/shared filesystem
File phải đọc/ghi đồng thời từ nhiều server
```

Ví dụ:

```text
WordPress uploads
Shared media folder
Shared application files
```

---

## 7. Best practice ngắn gọn

| Best practice | Giải thích |
|---|---|
| Backup dài hạn dùng S3 | Rẻ, bền, có lifecycle |
| EC2 disk dùng EBS gp3 | Tối ưu cost/performance |
| Multi-server shared file dùng EFS | Không phải tự dựng NFS |
| Database production nên dùng RDS/Aurora | Đừng tự chạy DB trên EC2 nếu không cần |
| Bật encryption | S3 SSE-KMS/SSE-S3, EBS KMS, EFS KMS |
| Dùng lifecycle | S3/EFS chuyển dữ liệu cũ sang tier rẻ |
| Monitor cost | Storage để lâu rất dễ quên và phát sinh phí |
| Snapshot không thay thế backup strategy đầy đủ | Cần retention, restore test, cross-region nếu quan trọng |

---

## 8. Câu nhớ nhanh nhất

```text
S3  = lưu object/file lâu dài, backup, static content
EBS = ổ cứng riêng cho EC2
EFS = folder dùng chung cho nhiều EC2
```

Với hệ thống production web/app phổ biến:

```text
EC2 chạy app       → EBS
User upload shared → S3 hoặc EFS
Backup/log/archive → S3
Database           → RDS/Aurora, không phải S3/EFS
```
