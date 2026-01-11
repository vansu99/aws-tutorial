# Amazon Elastic Block Store (EBS)

**Amazon Elastic Block Store (EBS)** là dịch vụ lưu trữ dạng khối (*block storage*) được thiết kế để hoạt động cùng với **Amazon EC2**.

EBS cung cấp các ổ đĩa ảo có thể gắn vào (*attach*) các EC2 instance, giúp bạn lưu trữ dữ liệu **liên tục** ngay cả khi instance bị dừng hoặc khởi động lại.

Với khả năng **hiệu suất cao**, **độ bền lớn** và **tính linh hoạt**, EBS được sử dụng phổ biến cho:
- Cơ sở dữ liệu
- Ứng dụng giao dịch
- Các workload yêu cầu độ trễ thấp (low latency)

---

## Đặc điểm chính của AWS EBS

### 1. Block Storage hiệu suất cao
- EC2 truy cập dữ liệu ở **cấp độ block**
- Hoạt động giống ổ SSD vật lý

### 2. Độ bền và sẵn sàng cao
- Dữ liệu được **replicate trong cùng Availability Zone (AZ)**
- Giảm rủi ro mất dữ liệu do lỗi phần cứng

### 3. Tùy chọn hiệu suất linh hoạt
Các loại EBS Volume phổ biến:

- **gp3 / gp2 (General Purpose SSD)**  
  → Cân bằng chi phí và hiệu năng

- **io2 / io1 (Provisioned IOPS SSD)**  
  → Hiệu suất cao, phù hợp cho database

- **st1 / sc1 (HDD)**  
  → Workload truy cập tuần tự, dung lượng lớn

### 4. Snapshot và Backup
- Tạo **EBS Snapshot** để sao lưu toàn bộ volume
- Snapshot được lưu trên **Amazon S3**

### 5. Mã hóa bảo mật (Encryption)
- Tích hợp sẵn với **AWS KMS**
- Mã hóa dữ liệu **at-rest** và **in-transit**

### 6. Tích hợp chặt chẽ với EC2
- Có thể **attach / detach / move** volume linh hoạt
- Không mất dữ liệu khi stop EC2

---

## Lưu ý về Availability Zone (AZ)

- Mỗi **EBS Volume chỉ tồn tại trong 1 AZ**
- Có thể attach vào **bất kỳ EC2 nào trong cùng AZ**
- **Không attach trực tiếp sang AZ khác**

👉 Muốn dùng ở AZ khác:
1. Tạo **EBS Snapshot**
2. Tạo **Volume mới từ Snapshot** tại AZ mong muốn

---

## Gắn EBS cho nhiều EC2 Instance

- Một số loại EBS (io1/io2) hỗ trợ **Multi-Attach**
- Các EC2 phải nằm **cùng AZ**
- Ứng dụng phải hỗ trợ **cluster-aware filesystem**

---

## Cách hoạt động của AWS EBS

EBS cung cấp **block-level storage** cho EC2.  
Khi attach vào instance, EBS hoạt động như ổ đĩa vật lý.

### Quy trình cơ bản
1. Tạo EBS Volume
2. Attach vào EC2 (cùng AZ)
3. Format filesystem (Ext4 / XFS)
4. Mount vào hệ thống
5. Snapshot khi cần backup

---

## AWS EBS có thể làm gì?

- Root volume cho EC2
- Lưu trữ dữ liệu ứng dụng / database
- Snapshot backup & restore
- Chạy filesystem hiệu suất cao
- Persistent storage cho container / Kubernetes

---

## Các trường hợp sử dụng phổ biến

| Use Case | Mô tả |
|--------|------|
| Database Storage | MySQL, PostgreSQL, Oracle, MongoDB |
| Transactional Workloads | I/O cao, độ trễ thấp |
| Boot Volume | Ổ hệ điều hành cho EC2 |
| Big Data Processing | Lưu dữ liệu xử lý song song |
| Backup & DR | Snapshot khôi phục nhanh |

---

## Best Practices AWS EBS

### 1. Chọn đúng loại Volume
- `gp3/gp2` → workload thông thường
- `io2/io1` → database cần IOPS cao
- `st1/sc1` → log, backup, sequential I/O

### 2. Snapshot định kỳ
- Dùng **AWS Backup**
- Đảm bảo khả năng khôi phục

### 3. Bật Encryption
- Dùng **KMS-managed key**
- Bảo vệ dữ liệu nhạy cảm

### 4. Tách volume theo chức năng
- OS
- Data
- Log

### 5. Giám sát hiệu suất
- Amazon CloudWatch:
  - IOPS
  - Throughput
  - Latency

---

## So sánh EBS với các dịch vụ lưu trữ khác

| Dịch vụ | Loại | Đặc điểm | Use Case |
|------|-----|--------|--------|
| Amazon EBS | Block | Hiệu suất cao | Database |
| Amazon EFS | File | Chia sẻ nhiều EC2 | Web, CMS |
| Amazon S3 | Object | Mở rộng lớn | Backup, Data Lake |
| Amazon FSx | Managed FS | Windows / HPC | File server |

---

## Ví dụ thực tế

### Tình huống
Hệ thống **MySQL trên EC2** cần:
- Tốc độ đọc/ghi cao
- Độ ổn định

### Giải pháp
- Dùng **io2 EBS Volume**
- Bật **Encryption (KMS)**
- Snapshot tự động hằng ngày

### Kết quả
- ⚡ Tăng **40% throughput**
- 🔐 Dữ liệu an toàn ngay cả khi EC2 gặp sự cố

---

## Kết luận

**Amazon EBS** là dịch vụ **block storage mạnh mẽ, linh hoạt và đáng tin cậy** cho EC2.

Với khả năng:
- Snapshot
- Encryption
- Tùy chỉnh hiệu suất

👉 **EBS là lựa chọn tối ưu cho workload cần tốc độ cao và khả năng khôi phục mạnh mẽ**.
