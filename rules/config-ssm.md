# Hướng dẫn cấu hình AWS Systems Manager (SSM) để login EC2 qua Session Manager

Tài liệu này hướng dẫn cấu hình **AWS Systems Manager Session Manager** cho phép login (shell) vào EC2 mà **không cần SSH key**, **không cần mở port 22**, **không cần Bastion host**.

Hỗ trợ: **Amazon Linux 2 / 2023**, **CentOS 7**, **Ubuntu 18.04+**.

---

## 1. Mô hình hoạt động

```
[ Admin laptop ] -- HTTPS --> [ AWS Systems Manager ] <-- HTTPS -- [ SSM Agent on EC2 ]
```

- EC2 chỉ cần outbound HTTPS (443) tới các SSM endpoint.
- Không cần Inbound rule SSH (22).
- Audit log đầy đủ qua CloudWatch Logs / S3.

---

## 2. Yêu cầu để EC2 xuất hiện trong SSM (Managed Instance)

Một EC2 muốn login được qua SSM cần đủ **3 điều kiện**:

1. **SSM Agent** đang chạy trên EC2.
2. **IAM Role** với policy `AmazonSSMManagedInstanceCore` được gắn vào EC2.
3. EC2 có **kết nối tới các SSM endpoint** (qua Internet hoặc VPC Endpoint).

---

## 3. Cấu hình IAM Role

### 3.1. Tạo Role

1. IAM Console → **Roles** → **Create role**.
2. **Trusted entity**: AWS service → **EC2**.
3. **Permissions**: chọn các managed policy:
   - `AmazonSSMManagedInstanceCore` (bắt buộc)
   - `CloudWatchAgentServerPolicy` (khuyến nghị nếu dùng kèm CloudWatch Agent)
4. Đặt tên role, ví dụ: `EC2-SSM-Role`.

### 3.2. Gắn Role vào EC2

```
EC2 Console → chọn Instance → Actions → Security → Modify IAM role → chọn EC2-SSM-Role.
```

> Có thể gắn / đổi role mà không cần restart instance, nhưng SSM Agent cần vài phút để nhận credential mới (hoặc restart agent để nhanh hơn).

### 3.3. (Tuỳ chọn) Tạo Instance Profile bằng AWS CLI

```bash
aws iam create-role \
  --role-name EC2-SSM-Role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name EC2-SSM-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name EC2-SSM-Role
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-SSM-Role \
  --role-name EC2-SSM-Role

aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --iam-instance-profile Name=EC2-SSM-Role
```

---

## 4. Cài đặt / Kiểm tra SSM Agent

### 4.1. Amazon Linux 2 / 2023

SSM Agent đã được **cài sẵn**. Chỉ cần kiểm tra trạng thái:

```bash
sudo systemctl status amazon-ssm-agent
```

Nếu chưa chạy:

```bash
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### 4.2. Ubuntu (18.04 / 20.04 / 22.04 / 24.04)

Phương án 1 — qua snap (đã có sẵn trên AMI Ubuntu Server chính chủ):

```bash
sudo snap list amazon-ssm-agent
sudo snap start amazon-ssm-agent
```

Phương án 2 — cài bằng deb:

```bash
ARCH=$(dpkg --print-architecture)   # amd64 hoặc arm64
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_${ARCH}/amazon-ssm-agent.deb
sudo dpkg -i amazon-ssm-agent.deb
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### 4.3. CentOS 7

```bash
sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
sudo systemctl status amazon-ssm-agent
```

> Với ARM64 CentOS / RHEL, dùng URL `linux_arm64/amazon-ssm-agent.rpm`.

### 4.4. Kiểm tra agent đăng ký thành công

Trên EC2:

```bash
sudo systemctl status amazon-ssm-agent
sudo tail -f /var/log/amazon/ssm/amazon-ssm-agent.log
```

Trên AWS Console:

```
Systems Manager → Fleet Manager (hoặc Managed Instances)
```

Instance phải xuất hiện với trạng thái **Online**. Nếu **Connection Lost**, xem mục [Troubleshoot](#7-troubleshoot).

---

## 5. Cấu hình Network

### 5.1. Phương án A — Public Subnet / NAT Gateway

EC2 chỉ cần **outbound HTTPS (443)** tới Internet. Không cần inbound SSH.

Security Group inbound: có thể để **trống** (không mở port nào).

Security Group outbound mặc định (allow all) → ok.

### 5.2. Phương án B — Private Subnet không có Internet (dùng VPC Endpoints)

Tạo các **Interface VPC Endpoint** trong VPC:

| Service Name | Bắt buộc |
|---|---|
| `com.amazonaws.<region>.ssm` | ✅ |
| `com.amazonaws.<region>.ssmmessages` | ✅ |
| `com.amazonaws.<region>.ec2messages` | ✅ |
| `com.amazonaws.<region>.kms` | nếu dùng KMS encrypt session |
| `com.amazonaws.<region>.logs` | nếu log session vào CloudWatch Logs |
| `com.amazonaws.<region>.s3` (Gateway endpoint) | nếu log session vào S3 hoặc patching |

**Lưu ý:** Security Group của các Endpoint phải allow **inbound 443** từ CIDR/SG của EC2.

---

## 6. Login vào EC2 qua SSM

### 6.1. Login qua AWS Console (không cần cài gì)

```
EC2 Console → chọn Instance → Connect → tab "Session Manager" → Connect
```

Hoặc:

```
Systems Manager → Session Manager → Start session → chọn Instance → Start session
```

### 6.2. Login qua AWS CLI (cần plugin)

#### Cài Session Manager Plugin trên máy admin

**macOS:**
```bash
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/mac/sessionmanager-bundle.zip" -o "sessionmanager-bundle.zip"
unzip sessionmanager-bundle.zip
sudo ./sessionmanager-bundle/install -i /usr/local/sessionmanagerplugin -b /usr/local/bin/session-manager-plugin
session-manager-plugin --version
```

**Linux (Ubuntu/Debian):**
```bash
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb
```

**Windows (PowerShell as Admin):**
```powershell
Invoke-WebRequest -Uri "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/windows/SessionManagerPluginSetup.exe" -OutFile "SessionManagerPluginSetup.exe"
Start-Process .\SessionManagerPluginSetup.exe
```

#### Khởi tạo session

```bash
aws ssm start-session \
  --target i-xxxxxxxxxxxxxxxxx \
  --region ap-southeast-1
```

### 6.3. Tunnel SSH / Port forwarding qua SSM

Forward port (ví dụ kéo MySQL 3306 trên EC2 về máy local):

```bash
aws ssm start-session \
  --target i-xxxxxxxxxxxxxxxxx \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3306"],"localPortNumber":["3306"]}' \
  --region ap-southeast-1
```

Forward port tới host khác qua EC2 (ví dụ tới RDS endpoint):

```bash
aws ssm start-session \
  --target i-xxxxxxxxxxxxxxxxx \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host":["mydb.xxxxx.ap-southeast-1.rds.amazonaws.com"],
    "portNumber":["3306"],
    "localPortNumber":["3306"]
  }' \
  --region ap-southeast-1
```

### 6.4. Dùng SSH client qua SSM (giữ nguyên `ssh user@i-xxxx`)

Thêm vào `~/.ssh/config` trên máy admin:

```
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --region ap-southeast-1"
```

Sau đó:

```bash
ssh ec2-user@i-xxxxxxxxxxxxxxxxx
```

> Cách này yêu cầu user (ví dụ `ec2-user`) đã có public key trong `~/.ssh/authorized_keys` của instance, vì SSM chỉ làm transport; xác thực vẫn theo SSH thông thường.

---

## 7. (Khuyến nghị) Bật log Session Manager

### 7.1. Log vào CloudWatch Logs

1. Tạo log group: `/aws/ssm/session-logs`.
2. Mở **Systems Manager → Session Manager → Preferences → Edit**.
3. Tích **CloudWatch logging** → chọn log group ở trên.
4. Có thể bật mã hoá KMS.

### 7.2. Log vào S3

Trong Preferences, tích **S3 logging**, nhập tên bucket.

### 7.3. Cấu hình user mặc định khi vào shell

Trong **Preferences**, có thể đặt:
- **Run As** (chạy với user OS cụ thể).
- **Shell profile**: chèn lệnh khi mở session, ví dụ:
  ```bash
  cd /home/ec2-user
  exec bash -l
  ```

---

## 8. Phân quyền user IAM được phép Login

User / Role IAM cần các permission tối thiểu để mở session:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:DescribeInstanceInformation",
        "ssm:DescribeSessions",
        "ssm:GetConnectionStatus",
        "ssm:StartSession"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:instance/*",
        "arn:aws:ssm:*:*:document/AWS-StartSSHSession",
        "arn:aws:ssm:*:*:document/AWS-StartPortForwardingSession",
        "arn:aws:ssm:*:*:document/AWS-StartPortForwardingSessionToRemoteHost"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:TerminateSession",
        "ssm:ResumeSession"
      ],
      "Resource": "arn:aws:ssm:*:*:session/${aws:username}-*"
    }
  ]
}
```

> Có thể giới hạn `Resource` theo tag (ví dụ `Environment=prod`) bằng `Condition: ssm:resourceTag/...`.

---

## 9. Troubleshoot

### Instance không hiện trong "Managed Instances"

Checklist:
1. SSM Agent đang chạy:
   ```bash
   sudo systemctl status amazon-ssm-agent
   ```
2. IAM Role gắn vào EC2 có `AmazonSSMManagedInstanceCore`:
   ```bash
   curl -s http://169.254.169.254/latest/meta-data/iam/info
   ```
3. Network ra được SSM endpoint:
   ```bash
   curl -v https://ssm.ap-southeast-1.amazonaws.com
   curl -v https://ssmmessages.ap-southeast-1.amazonaws.com
   curl -v https://ec2messages.ap-southeast-1.amazonaws.com
   ```
4. Đồng hồ hệ thống đúng (token AWS yêu cầu lệch < 5 phút):
   ```bash
   timedatectl
   ```

### Agent log

```bash
sudo tail -f /var/log/amazon/ssm/amazon-ssm-agent.log
sudo tail -f /var/log/amazon/ssm/errors.log
```

### Restart agent sau khi đổi IAM Role

```bash
sudo systemctl restart amazon-ssm-agent
```

### Lỗi "TargetNotConnected" khi `start-session`

→ Instance đang **Connection Lost**. Khắc phục theo checklist phía trên.

### Lỗi "AccessDenied" trên máy admin

→ User IAM thiếu quyền `ssm:StartSession`. Cập nhật policy ở mục 8.

---

## 10. Tổng kết các đường dẫn / lệnh quan trọng

| Mục | Giá trị |
|---|---|
| Service name | `amazon-ssm-agent` |
| Log file | `/var/log/amazon/ssm/amazon-ssm-agent.log` |
| Managed policy bắt buộc trên EC2 | `AmazonSSMManagedInstanceCore` |
| Endpoint cần truy cập | `ssm`, `ssmmessages`, `ec2messages` |
| Lệnh login | `aws ssm start-session --target i-xxxx` |
| Console | Systems Manager → Session Manager |
