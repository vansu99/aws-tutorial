# Amazon EC2

## Mục tiêu bài học

Sau bài này, bạn sẽ hiểu:

- Amazon EC2 là gì.
- EC2 khác VPS/server truyền thống như thế nào.
- Các thành phần quan trọng khi tạo EC2.
- Cách triển khai một Web Server Apache đơn giản trên Amazon Linux 2023.
- Các best practices thực tế để dùng EC2 an toàn, tiết kiệm và dễ vận hành.

---

# 1. Khái niệm và Phép ẩn dụ

## 1.1. Amazon EC2 là gì?

**Amazon EC2**, viết tắt của **Amazon Elastic Compute Cloud**, là dịch vụ cho phép bạn thuê máy chủ ảo trên AWS.

Nói đơn giản:

> EC2 giống như bạn thuê một máy tính/server cấu hình sẵn trên cloud, có CPU, RAM, ổ cứng, hệ điều hành và mạng Internet. Bạn có thể bật, tắt, nâng cấp, giảm cấu hình hoặc xóa nó khi không cần nữa.

Nếu trước đây bạn muốn chạy một website, bạn có thể phải:

- Mua server vật lý.
- Lắp đặt trong phòng máy.
- Cài hệ điều hành.
- Cấu hình mạng.
- Bảo trì phần cứng.
- Dự phòng khi server hỏng.

Với EC2, AWS lo phần hạ tầng vật lý. Bạn chỉ cần chọn cấu hình và sử dụng.

---

## 1.2. Phép ẩn dụ: EC2 giống như thuê căn hộ

Hãy tưởng tượng bạn muốn có một chỗ ở.

### Mua nhà riêng

Bạn phải:

- Mua đất.
- Xây nhà.
- Bảo trì điện nước.
- Tự sửa khi hỏng.
- Tốn chi phí lớn ban đầu.

Đây giống như việc mua server vật lý.

### Thuê căn hộ

Bạn chỉ cần:

- Chọn diện tích.
- Chọn vị trí.
- Chọn nội thất.
- Trả tiền theo thời gian thuê.
- Không muốn dùng nữa thì trả lại.

Đây giống EC2.

EC2 cho bạn “thuê máy chủ” theo nhu cầu. Hôm nay cần máy nhỏ thì dùng máy nhỏ. Ngày mai traffic tăng thì đổi sang máy lớn hơn hoặc thêm nhiều máy mới.

---

## 1.3. Vì sao gọi là “Elastic”?

**Elastic** nghĩa là co giãn linh hoạt.

EC2 “elastic” vì bạn có thể:

- Tăng cấu hình máy chủ khi hệ thống tải cao.
- Giảm cấu hình khi tải thấp.
- Tạo thêm nhiều instance khi nhiều người truy cập.
- Xóa bớt instance khi không còn cần.
- Chỉ trả tiền cho tài nguyên đang dùng.

Ví dụ:

Một website bán hàng bình thường chỉ cần 1 server. Nhưng vào ngày sale, lượng truy cập tăng gấp 10 lần. Nếu dùng server truyền thống, bạn phải chuẩn bị sẵn máy rất lớn, kể cả khi bình thường không dùng hết.

Với EC2, bạn có thể kết hợp **Auto Scaling** để tự động tăng số lượng server khi traffic tăng và giảm lại khi traffic thấp.

---

## 1.4. Vì sao gọi là “Compute Cloud”?

**Compute** là năng lực xử lý: CPU, RAM, network, chạy ứng dụng, xử lý request, chạy batch job.

**Cloud** là môi trường hạ tầng được cung cấp qua Internet.

Vì vậy, **Elastic Compute Cloud** có thể hiểu là:

> Năng lực tính toán linh hoạt trên cloud.

EC2 là một trong những nền tảng quan trọng nhất của AWS. Rất nhiều dịch vụ hoặc kiến trúc cloud đều xoay quanh EC2, ví dụ:

- Web server.
- API server.
- Worker xử lý queue.
- Batch processing.
- Video encoding.
- Machine learning workload.
- Bastion host.
- Self-hosted database hoặc internal tool.

---

# 2. Các thành phần then chốt của EC2

Khi tạo một EC2 instance, bạn cần hiểu các thành phần sau.

---

## 2.1. Instance Types

**Instance Type** là loại máy chủ EC2 bạn chọn.

Nó quyết định:

- Bao nhiêu CPU.
- Bao nhiêu RAM.
- Hiệu năng network.
- Có GPU hay không.
- Phù hợp workload nào.
- Chi phí mỗi giờ.

Cách đặt tên thường gặp:

```text
m5.2xlarge
```

Trong đó:

```text
m  = instance family / dòng máy
5  = generation / thế hệ
2xlarge = kích thước trong dòng đó
```

---

## 2.2. Các dòng instance phổ biến

| Dòng | Ý nghĩa                       | Phù hợp cho                              |
| ---- | ----------------------------- | ---------------------------------------- |
| T    | Burstable / tiết kiệm chi phí | Web nhỏ, dev/test, server ít tải         |
| M    | General Purpose               | Web app, API, backend thông thường       |
| C    | Compute Optimized             | Xử lý CPU cao, batch job, encoding       |
| R    | Memory Optimized              | Database, cache, xử lý dữ liệu trong RAM |
| G    | GPU Instance                  | AI/ML, render video, xử lý đồ họa        |

---

## 2.3. Nên chọn loại nào?

### Dòng T

Dùng khi:

- Website nhỏ.
- Môi trường development/test.
- Server ít traffic.
- Muốn tiết kiệm chi phí.

Ví dụ:

```text
t3.micro
t3.small
t4g.micro
```

Phù hợp cho người mới học hoặc demo.

---

### Dòng M

Dùng khi:

- Ứng dụng web production vừa và nhỏ.
- API backend.
- Workload cân bằng giữa CPU và RAM.

Ví dụ:

```text
m6i.large
m7i.large
```

Đây là lựa chọn “an toàn” khi chưa biết workload nghiêng về CPU hay RAM.

---

### Dòng C

Dùng khi:

- App cần CPU mạnh.
- Batch processing.
- Convert video.
- High-performance web server.
- Scientific computing.

Ví dụ:

```text
c6i.large
c7i.large
```

Nếu ứng dụng thường xuyên CPU cao, dòng C hợp lý hơn dòng M.

---

### Dòng R

Dùng khi:

- Database.
- Redis/cache.
- In-memory processing.
- Ứng dụng cần nhiều RAM.

Ví dụ:

```text
r6i.large
r7i.large
```

Nếu app bị thiếu RAM, swap nhiều, hoặc database/cache chạy nặng, nên xem xét dòng R.

---

### Dòng G

Dùng khi:

- Machine Learning.
- AI inference/training.
- Render video.
- Xử lý hình ảnh/đồ họa.

Ví dụ:

```text
g4dn.xlarge
g5.xlarge
```

Dòng này thường đắt, chỉ dùng khi thật sự cần GPU.

---

## 2.4. Amazon Machine Image, AMI

**AMI** là “khuôn mẫu” để tạo EC2 instance.

Hãy tưởng tượng bạn cần cài một laptop mới. Bạn có thể:

- Cài Windows/Linux từ đầu.
- Cài driver.
- Cài phần mềm.
- Cấu hình bảo mật.
- Cài monitoring agent.

Nếu làm thủ công nhiều lần sẽ rất mất thời gian.

AMI giúp bạn tạo sẵn một “bản mẫu” gồm:

- Hệ điều hành.
- Phần mềm đã cài.
- Cấu hình hệ thống.
- Agent monitoring.
- Tool cần thiết cho ứng dụng.

Sau đó, bạn có thể tạo nhiều EC2 instance từ AMI đó.

Ví dụ:

```text
Amazon Linux 2023 AMI
Ubuntu Server AMI
Windows Server AMI
Custom AMI của công ty
Marketplace AMI
```

### Khi nào cần custom AMI?

Khi bạn muốn:

- Tạo server nhanh hơn.
- Chuẩn hóa môi trường production.
- Dùng với Auto Scaling.
- Giảm thời gian bootstrap.
- Tránh cài đặt thủ công lặp lại.

Ví dụ thực tế:

Bạn có 10 web server cần cài Nginx, PHP, CloudWatch Agent, CodeDeploy Agent. Thay vì cài từng máy, bạn tạo một AMI chuẩn rồi launch 10 máy từ AMI đó.

---

## 2.5. Security Groups

**Security Group** là tường lửa ảo của EC2.

Nó kiểm soát traffic:

- Đi vào EC2: inbound rules.
- Đi ra từ EC2: outbound rules.

Ví dụ:

Một web server cần cho user truy cập website thì mở:

```text
HTTP  port 80
HTTPS port 443
```

Nếu cần SSH vào Linux server thì mở:

```text
SSH port 22
```

Nhưng không nên mở SSH cho toàn Internet nếu không cần.

---

## 2.6. Security Group hoạt động như thế nào?

Mặc định:

- Inbound traffic bị chặn.
- Outbound traffic được cho phép.
- Security Group chỉ có rule Allow, không có rule Deny.
- Có thể gắn một Security Group cho nhiều EC2.
- Có thể tham chiếu IP hoặc Security Group khác.

Ví dụ Security Group cho web server:

| Type  | Port | Source     | Ý nghĩa                      |
| ----- | ---: | ---------- | ---------------------------- |
| HTTP  |   80 | 0.0.0.0/0  | Cho mọi người truy cập web   |
| HTTPS |  443 | 0.0.0.0/0  | Cho mọi người truy cập HTTPS |
| SSH   |   22 | IP công ty | Chỉ IP công ty được SSH      |

Không nên cấu hình như sau trong production nếu không cần:

```text
SSH 22 0.0.0.0/0
```

Vì điều này nghĩa là ai trên Internet cũng có thể thử kết nối SSH vào server của bạn.

---

## 2.7. Key Pairs

**Key Pair** dùng để đăng nhập bảo mật vào EC2 Linux bằng SSH.

Key Pair gồm:

- Public key: AWS gắn vào EC2.
- Private key: bạn tải về và giữ bí mật.

Khi SSH vào EC2, bạn dùng private key để chứng minh bạn có quyền truy cập.

Ví dụ:

```bash
ssh -i my-key.pem ec2-user@PUBLIC_IP
```

Với Amazon Linux, user mặc định thường là:

```text
ec2-user
```

Với Ubuntu, user mặc định thường là:

```text
ubuntu
```

### Lưu ý bảo mật

Private key giống như chìa khóa nhà. Nếu lộ private key, người khác có thể SSH vào server của bạn nếu Security Group cho phép.

Best practice:

- Không commit file `.pem` lên Git.
- Không gửi key qua chat/email.
- Phân quyền file key chặt chẽ:

```bash
chmod 400 my-key.pem
```

---

## 2.8. EBS, Elastic Block Store

**EBS** là ổ cứng gắn vào EC2.

Nếu EC2 là “máy tính”, thì EBS là “ổ đĩa”.

EBS dùng để lưu:

- Hệ điều hành.
- Source code.
- Log.
- File upload.
- Database data nếu bạn tự cài database trên EC2.

Khi tạo EC2, AWS thường tự tạo một **Root EBS Volume** để chứa hệ điều hành.

Ví dụ:

```text
EC2 instance
 └── Root EBS volume: /dev/xvda, 8GB, gp3
```

Bạn cũng có thể gắn thêm ổ EBS khác:

```text
/data
/backup
/mysql
/logs
```

---

## 2.9. EBS cần nhớ gì?

Một số điểm quan trọng:

- EBS là network drive gắn vào EC2.
- Dữ liệu trên EBS có thể được giữ lại sau khi stop instance.
- EBS thường gắn với một Availability Zone cụ thể.
- Có thể snapshot EBS để backup.
- Root EBS thường bị xóa khi terminate EC2 nếu bật “Delete on termination”.
- Non-root EBS thường không bị xóa mặc định khi terminate.

Ví dụ:

Nếu bạn terminate EC2 nhưng root volume có `Delete on termination = true`, dữ liệu hệ điều hành và dữ liệu nằm trong root volume sẽ mất.

Vì vậy, với server quan trọng, cần kiểm tra:

```text
Delete on termination
Snapshot backup
AMI backup
Data volume riêng
```

---

# 3. Ví dụ thực hành: Triển khai Web Server Apache trên EC2 Amazon Linux 2023

## 3.1. Mục tiêu thực hành

Ta sẽ tạo một EC2 instance chạy **Amazon Linux 2023**, tự động cài **Apache HTTP Server** bằng **User Data**, sau đó truy cập website qua trình duyệt.

Kết quả mong muốn:

```text
http://PUBLIC_IP
```

Hiển thị nội dung web đơn giản.

---

## 3.2. Kiến trúc đơn giản

```text
User Browser
     |
     | HTTP port 80
     v
Security Group
     |
     v
EC2 Amazon Linux 2023
     |
     v
Apache Web Server
```

---

## 3.3. Bước 1: Vào EC2 Console

Trên AWS Console:

```text
EC2 → Instances → Launch instances
```

Đặt tên instance:

```text
demo-apache-webserver
```

---

## 3.4. Bước 2: Chọn AMI

Chọn:

```text
Amazon Linux 2023 AMI
```

Lý do chọn Amazon Linux 2023:

- Do AWS cung cấp.
- Tối ưu cho EC2.
- Phù hợp học AWS.
- Dễ dùng với EC2 Instance Connect.
- Package manager dùng `dnf`.

---

## 3.5. Bước 3: Chọn Instance Type

Với demo, chọn:

```text
t3.micro
```

Hoặc nếu tài khoản hỗ trợ Graviton:

```text
t4g.micro
```

Với môi trường học, nên chọn instance nhỏ để tiết kiệm chi phí.

---

## 3.6. Bước 4: Chọn Key Pair

Nếu cần SSH vào server, tạo hoặc chọn Key Pair.

Ví dụ:

```text
demo-keypair
```

Nếu chỉ dùng User Data và EC2 Instance Connect, bạn vẫn nên hiểu Key Pair dùng để SSH truyền thống.

---

## 3.7. Bước 5: Cấu hình Network và Security Group

Chọn VPC mặc định hoặc VPC bạn đang dùng.

Security Group mở các port:

| Type | Port | Source    | Mục đích                  |
| ---- | ---: | --------- | ------------------------- |
| HTTP |   80 | 0.0.0.0/0 | Cho user truy cập website |
| SSH  |   22 | My IP     | Cho bạn SSH vào EC2       |

Không nên mở SSH từ toàn Internet nếu không cần.

Cấu hình nên là:

```text
HTTP 80  0.0.0.0/0
SSH  22  <IP của bạn>/32
```

Nếu làm demo nhanh, có thể mở HTTP public. Nhưng SSH nên giới hạn theo IP cá nhân.

---

## 3.8. Bước 6: Cấu hình Storage

Với demo:

```text
Root volume: 8GB hoặc 10GB
Volume type: gp3
```

gp3 là lựa chọn phổ biến, cân bằng giữa chi phí và hiệu năng.

---

## 3.9. Bước 7: Thêm User Data

Trong phần **Advanced details → User data**, nhập script sau:

```bash
#!/bin/bash

dnf update -y
dnf install -y httpd

systemctl enable httpd
systemctl start httpd

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>EC2 Apache Demo</title>
</head>
<body>
    <h1>Hello from Amazon EC2!</h1>
    <p>This web server was automatically installed using EC2 User Data.</p>
    <p>OS: Amazon Linux 2023</p>
</body>
</html>
EOF
```

Giải thích nhanh:

```bash
dnf update -y
```

Cập nhật package.

```bash
dnf install -y httpd
```

Cài Apache HTTP Server.

```bash
systemctl enable httpd
```

Cho Apache tự khởi động khi server reboot.

```bash
systemctl start httpd
```

Khởi động Apache ngay lập tức.

```bash
cat <<EOF > /var/www/html/index.html
```

Tạo file HTML demo.

---

## 3.10. Bước 8: Launch Instance

Bấm:

```text
Launch instance
```

Chờ trạng thái:

```text
Instance state: Running
Status checks: 2/2 checks passed
```

---

## 3.11. Bước 9: Truy cập website

Copy **Public IPv4 address** của EC2.

Mở trình duyệt:

```text
http://<PUBLIC_IP>
```

Ví dụ:

```text
http://13.115.xxx.xxx
```

Nếu thấy dòng sau là thành công:

```text
Hello from Amazon EC2!
```

---

## 3.12. Nếu không truy cập được thì kiểm tra gì?

Kiểm tra theo thứ tự:

### 1. Instance đã Running chưa?

```text
EC2 → Instances → Instance state
```

### 2. Security Group đã mở port 80 chưa?

Inbound rule phải có:

```text
HTTP TCP 80 0.0.0.0/0
```

### 3. Apache đã chạy chưa?

SSH vào instance và chạy:

```bash
sudo systemctl status httpd
```

Nếu chưa chạy:

```bash
sudo systemctl start httpd
```

### 4. User Data có chạy không?

Kiểm tra log:

```bash
sudo cat /var/log/cloud-init-output.log
```

### 5. Có dùng HTTPS nhầm không?

Demo này chỉ mở HTTP:

```text
http://PUBLIC_IP
```

Không phải:

```text
https://PUBLIC_IP
```

---

# 4. AWS Best Practices cho EC2

## 4.1. Least Privilege cho Security Groups

Chỉ mở những port thật sự cần.

Không nên:

```text
All traffic 0.0.0.0/0
SSH 22 0.0.0.0/0
RDP 3389 0.0.0.0/0
MySQL 3306 0.0.0.0/0
```

Nên:

```text
HTTP 80 từ Internet nếu là web public
HTTPS 443 từ Internet nếu là web public
SSH 22 chỉ từ IP quản trị
Database chỉ cho phép từ Security Group của application server
```

Ví dụ tốt:

```text
ALB Security Group:
- Allow HTTP/HTTPS from 0.0.0.0/0

EC2 Web Security Group:
- Allow HTTP only from ALB Security Group
- Allow SSH only from Bastion hoặc Session Manager
```

---

## 4.2. Dùng IAM Role thay vì lưu Access Key trên EC2

Không nên lưu access key trong code:

```env
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
```

Vì nếu server bị lộ source code hoặc file `.env`, attacker có thể dùng key để truy cập AWS.

Best practice:

> Gắn IAM Role vào EC2 để ứng dụng tự lấy temporary credentials.

Ví dụ:

EC2 cần upload file lên S3.

Không nên:

```php
// Lưu Access Key trong .env
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

Nên:

```text
EC2 Instance Role
 └── Policy: cho phép s3:PutObject vào bucket cần thiết
```

Ứng dụng Laravel, Node.js, Python SDK có thể tự lấy credentials từ instance metadata.

---

## 4.3. Chọn Pricing Model phù hợp

EC2 có nhiều mô hình tính giá.

### On-Demand

Dùng khi:

- Mới bắt đầu.
- Workload ngắn hạn.
- Dev/test.
- Chưa biết traffic.
- Không muốn cam kết.

Ưu điểm:

- Linh hoạt.
- Không cần trả trước.
- Bật bao lâu trả bấy lâu.

Nhược điểm:

- Giá cao hơn Reserved/Savings Plans.

---

### Reserved Instances

Dùng khi:

- Hệ thống chạy ổn định 24/7.
- Workload ít thay đổi.
- Có thể cam kết 1 năm hoặc 3 năm.

Ví dụ:

```text
Production database server
Internal system chạy quanh năm
```

---

### Savings Plans

Dùng khi:

- Hệ thống chạy lâu dài.
- Muốn tiết kiệm chi phí.
- Cần linh hoạt hơn Reserved Instance.

Ví dụ:

Bạn cam kết dùng khoảng:

```text
$10/hour trong 1 năm
```

AWS giảm giá so với On-Demand cho phần usage tương ứng.

---

### Spot Instances

Dùng khi:

- Job có thể bị gián đoạn.
- Batch processing.
- Render video.
- Data processing.
- Test workload không quan trọng.

Ưu điểm:

- Rẻ hơn rất nhiều so với On-Demand.

Nhược điểm:

- AWS có thể thu hồi instance khi cần capacity.
- Ứng dụng phải chịu được việc bị dừng đột ngột.

---

## 4.4. Luôn Tagging tài nguyên

Tag giúp quản lý chi phí và vận hành.

Ví dụ tag nên có:

```text
Name        = prod-web-01
Environment = production
Project     = video-app
Owner       = backend-team
CostCenter  = media-platform
Backup      = daily
```

Không có tag, khi bill tăng bạn rất khó biết server nào gây chi phí.

Tagging giúp:

- Phân tích chi phí theo project.
- Biết ai quản lý tài nguyên.
- Tự động backup theo tag.
- Tự động shutdown dev server ngoài giờ.
- Dễ audit tài nguyên.

---

## 4.5. Monitoring bằng CloudWatch

Không nên tạo EC2 rồi “để đó”.

Cần theo dõi:

- CPUUtilization.
- NetworkIn / NetworkOut.
- Disk usage.
- Memory usage.
- Status check.
- Application log.
- HTTP 5xx nếu có Load Balancer.

Mặc định CloudWatch có CPU, network, status check. Nhưng memory và disk usage thường cần cài CloudWatch Agent.

Ví dụ alarm nên có:

```text
CPU > 80% trong 5 phút
StatusCheckFailed > 0
Disk usage > 80%
Memory usage > 80%
```

Monitoring giúp bạn phát hiện sớm:

- Server quá tải.
- Ổ cứng đầy.
- Instance lỗi phần cứng.
- App bị memory leak.
- Traffic tăng bất thường.

---

## 4.6. Không đặt database quan trọng trực tiếp trên EC2 nếu không cần

Bạn có thể tự cài MySQL/PostgreSQL trên EC2. Nhưng nếu là production, nên cân nhắc dùng:

```text
Amazon RDS
Amazon Aurora
```

Vì RDS hỗ trợ sẵn:

- Backup.
- Restore.
- Multi-AZ.
- Patch.
- Monitoring.
- Read replica.
- Automated snapshot.

Tự quản database trên EC2 nghĩa là bạn phải tự lo nhiều thứ hơn.

---

## 4.7. Dùng Auto Scaling và Load Balancer cho production

Một EC2 đơn lẻ là điểm lỗi đơn.

Nếu instance chết, website chết.

Kiến trúc tốt hơn:

```text
Users
  |
Route 53
  |
Application Load Balancer
  |
Auto Scaling Group
  |
EC2 instances across multiple AZs
```

Lợi ích:

- Tự thay instance lỗi.
- Tự scale khi traffic tăng.
- Chạy nhiều AZ để tăng availability.
- Không phụ thuộc vào một server duy nhất.

---

## 4.8. Backup bằng AMI hoặc EBS Snapshot

Với EC2 quan trọng, cần backup.

Có hai khái niệm dễ nhầm:

### EBS Snapshot

Backup một volume cụ thể.

Ví dụ:

```text
Backup ổ /data
Backup ổ database
```

### AMI

Backup template của cả EC2, bao gồm root volume và snapshot liên quan.

Dùng khi:

- Muốn restore cả server.
- Muốn clone server.
- Muốn tạo nhiều server giống nhau.
- Muốn dùng với Auto Scaling.

Best practice:

- Tạo backup định kỳ.
- Test restore định kỳ.
- Không chỉ backup mà không kiểm tra khả năng restore.

---

## 4.9. Tắt hoặc xóa tài nguyên không dùng

EC2 có thể phát sinh phí nếu bạn quên.

Cần chú ý:

- Instance đang running.
- EBS volume còn tồn tại.
- Elastic IP không gắn vào instance.
- Snapshot cũ.
- Load Balancer không dùng.
- NAT Gateway không dùng.
- Public IPv4.

Một lỗi phổ biến:

> Stop EC2 rồi nghĩ là hết phí hoàn toàn.

Thực tế:

- Stop EC2: không tính phí compute.
- Nhưng EBS volume vẫn tính phí.
- Elastic IP có thể vẫn tính phí.
- Snapshot vẫn tính phí.

---

# 5. Các Use Case thực tế

## 5.1. Kịch bản 1: Hosting ứng dụng Web có lưu lượng thay đổi

### Bối cảnh

Một công ty thương mại điện tử chạy website bán hàng.

Traffic bình thường:

```text
1,000 users/ngày
```

Ngày sale:

```text
50,000 users/ngày
```

Nếu dùng một server cố định, có hai vấn đề:

- Server nhỏ thì ngày sale bị sập.
- Server lớn thì ngày thường lãng phí.

---

### Kiến trúc đề xuất

```text
Users
  |
Route 53
  |
Application Load Balancer
  |
Auto Scaling Group
  |
EC2 Web Servers across multiple AZs
  |
RDS Database
```

### Cách hoạt động

- User truy cập domain.
- Route 53 trỏ về Application Load Balancer.
- Load Balancer phân phối request đến nhiều EC2.
- Auto Scaling tự thêm EC2 khi CPU/request tăng.
- Khi traffic giảm, Auto Scaling xóa bớt EC2.
- Database dùng RDS để dễ backup và HA.

### Lợi ích

- Chịu tải tốt hơn.
- Không phụ thuộc vào một EC2.
- Tiết kiệm chi phí khi traffic thấp.
- Tự phục hồi khi instance lỗi.

---

## 5.2. Kịch bản 2: Môi trường Development và Test

### Bối cảnh

Team backend cần môi trường để test API.

Yêu cầu:

- Không cần chạy 24/7.
- Chỉ dùng trong giờ làm việc.
- Có thể tạo/xóa nhanh.
- Chi phí thấp.

---

### Kiến trúc đơn giản

```text
Developer
  |
EC2 Dev Server
  |
Docker / Node.js / Laravel / Java App
```

### Cách tối ưu

Dùng instance nhỏ:

```text
t3.micro
t3.small
t4g.micro
```

Dùng On-Demand vì workload ngắn hạn.

Có thể setup lịch tự động stop instance:

```text
Stop lúc 20:00
Start lúc 08:00
```

Tag tài nguyên:

```text
Environment = dev
AutoStop = true
Owner = backend-team
```

### Lợi ích

- Tiết kiệm chi phí.
- Developer có môi trường riêng.
- Dễ reset môi trường.
- Không ảnh hưởng production.

---

## 5.3. Kịch bản 3: Xử lý dữ liệu lớn hoặc render video bằng Spot Instances

### Bối cảnh

Một hệ thống video cần convert hàng ngàn video sang nhiều độ phân giải:

```text
1080p
720p
480p
thumbnail
preview clip
```

Các job này:

- Tốn CPU/GPU.
- Có thể chạy song song.
- Nếu bị gián đoạn có thể chạy lại.
- Không cần server luôn chạy 24/7.

---

### Kiến trúc đề xuất

```text
S3 Upload Bucket
  |
SQS Queue
  |
Auto Scaling Group with Spot Instances
  |
EC2 Workers
  |
S3 Output Bucket
```

### Cách hoạt động

- User upload video lên S3.
- Hệ thống đẩy message vào SQS.
- EC2 worker lấy job từ SQS.
- Worker convert video.
- File sau xử lý được lưu lại vào S3.
- Nếu Spot Instance bị thu hồi, job chưa xong sẽ được xử lý lại.

### Lợi ích

- Tiết kiệm chi phí lớn so với On-Demand.
- Scale được nhiều worker khi queue tăng.
- Phù hợp workload batch.
- Không ảnh hưởng user trực tiếp nếu thiết kế retry tốt.

### Lưu ý

Ứng dụng phải chịu được việc instance bị dừng.

Cần thiết kế:

- Retry job.
- Idempotency.
- Checkpoint nếu xử lý lâu.
- Lưu output vào S3, không lưu trên ổ tạm của instance.
- Theo dõi queue length để scale.

---

# 6. Tóm tắt dễ nhớ

## EC2 là gì?

```text
EC2 = máy chủ ảo thuê theo nhu cầu trên AWS
```

## Khi tạo EC2 cần nhớ

```text
AMI              = khuôn mẫu hệ điều hành/server
Instance Type    = cấu hình CPU/RAM/network
Security Group   = tường lửa ảo
Key Pair         = chìa khóa SSH
EBS              = ổ cứng gắn kèm
User Data        = script chạy khi instance khởi tạo lần đầu
IAM Role         = quyền AWS cho EC2, thay cho Access Key
```

## Quy tắc vàng

```text
Chỉ mở port cần thiết
Không lưu Access Key trên server
Dùng IAM Role cho EC2
Tag đầy đủ
Monitor bằng CloudWatch
Backup bằng Snapshot/AMI
Dùng Auto Scaling cho workload thay đổi
Dùng Spot cho batch job có thể retry
Tắt/xóa tài nguyên không dùng
```

---

# 7. Lộ trình học tiếp theo

Sau khi hiểu EC2, bạn nên học tiếp theo thứ tự:

1. **VPC, Subnet, Route Table, Internet Gateway**
2. **Security Group vs NACL**
3. **EBS, Snapshot, AMI**
4. **Elastic Load Balancer**
5. **Auto Scaling Group**
6. **IAM Role cho EC2**
7. **CloudWatch Monitoring**
8. **Systems Manager Session Manager**
9. **EC2 Pricing Optimization**
10. **Backup & Disaster Recovery cho EC2**

---

# 8. Bài tập thực hành đề xuất

## Bài 1: Tạo Web Server đơn giản

- Launch EC2 Amazon Linux 2023.
- Mở port 80.
- Dùng User Data cài Apache.
- Truy cập bằng Public IP.

## Bài 2: SSH vào EC2

- Tạo Key Pair.
- SSH vào EC2 bằng private key.
- Kiểm tra Apache status.

```bash
sudo systemctl status httpd
```

## Bài 3: Tạo AMI từ EC2

- Cài Apache xong.
- Tạo AMI.
- Launch instance mới từ AMI.
- So sánh thời gian setup với User Data.

## Bài 4: Tạo CloudWatch Alarm

- Tạo alarm CPU > 80%.
- Gửi notification qua SNS/email.
- Test bằng stress tool.

## Bài 5: Tối ưu bảo mật Security Group

- Đóng SSH từ 0.0.0.0/0.
- Chỉ cho phép SSH từ IP cá nhân.
- Nếu dùng ALB, chỉ cho phép EC2 nhận HTTP từ Security Group của ALB.

---

# Kết luận

Amazon EC2 là nền tảng cốt lõi để hiểu cloud computing trên AWS.

Nếu ví AWS như một thành phố cloud, thì EC2 chính là những “căn hộ/máy chủ” mà bạn có thể thuê, cấu hình, mở rộng và trả lại bất cứ lúc nào.

Với lập trình viên, EC2 giúp bạn hiểu rõ cách một ứng dụng thật sự chạy trên hạ tầng cloud:

- Code chạy ở đâu.
- Request đi vào server như thế nào.
- Firewall bảo vệ ra sao.
- Ổ cứng lưu dữ liệu thế nào.
- Server scale khi traffic tăng như thế nào.
- Chi phí phát sinh từ đâu.

Nắm chắc EC2 là bước rất quan trọng trước khi học các kiến trúc nâng cao như Load Balancer, Auto Scaling, Container, Serverless và Microservices trên AWS.
