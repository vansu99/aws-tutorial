# Hướng Dẫn Triển Khai Laravel Trên AWS - Chi Tiết Toàn Diện

## Tác giả: Senior AWS Solutions Architect

## Cập nhật: Tháng 6/2026

---

## 1. KIẾN TRÚC ĐỀ XUẤT

### 1.1 Text Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Cloud (Region: ap-southeast-1)        │
│                                                                  │
│  ┌──────────────┐         ┌──────────────────────────────────┐  │
│  │  Route 53    │         │  VPC: 10.0.0.0/16                │  │
│  │  (DNS)       │────────▶│                                  │  │
│  └──────────────┘         │  ┌────────────────────────────┐  │  │
│                           │  │ Public Subnet: 10.0.1.0/24 │  │  │
│  ┌──────────────┐         │  │                            │  │  │
│  │  Let's       │         │  │  ┌──────────────────────┐  │  │  │
│  │  Encrypt     │         │  │  │  EC2 t3.micro        │  │  │  │
│  │  (HTTPS)     │─────────│──│──│  Amazon Linux 2023   │  │  │  │
│  └──────────────┘         │  │  │                      │  │  │  │
│                           │  │  │  ┌────────────────┐  │  │  │  │
│                           │  │  │  │ Nginx + PHP-FPM│  │  │  │  │
│                           │  │  │  │ Laravel App    │  │  │  │  │
│                           │  │  │  │ MySQL 8.0      │  │  │  │  │
│                           │  │  │  └────────────────┘  │  │  │  │
│                           │  │  │                      │  │  │  │
│                           │  │  │  EBS (gp3, encrypted)│  │  │  │
│                           │  │  └──────────────────────┘  │  │  │
│                           │  │                            │  │  │
│                           │  │  Elastic IP               │  │  │
│                           │  └────────────────────────────┘  │  │
│                           └──────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ S3 Bucket    │  │ CloudWatch   │  │ Systems Manager      │  │
│  │ (Backup)     │  │ (Monitoring) │  │ (Session Manager)    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ CloudTrail   │  │ IAM          │  │ SSM Parameter Store  │  │
│  │ (Audit)      │  │ (Roles)      │  │ (Secrets)            │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Giải Thích Lựa Chọn EC2 + MySQL Trong EC2

| Tiêu chí          | EC2 + MySQL local | RDS MySQL             |
| ----------------- | ----------------- | --------------------- |
| Chi phí/tháng     | ~$0 (MySQL free)  | ~$15-30 (db.t3.micro) |
| Quản lý           | Tự quản lý        | AWS quản lý           |
| Backup            | Tự cấu hình       | Tự động               |
| Performance       | Tốt cho 1-3 user  | Overkill              |
| Network latency   | 0ms (localhost)   | 1-5ms                 |
| High Availability | Không             | Multi-AZ option       |

**Lý do chọn EC2 + MySQL local:**

1. **Chi phí**: Traffic 1-3 user/ngày, RDS db.t3.micro tốn thêm ~$15-30/tháng — không cần thiết
2. **Latency**: MySQL trên localhost = 0ms network latency, tối ưu cho single server
3. **Đơn giản**: Một server duy nhất, dễ quản lý, dễ backup (snapshot toàn bộ)
4. **Phù hợp workload**: Laravel với 1-3 user không cần connection pooling hay read replica

### 1.3 Hạn Chế & Rủi Ro

| Rủi ro                      | Mức độ              | Giải pháp                                  |
| --------------------------- | ------------------- | ------------------------------------------ |
| EC2 down = DB down          | Trung bình          | EBS snapshot hàng ngày, có runbook restore |
| Không có Multi-AZ           | Thấp (traffic thấp) | Chấp nhận downtime ngắn khi restore        |
| Tự quản lý MySQL update     | Thấp                | Cron job check update hàng tuần            |
| RAM chia sẻ giữa app và DB  | Thấp (1-3 user)     | Monitor RAM, swap nếu cần                  |
| Không có automated failover | Thấp                | Manual restore từ snapshot ~10-15 phút     |

**Kết luận**: Với traffic 1-3 user/ngày, rủi ro chấp nhận được. Tiết kiệm ~$15-30/tháng so với RDS.

---

## 2. HƯỚNG DẪN SETUP TỪNG BƯỚC (AWS CONSOLE)

### 2.1 VPC & Networking

#### Bước 1: Tạo VPC

1. Vào **VPC Console** → **Create VPC**
2. Chọn **VPC and more** (tạo luôn subnet, route table, IGW)
3. Cấu hình:
    - Name tag: `laravel-vpc`
    - IPv4 CIDR: `10.0.0.0/16`
    - Number of Availability Zones: `1` (tiết kiệm, traffic thấp)
    - Number of public subnets: `1`
    - Number of private subnets: `0` (không cần vì DB nằm trong EC2)
    - NAT gateways: `None` (tiết kiệm ~$32/tháng)
    - VPC endpoints: `None` (thêm sau nếu cần)
4. Click **Create VPC**

**Giải thích**: Chọn 1 AZ vì traffic thấp, không cần HA ở giai đoạn này. NAT Gateway tốn $32/tháng — không cần khi EC2 ở public subnet.

#### Bước 2: Kiểm tra Route Table

VPC wizard đã tạo sẵn:

- Route table với route `0.0.0.0/0 → Internet Gateway`
- Đã associate với public subnet

Verify: **VPC** → **Route Tables** → chọn route table → tab **Routes**:

```
Destination: 10.0.0.0/16  → Target: local
Destination: 0.0.0.0/0    → Target: igw-xxxxxxx
```

### 2.2 Security Group

#### Bước 3: Tạo Security Group cho EC2

1. Vào **EC2 Console** → **Security Groups** → **Create security group**
2. Cấu hình:
    - Name: `laravel-ec2-sg`
    - Description: `Security group for Laravel EC2 instance`
    - VPC: Chọn `laravel-vpc`

3. **Inbound Rules** (QUAN TRỌNG - tối thiểu quyền):

| Type  | Protocol | Port | Source    | Mục đích                        |
| ----- | -------- | ---- | --------- | ------------------------------- |
| HTTP  | TCP      | 80   | 0.0.0.0/0 | Web traffic (redirect to HTTPS) |
| HTTPS | TCP      | 443  | 0.0.0.0/0 | Web traffic chính               |

> ⚠️ **KHÔNG MỞ SSH PORT 22 ra 0.0.0.0/0!** Dùng Session Manager thay thế.

> Nếu BẮT BUỘC cần SSH (debugging): chỉ mở cho IP cụ thể của bạn, ví dụ `1.2.3.4/32`

4. **Outbound Rules**:

| Type        | Protocol | Port | Destination | Mục đích                                                     |
| ----------- | -------- | ---- | ----------- | ------------------------------------------------------------ |
| All traffic | All      | All  | 0.0.0.0/0   | Allow outbound (cần cho yum update, composer, Let's Encrypt) |

5. Click **Create security group**

### 2.3 IAM Role cho EC2

#### Bước 4: Tạo IAM Role

1. Vào **IAM Console** → **Roles** → **Create role**
2. Trusted entity type: **AWS service**
3. Use case: **EC2**
4. Attach policies:
    - `AmazonSSMManagedInstanceCore` (cho Session Manager)
    - `CloudWatchAgentServerPolicy` (cho CloudWatch monitoring)
5. Tạo thêm **inline policy** cho S3 backup:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["s3:PutObject", "s3:GetObject", "s3:ListBucket"],
            "Resource": [
                "arn:aws:s3:::your-laravel-backup-bucket",
                "arn:aws:s3:::your-laravel-backup-bucket/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:GetParametersByPath"
            ],
            "Resource": "arn:aws:ssm:ap-southeast-1:YOUR_ACCOUNT_ID:parameter/laravel/*"
        }
    ]
}
```

6. Role name: `laravel-ec2-role`
7. Click **Create role**

**Giải thích least privilege**:

- Chỉ allow S3 actions cần thiết cho backup
- Chỉ allow SSM Parameter Store cho path `/laravel/*`
- Không dùng `AdministratorAccess` hay `AmazonS3FullAccess`

### 2.4 Launch EC2 Instance

#### Bước 5: Tạo EC2 Instance

1. Vào **EC2 Console** → **Launch instance**
2. Cấu hình:

**Name and tags:**

- Name: `laravel-production`

**Application and OS Images:**

- Amazon Linux 2023 AMI (HVM, 64-bit x86)
- Architecture: 64-bit (x86)

**Instance type:**

- `t3.micro` (2 vCPU, 1 GiB RAM, Free Tier eligible)

**Key pair:**

- Chọn `Proceed without a key pair` nếu chỉ dùng Session Manager
- HOẶC tạo key pair mới (backup access) — lưu file .pem an toàn

**Network settings:**

- VPC: `laravel-vpc`
- Subnet: public subnet đã tạo
- Auto-assign public IP: **Disable** (sẽ dùng Elastic IP)
- Security group: Chọn `laravel-ec2-sg` đã tạo

**Configure storage:**

- Size: `20 GiB` (đủ cho OS + Laravel + MySQL data nhỏ)
- Volume type: `gp3` (tốt hơn gp2, cùng giá)
- ✅ **Encrypted**: Yes
- KMS key: `aws/ebs` (default, miễn phí)

**Advanced details:**

- IAM instance profile: `laravel-ec2-role`
- Metadata version: **V2 only (token required)** ← Bảo mật hơn
- User data: (để trống, cài manual cho rõ ràng)

3. Click **Launch instance**

#### Bước 6: Allocate & Associate Elastic IP

1. **EC2 Console** → **Elastic IPs** → **Allocate Elastic IP address**
2. Click **Allocate**
3. Chọn Elastic IP vừa tạo → **Actions** → **Associate Elastic IP address**
4. Instance: chọn `laravel-production`
5. Click **Associate**

> ⚠️ **Lưu ý**: Elastic IP miễn phí khi đang associate với running instance. Tính phí $3.6/tháng nếu instance stopped hoặc EIP không associate.

### 2.5 Kết Nối Server Bằng Session Manager

#### Bước 7: Kết nối qua Session Manager

**Prerequisite**: IAM Role đã attach `AmazonSSMManagedInstanceCore` ✓

1. Đợi 2-3 phút sau khi launch để SSM Agent register
2. Vào **Systems Manager** → **Session Manager** → **Start session**
3. Chọn instance `laravel-production`
4. Click **Start session**

Bạn sẽ có shell access mà KHÔNG cần mở port 22.

**Chuyển sang root để cài đặt:**

```bash
sudo su -
```

**Tại sao Session Manager tốt hơn SSH:**

- Không cần mở port 22 trên Security Group
- Không cần quản lý SSH key
- Có audit log tự động trên CloudTrail
- Có thể log toàn bộ session output lên S3/CloudWatch
- Hỗ trợ IAM-based access control

### 2.6 Cài Đặt Software Stack

#### Bước 8: Update OS & Cài đặt packages

```bash
# Update system
sudo dnf update -y

# Cài Nginx
sudo dnf install -y nginx

# Cài PHP 8.2 (hoặc 8.3 tùy Laravel version)
sudo dnf install -y php8.2 php8.2-fpm php8.2-mysqlnd php8.2-mbstring \
  php8.2-xml php8.2-curl php8.2-zip php8.2-bcmath php8.2-intl \
  php8.2-gd php8.2-opcache php8.2-soap php8.2-redis

# Cài MySQL 8.0
sudo dnf install -y mysql-community-server mysql-community-client
# Nếu không có repo, dùng MariaDB thay thế:
# sudo dnf install -y mariadb105-server mariadb105

# Cài các tool cần thiết
sudo dnf install -y git unzip cronie amazon-cloudwatch-agent

# Cài Composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer

# Verify
php -v
nginx -v
mysql --version
composer --version
```

#### Bước 9: Cấu hình MySQL

```bash
# Start MySQL
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Lấy temporary password (MySQL 8.0)
sudo grep 'temporary password' /var/log/mysqld.log

# Secure installation
sudo mysql_secure_installation
# → Đổi root password (dùng password mạnh)
# → Remove anonymous users: Y
# → Disallow root login remotely: Y
# → Remove test database: Y
# → Reload privilege tables: Y

# Đăng nhập MySQL
mysql -u root -p

# Tạo database và user cho Laravel
CREATE DATABASE laravel_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY 'YOUR_STRONG_PASSWORD_HERE';
GRANT ALL PRIVILEGES ON laravel_db.* TO 'laravel_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**QUAN TRỌNG**: MySQL user chỉ có quyền từ `localhost` — không thể connect từ bên ngoài.

#### Bước 10: Cấu hình MySQL chỉ listen localhost

```bash
sudo nano /etc/my.cnf
# Thêm trong [mysqld]:
```

```ini
[mysqld]
bind-address = 127.0.0.1
skip-networking = 0
```

```bash
sudo systemctl restart mysqld
# Verify
sudo ss -tlnp | grep 3306
# Kết quả phải là: 127.0.0.1:3306 (KHÔNG phải 0.0.0.0:3306)
```

#### Bước 11: Deploy Laravel Source Code

```bash
# Tạo thư mục web
sudo mkdir -p /var/www/laravel
sudo chown -R ec2-user:nginx /var/www/laravel

# Clone source code (switch sang ec2-user)
sudo su - ec2-user
cd /var/www/laravel
git clone https://github.com/YOUR_REPO/your-laravel-app.git .

# Hoặc nếu private repo, dùng deploy key hoặc token:
# git clone https://YOUR_TOKEN@github.com/YOUR_REPO/your-laravel-app.git .

# Cài dependencies
composer install --no-dev --optimize-autoloader

# Set permissions
sudo chown -R ec2-user:nginx /var/www/laravel
sudo chmod -R 775 /var/www/laravel/storage
sudo chmod -R 775 /var/www/laravel/bootstrap/cache

# SELinux context (nếu enabled trên Amazon Linux 2023)
sudo chcon -R -t httpd_sys_rw_content_t /var/www/laravel/storage
sudo chcon -R -t httpd_sys_rw_content_t /var/www/laravel/bootstrap/cache
```

#### Bước 12: Cấu hình .env

```bash
cd /var/www/laravel
cp .env.example .env
nano .env
```

```env
APP_NAME="YourApp"
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://yourdomain.com

LOG_CHANNEL=daily
LOG_LEVEL=warning

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_db
DB_USERNAME=laravel_user
DB_PASSWORD=YOUR_STRONG_PASSWORD_HERE

CACHE_DRIVER=file
QUEUE_CONNECTION=database
SESSION_DRIVER=file
SESSION_LIFETIME=120

# Mail (cấu hình SES nếu cần)
MAIL_MAILER=ses
MAIL_FROM_ADDRESS="noreply@yourdomain.com"
```

```bash
# Generate app key
php artisan key:generate

# Run migrations
php artisan migrate --force

# Cache config cho production
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Tạo storage link
php artisan storage:link
```

#### Bước 13: Cấu hình Nginx Virtual Host

```bash
sudo nano /etc/nginx/conf.d/laravel.conf
```

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    root /var/www/laravel/public;

    index index.php;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }

    # Deny access to sensitive files
    location ~* \.(env|log|htaccess)$ {
        deny all;
    }

    # Static file caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    client_max_body_size 20M;
}
```

```bash
# Remove default config
sudo rm -f /etc/nginx/conf.d/default.conf

# Test config
sudo nginx -t

# Start services
sudo systemctl start nginx
sudo systemctl start php-fpm
sudo systemctl enable nginx
sudo systemctl enable php-fpm
```

#### Bước 14: Cấu hình HTTPS với Let's Encrypt

```bash
# Cài certbot
sudo dnf install -y certbot python3-certbot-nginx

# Chạy certbot (domain đã trỏ về Elastic IP)
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Chọn:
# - Nhập email
# - Agree to terms
# - Redirect HTTP to HTTPS: Yes

# Verify auto-renewal
sudo certbot renew --dry-run

# Crontab tự động renew (certbot đã tạo timer, nhưng verify):
sudo systemctl status certbot-renew.timer
# Nếu không có, thêm cron:
echo "0 3 * * * /usr/bin/certbot renew --quiet" | sudo tee -a /var/spool/cron/root
```

#### Bước 15: Trỏ Domain Route 53

1. Vào **Route 53 Console** → **Hosted zones** → Chọn domain
2. **Create record**:
    - Record name: (để trống cho root domain, hoặc `www`)
    - Record type: `A`
    - Value: `YOUR_ELASTIC_IP`
    - TTL: `300`
3. Tạo thêm record cho `www`:
    - Record name: `www`
    - Record type: `CNAME`
    - Value: `yourdomain.com`
    - TTL: `300`

#### Bước 16: Cấu hình Laravel Queue & Scheduler

**Queue Worker (dùng systemd):**

```bash
sudo nano /etc/systemd/system/laravel-worker.service
```

```ini
[Unit]
Description=Laravel Queue Worker
After=network.target

[Service]
User=ec2-user
Group=nginx
Restart=always
RestartSec=3
WorkingDirectory=/var/www/laravel
ExecStart=/usr/bin/php artisan queue:work database --sleep=3 --tries=3 --max-time=3600

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start laravel-worker
sudo systemctl enable laravel-worker
```

**Scheduler (cron):**

```bash
# Thêm crontab cho ec2-user
crontab -e
# Thêm dòng:
* * * * * cd /var/www/laravel && php artisan schedule:run >> /dev/null 2>&1
```

---

## 3. SECURITY HARDENING

### 3.1 Session Manager thay SSH

Đã cấu hình ở trên. Bổ sung logging:

1. **Systems Manager** → **Session Manager** → **Preferences** → **Edit**
2. Enable:
    - ✅ CloudWatch logging → Log group: `/aws/ssm/session-logs`
    - ✅ S3 logging → Bucket: `your-laravel-backup-bucket` prefix `session-logs/`
3. **Save**

### 3.2 Security Group - Đã cấu hình

Tóm tắt:

- ✅ Inbound: Chỉ 80 và 443 từ 0.0.0.0/0
- ✅ KHÔNG mở port 22
- ✅ KHÔNG mở port 3306
- ✅ Outbound: All (cần cho update, composer, certbot)

### 3.3 OS-Level Security

```bash
# Disable root SSH login (nếu có SSH)
sudo sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/^PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Auto security updates
sudo dnf install -y dnf-automatic
sudo nano /etc/dnf/automatic.conf
# Set: apply_updates = yes, upgrade_type = security
sudo systemctl enable --now dnf-automatic-install.timer

# Verify MySQL only on localhost
sudo ss -tlnp | grep 3306
# Phải hiển thị: 127.0.0.1:3306

# PHP-FPM hardening
sudo nano /etc/php-fpm.d/www.conf
# Set: expose_php = Off (trong php.ini)
sudo nano /etc/php.ini
# expose_php = Off
# display_errors = Off
# allow_url_fopen = Off (nếu app không cần)
```

### 3.4 Laravel .env Security

```bash
# Đảm bảo .env không trong Git
cat /var/www/laravel/.gitignore | grep .env
# Phải có: .env

# Permission restrictive
chmod 640 /var/www/laravel/.env
chown ec2-user:nginx /var/www/laravel/.env
```

### 3.5 SSM Parameter Store cho Secrets (Khuyến nghị)

Thay vì hardcode password trong .env, lưu vào SSM Parameter Store:

1. **Systems Manager** → **Parameter Store** → **Create parameter**
2. Tạo các parameter:
    - Name: `/laravel/production/db-password`
    - Type: **SecureString**
    - Value: `YOUR_DB_PASSWORD`

3. Tạo script lấy secrets khi deploy:

```bash
#!/bin/bash
# /opt/scripts/load-env-secrets.sh
DB_PASS=$(aws ssm get-parameter --name "/laravel/production/db-password" \
  --with-decryption --query "Parameter.Value" --output text --region ap-southeast-1)

sed -i "s/DB_PASSWORD=.*/DB_PASSWORD=$DB_PASS/" /var/www/laravel/.env
```

**Chi phí**: SSM Parameter Store Standard tier = **MIỄN PHÍ** (10,000 parameters).
Advanced tier mới tính phí. → Dùng Standard cho case này.

### 3.6 IAM Best Practices

- ✅ MFA cho root account: **IAM** → **Security credentials** → **MFA** → **Assign MFA**
- ✅ Tạo IAM user riêng cho daily use (không dùng root)
- ✅ EC2 dùng IAM Role, không dùng Access Key
- ✅ IAM password policy: min 14 chars, require symbols, rotate 90 days

### 3.7 AWS Security Services

**CloudTrail** (MIỄN PHÍ 1 trail):

1. **CloudTrail** → **Create trail**
2. Name: `laravel-audit-trail`
3. Storage: S3 bucket (tạo mới hoặc dùng backup bucket)
4. Management events: Read + Write
5. ✅ Enable log file validation

**GuardDuty** (~$1-2/tháng cho account nhỏ):

1. **GuardDuty** → **Get started** → **Enable GuardDuty**
2. Tự động detect threats, compromised credentials, crypto mining

**AWS Config** (tùy ngân sách, ~$1-3/tháng):

- Có thể bỏ qua ở giai đoạn 1, bật ở giai đoạn 2

### 3.8 Nginx Security Headers (đã cấu hình ở trên)

Bổ sung Content Security Policy nếu cần:

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

---

## 4. BACKUP & RESTORE

### 4.1 EBS Snapshot với Data Lifecycle Manager

#### Tạo DLM Policy:

1. **EC2 Console** → **Lifecycle Manager** → **Create lifecycle policy**
2. Policy type: **EBS snapshot policy**
3. Target: Tag `Name` = `laravel-production`
4. Schedule:
    - Name: `daily-snapshot`
    - Frequency: **Daily**
    - Starting at: `03:00 UTC` (10:00 AM VN, low traffic)
    - Retention: **7 snapshots** (giữ 7 ngày)
5. ✅ Cross Region copy: Không (tiết kiệm, enable khi cần DR)
6. ✅ Enable policy

**Chi phí**: EBS snapshot = ~$0.05/GB/tháng. 20GB × 7 copies = ~$7/tháng ở mức cao nhất.
Thực tế incremental → ~$1-3/tháng.

### 4.2 MySQL Dump Tự Động

```bash
# Tạo script backup
sudo mkdir -p /opt/scripts
sudo nano /opt/scripts/mysql-backup.sh
```

```bash
#!/bin/bash
# /opt/scripts/mysql-backup.sh

BACKUP_DIR="/tmp/mysql-backups"
BUCKET="your-laravel-backup-bucket"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="laravel_db"
RETENTION_DAYS=14

# Tạo backup directory
mkdir -p $BACKUP_DIR

# Dump database
mysqldump -u laravel_user -p'YOUR_PASSWORD' \
  --single-transaction \
  --routines \
  --triggers \
  $DB_NAME | gzip > $BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz

# Upload to S3
aws s3 cp $BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz \
  s3://$BUCKET/mysql-backups/${DB_NAME}_${DATE}.sql.gz \
  --storage-class STANDARD_IA

# Xóa local file
rm -f $BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz

# Log
echo "$(date): Backup completed - ${DB_NAME}_${DATE}.sql.gz" >> /var/log/mysql-backup.log
```

```bash
# Phân quyền
sudo chmod +x /opt/scripts/mysql-backup.sh

# Cron: chạy 2:00 AM UTC hàng ngày
echo "0 2 * * * /opt/scripts/mysql-backup.sh" | sudo tee -a /var/spool/cron/root
```

### 4.3 S3 Bucket & Lifecycle Policy

#### Tạo S3 Bucket:

1. **S3 Console** → **Create bucket**
2. Name: `your-laravel-backup-bucket` (phải unique globally)
3. Region: Cùng region với EC2
4. ✅ Block all public access: **Yes**
5. ✅ Bucket Versioning: **Enable**
6. ✅ Default encryption: **SSE-S3**
7. Click **Create bucket**

#### Lifecycle Policy:

1. Chọn bucket → **Management** → **Create lifecycle rule**
2. Rule name: `backup-lifecycle`
3. Prefix filter: `mysql-backups/`
4. Transitions:
    - Move to **Standard-IA**: 0 days (upload trực tiếp IA)
    - Move to **Glacier Instant Retrieval**: 30 days
5. Expiration:
    - **Delete current versions**: 90 days
    - **Delete noncurrent versions**: 30 days

### 4.4 Retention Policy Đề Xuất

| Loại backup          | Retention           | Storage      | Chi phí ước tính |
| -------------------- | ------------------- | ------------ | ---------------- |
| EBS Snapshot (daily) | 7 ngày              | EBS Snapshot | ~$1-3/tháng      |
| MySQL dump (daily)   | 14 ngày Standard-IA | S3           | ~$0.1-0.5/tháng  |
| MySQL dump (monthly) | 90 ngày → Glacier   | S3 Glacier   | ~$0.01/tháng     |

### 4.5 Runbook Restore

#### Scenario 1: EC2 Instance Lỗi (không boot được)

```
1. Vào EC2 Console → Stop instance (nếu chưa stop)
2. Detach EBS volume hiện tại
3. Vào Snapshots → Tìm snapshot mới nhất
4. Create Volume từ snapshot (cùng AZ)
5. Attach volume mới vào instance (device: /dev/xvda)
6. Start instance
7. Verify: Session Manager → check services running
8. Thời gian ước tính: 10-15 phút
```

#### Scenario 2: Database Bị Corrupt / Mất Data

```
1. Connect via Session Manager
2. Stop Laravel queue worker:
   sudo systemctl stop laravel-worker
3. Download backup mới nhất từ S3:
   aws s3 cp s3://your-bucket/mysql-backups/LATEST.sql.gz /tmp/
4. Gunzip:
   gunzip /tmp/laravel_db_YYYYMMDD.sql.gz
5. Restore:
   mysql -u root -p laravel_db < /tmp/laravel_db_YYYYMMDD.sql
6. Restart services:
   sudo systemctl start laravel-worker
   php artisan cache:clear
7. Thời gian ước tính: 5-10 phút
```

#### Scenario 3: Toàn bộ EC2 mất (terminated accident)

```
1. Launch EC2 mới (cùng config)
2. Restore EBS từ snapshot mới nhất
3. Attach EBS, assign Elastic IP cũ
4. Start instance
5. Verify tất cả services
6. Thời gian ước tính: 15-25 phút
```

---

## 5. MONITORING & OPERATIONS

### 5.1 CloudWatch Agent Setup

```bash
# CloudWatch Agent đã cài ở trên
# Tạo config file:
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

```json
{
    "agent": {
        "metrics_collection_interval": 300,
        "run_as_user": "root"
    },
    "metrics": {
        "namespace": "Laravel/Production",
        "metrics_collected": {
            "mem": {
                "measurement": ["mem_used_percent"],
                "metrics_collection_interval": 300
            },
            "disk": {
                "measurement": ["disk_used_percent"],
                "metrics_collection_interval": 300,
                "resources": ["/"]
            },
            "cpu": {
                "measurement": ["cpu_usage_active"],
                "metrics_collection_interval": 300
            }
        }
    },
    "logs": {
        "logs_collected": {
            "files": {
                "collect_list": [
                    {
                        "file_path": "/var/www/laravel/storage/logs/laravel*.log",
                        "log_group_name": "/laravel/app",
                        "log_stream_name": "{instance_id}",
                        "retention_in_days": 14
                    },
                    {
                        "file_path": "/var/log/nginx/error.log",
                        "log_group_name": "/laravel/nginx-error",
                        "log_stream_name": "{instance_id}",
                        "retention_in_days": 7
                    },
                    {
                        "file_path": "/var/log/mysqld.log",
                        "log_group_name": "/laravel/mysql",
                        "log_stream_name": "{instance_id}",
                        "retention_in_days": 7
                    }
                ]
            }
        }
    }
}
```

```bash
# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s

sudo systemctl enable amazon-cloudwatch-agent
```

### 5.2 CloudWatch Alarms

Tạo alarms tại **CloudWatch** → **Alarms** → **Create alarm**:

#### Alarm 1: Disk Usage > 80%

- Namespace: `Laravel/Production`
- Metric: `disk_used_percent`
- Condition: Greater than `80`
- Period: 5 minutes
- Action: SNS notification (tạo SNS topic + email subscription)

#### Alarm 2: CPU High > 90% sustained

- Namespace: `AWS/EC2`
- Metric: `CPUUtilization`
- Condition: Greater than `90`
- Period: 5 minutes, 3 consecutive periods
- Action: SNS notification

#### Alarm 3: Status Check Failed

- Namespace: `AWS/EC2`
- Metric: `StatusCheckFailed`
- Condition: Greater than `0`
- Period: 1 minute
- Action: SNS notification + EC2 recover action

**Tạo SNS Topic:**

1. **SNS** → **Create topic** → Type: Standard → Name: `laravel-alerts`
2. **Create subscription** → Protocol: Email → Endpoint: your-email@domain.com
3. Confirm subscription từ email

### 5.3 Log Strategy

| Log          | Location                         | CloudWatch?       | Retention       |
| ------------ | -------------------------------- | ----------------- | --------------- |
| Laravel App  | `/var/www/laravel/storage/logs/` | ✅ 14 ngày        |                 |
| Nginx Error  | `/var/log/nginx/error.log`       | ✅ 7 ngày         |                 |
| Nginx Access | `/var/log/nginx/access.log`      | ❌ (traffic thấp) | Local logrotate |
| MySQL        | `/var/log/mysqld.log`            | ✅ 7 ngày         |                 |
| System       | `/var/log/messages`              | ❌                | Local logrotate |

**Laravel log rotation** (trong `config/logging.php`):

```php
'daily' => [
    'driver' => 'daily',
    'path' => storage_path('logs/laravel.log'),
    'level' => 'warning',  // Production: warning trở lên
    'days' => 14,
],
```

### 5.4 Runbook Xử Lý Sự Cố

#### Sự cố 1: Website trả về 502 Bad Gateway

```
1. Connect Session Manager
2. Check PHP-FPM: sudo systemctl status php-fpm
3. Nếu stopped: sudo systemctl restart php-fpm
4. Check logs: sudo tail -50 /var/log/nginx/error.log
5. Check PHP-FPM logs: sudo tail -50 /var/log/php-fpm/www-error.log
6. Common fix: permission issue → sudo chown -R ec2-user:nginx /var/www/laravel/storage
```

#### Sự cố 2: Disk đầy

```
1. Check disk: df -h
2. Tìm file lớn: sudo du -sh /var/* | sort -rh | head -10
3. Clear Laravel logs: sudo truncate -s 0 /var/www/laravel/storage/logs/*.log
4. Clear old mysql dumps: rm /tmp/mysql-backups/*
5. Check journal: sudo journalctl --vacuum-size=100M
6. Clear yum cache: sudo dnf clean all
```

#### Sự cố 3: MySQL không start

```
1. Check status: sudo systemctl status mysqld
2. Check log: sudo tail -100 /var/log/mysqld.log
3. Check disk space: df -h (InnoDB cần space cho redo log)
4. Nếu corrupted: restore từ backup (xem Section 4.5)
5. Nếu OOM killed: check dmesg, consider tăng swap
```

#### Sự cố 4: Out of Memory

```
1. Check: free -m
2. Check OOM: dmesg | grep -i oom
3. Tạo swap file:
   sudo dd if=/dev/zero of=/swapfile bs=128M count=8  (1GB swap)
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
4. Giảm MySQL buffer: innodb_buffer_pool_size = 128M
5. Giảm PHP-FPM processes: pm.max_children = 3
```

### 5.5 Checklist Sau Deploy

```
□ Website accessible via HTTPS
□ HTTP redirect to HTTPS working
□ Laravel .env APP_DEBUG = false
□ Laravel .env APP_ENV = production
□ php artisan config:cache đã chạy
□ php artisan route:cache đã chạy
□ Storage permission correct (775)
□ Queue worker running: systemctl status laravel-worker
□ Scheduler running: check crontab
□ MySQL only on localhost: ss -tlnp | grep 3306
□ Security Group: no port 22 open
□ Let's Encrypt cert valid: certbot certificates
□ CloudWatch Agent running
□ Backup script tested: chạy thủ công 1 lần
□ Session Manager accessible
□ DNS resolving correctly: nslookup yourdomain.com
```

---

## 6. COST OPTIMIZATION

### 6.1 Ước Tính Chi Phí Hàng Tháng (Region: ap-southeast-1)

| Service             | Cấu hình                         | Chi phí/tháng (USD)       |
| ------------------- | -------------------------------- | ------------------------- |
| EC2 t3.micro        | On-Demand, 24/7                  | ~$10.51                   |
| EBS gp3             | 20 GiB                           | ~$1.60                    |
| Elastic IP          | Associated w/ running instance   | $3.60 (pricing mới 2024+) |
| Route 53            | 1 hosted zone + queries          | ~$0.50 + ~$0.01           |
| S3 (backup)         | ~5GB Standard-IA                 | ~$0.15                    |
| EBS Snapshots       | 20GB × 7 incremental             | ~$1-2                     |
| CloudWatch          | Custom metrics + logs (5GB free) | ~$0-3                     |
| GuardDuty           | Account nhỏ                      | ~$1-2                     |
| CloudTrail          | 1 trail (free)                   | $0                        |
| SSM Parameter Store | Standard (free)                  | $0                        |
| Data Transfer       | < 1GB/tháng (traffic thấp)       | ~$0                       |
| **TỔNG**            |                                  | **~$18-24/tháng**         |

### 6.2 Tối Ưu Cost

#### Đã áp dụng:

- ✅ Không dùng RDS (tiết kiệm $15-30/tháng)
- ✅ Không dùng NAT Gateway (tiết kiệm $32/tháng)
- ✅ Không dùng ALB (tiết kiệm $16+/tháng)
- ✅ gp3 thay vì gp2 (rẻ hơn 20%, performance tốt hơn)
- ✅ SSM Parameter Store Standard (free) thay vì Secrets Manager ($0.40/secret/tháng)
- ✅ CloudWatch log retention ngắn (7-14 ngày)
- ✅ S3 Lifecycle → Glacier cho backup cũ

#### Có thể áp dụng thêm:

- **Reserved Instance** (1 năm, no upfront): EC2 t3.micro ~$6.57/tháng (tiết kiệm 37%)
- **Savings Plan** (1 năm): tương tự RI, linh hoạt hơn
- **Spot Instance**: KHÔNG khuyến nghị cho production web server

#### So sánh với RI:

| Option             | Monthly | Annual  | Savings |
| ------------------ | ------- | ------- | ------- |
| On-Demand          | $10.51  | $126.12 | -       |
| 1yr RI No Upfront  | ~$6.57  | ~$78.84 | 37%     |
| 1yr RI All Upfront | ~$5.84  | ~$70.08 | 44%     |

**Khuyến nghị**: Chạy On-Demand 1-2 tháng đầu, nếu stable thì mua RI 1 năm.

### 6.3 Chi Phí KHÔNG nên cắt giảm

- ❌ Đừng tắt EBS encryption (free, security critical)
- ❌ Đừng tắt CloudTrail (free tier, audit requirement)
- ❌ Đừng bỏ backup (data loss = catastrophic)
- ❌ Đừng bỏ Elastic IP để dùng public IP (IP thay đổi khi reboot = downtime)

---

## 7. ROADMAP CẢI THIỆN

### Giai Đoạn 1: MVP Low-Cost (Hiện tại - $18-24/tháng)

**Đã triển khai:**

- [x] EC2 t3.micro + MySQL local
- [x] HTTPS Let's Encrypt
- [x] Session Manager (no SSH)
- [x] EBS encryption
- [x] Security Group hardened
- [x] Daily backup (EBS snapshot + MySQL dump to S3)
- [x] CloudWatch basic monitoring
- [x] CloudTrail
- [x] Route 53 DNS
- [x] IAM Role least privilege
- [x] SSM Parameter Store cho secrets

**Thời gian**: 1-2 ngày setup

### Giai Đoạn 2: Tăng Bảo Mật & Vận Hành (Khi stable, +$5-10/tháng)

**Nâng cấp:**

- [ ] Enable GuardDuty ($1-2/tháng)
- [ ] Enable AWS Config basic ($1-3/tháng)
- [ ] Mua Reserved Instance (tiết kiệm 37%)
- [ ] Setup CI/CD đơn giản (GitHub Actions → deploy script)
- [ ] Implement blue-green deploy script
- [ ] Thêm swap file 1GB (prevent OOM)
- [ ] Setup log alerting (CloudWatch Logs → SNS)
- [ ] Weekly security scan (AWS Inspector, free tier)
- [ ] Implement proper Laravel health check endpoint
- [ ] Cross-region EBS snapshot (DR)

**Trigger**: Sau 1-2 tháng stable operation

### Giai Đoạn 3: Production Grade (Khi traffic tăng, +$50-100/tháng)

**Khi nào chuyển**: Traffic > 50 user/ngày hoặc cần HA

**Nâng cấp:**

- [ ] Application Load Balancer (ALB) + ACM certificate
- [ ] RDS MySQL (Multi-AZ standby)
- [ ] CloudFront CDN (static assets)
- [ ] WAF (Web Application Firewall)
- [ ] EC2 Auto Scaling Group (min 1, max 3)
- [ ] ElastiCache Redis (session + cache)
- [ ] Separate storage: EFS hoặc S3 cho uploads
- [ ] VPC private subnets cho RDS
- [ ] AWS Secrets Manager
- [ ] CI/CD pipeline đầy đủ (CodePipeline hoặc GitHub Actions)
- [ ] Staging environment

**Kiến trúc Giai đoạn 3:**

```
CloudFront → ALB → EC2 ASG (2 AZ)
                         ↓
                    RDS Multi-AZ
                    ElastiCache
                    S3 (uploads)
```

---

## 8. MASTER CHECKLIST

### Pre-Deploy Checklist

```
□ AWS Account có MFA cho root
□ IAM user riêng cho daily use (không dùng root)
□ Region đã chọn phù hợp (ap-southeast-1 cho VN)
□ Budget alarm đã set ($30/tháng warning)
```

### Infrastructure Checklist

```
□ VPC created với CIDR 10.0.0.0/16
□ Public subnet + Internet Gateway + Route Table
□ Security Group: chỉ port 80, 443 inbound
□ IAM Role: SSM + CloudWatch + S3 limited + SSM Parameter
□ EC2 launched: t3.micro, Amazon Linux 2023, gp3 encrypted
□ Elastic IP associated
□ IMDSv2 only enabled
□ Session Manager functional
```

### Application Checklist

```
□ Nginx installed + configured
□ PHP-FPM installed + hardened
□ MySQL installed + secured + localhost only
□ Composer installed
□ Laravel deployed + dependencies installed
□ .env configured (APP_DEBUG=false, APP_ENV=production)
□ Storage permissions correct
□ Artisan caches generated (config, route, view)
□ Queue worker running as systemd service
□ Scheduler cron configured
□ HTTPS working (Let's Encrypt)
□ HTTP → HTTPS redirect working
```

### Security Checklist

```
□ No SSH port open in Security Group
□ MySQL bind-address = 127.0.0.1
□ Root SSH login disabled
□ PHP expose_php = Off
□ PHP display_errors = Off
□ Nginx server_tokens off
□ .env not in Git
□ Sensitive files denied in Nginx
□ Auto security updates enabled
□ CloudTrail enabled
□ Session Manager logging enabled
□ MFA on root + IAM users
```

### Backup & Monitoring Checklist

```
□ DLM policy: daily EBS snapshot, retain 7
□ MySQL dump cron: daily to S3
□ S3 lifecycle policy configured
□ CloudWatch Agent running (CPU, RAM, Disk)
□ Alarms configured (disk 80%, CPU 90%, status check)
□ SNS topic + email subscription confirmed
□ Restore tested at least once
□ Runbook documented and accessible
```

### Cost Checklist

```
□ AWS Budgets alarm set
□ No unused Elastic IPs
□ No unused EBS volumes
□ S3 lifecycle policy active
□ CloudWatch log retention set (not infinite)
□ Consider RI after 1-2 months stable
```

---

## 9. LỆNH THAM KHẢO NHANH

### Daily Operations

```bash
# Check all services
sudo systemctl status nginx php-fpm mysqld laravel-worker amazon-cloudwatch-agent

# View Laravel logs
tail -f /var/www/laravel/storage/logs/laravel.log

# Clear Laravel cache
cd /var/www/laravel && php artisan cache:clear && php artisan config:cache

# Deploy update
cd /var/www/laravel
git pull origin main
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
sudo systemctl restart php-fpm
sudo systemctl restart laravel-worker

# Check disk usage
df -h

# Check memory
free -m

# Check MySQL connections
mysql -u root -p -e "SHOW PROCESSLIST;"

# Manual backup
/opt/scripts/mysql-backup.sh

# Check SSL cert expiry
sudo certbot certificates
```

### Troubleshooting Commands

```bash
# Check what's listening
sudo ss -tlnp

# Check security group from inside
curl -s http://169.254.169.254/latest/meta-data/security-groups

# Check instance metadata
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/

# Check CloudWatch Agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status

# PHP-FPM status
sudo systemctl status php-fpm
sudo cat /var/log/php-fpm/www-error.log

# Nginx test config
sudo nginx -t

# MySQL check
mysql -u laravel_user -p -e "SELECT 1;"
```

---

## 10. TÓM TẮT QUYẾT ĐỊNH THIẾT KẾ

| Quyết định                               | Lý do                                                 |
| ---------------------------------------- | ----------------------------------------------------- |
| EC2 thay vì ECS/Lambda                   | Đơn giản, chi phí thấp, quen thuộc với Laravel        |
| MySQL local thay vì RDS                  | Tiết kiệm $15-30/tháng, traffic thấp không cần HA     |
| 1 AZ thay vì Multi-AZ                    | Traffic 1-3 user, chấp nhận downtime ngắn khi restore |
| Session Manager thay SSH                 | Bảo mật hơn, audit log, không cần manage key          |
| Let's Encrypt thay ACM                   | Không cần ALB/CloudFront, cert miễn phí               |
| gp3 thay vì gp2                          | Rẻ hơn 20%, IOPS cao hơn baseline                     |
| SSM Parameter Store thay Secrets Manager | Free tier, đủ dùng cho case này                       |
| Nginx thay Apache                        | Performance tốt hơn, RAM ít hơn, phù hợp t3.micro     |
| Standard-IA cho backup S3                | Rẻ hơn Standard 45%, backup ít truy cập               |
| CloudWatch Agent thay Prometheus         | Native AWS, không tốn thêm resource trên t3.micro     |
| File cache/session thay Redis            | Tiết kiệm RAM, traffic thấp không cần Redis           |
| No NAT Gateway                           | EC2 ở public subnet, tiết kiệm $32/tháng              |
| No ALB                                   | Single instance, tiết kiệm $16+/tháng                 |
| IMDSv2 only                              | Prevent SSRF attacks lấy credentials                  |

---

_Document version: 1.0 | Last updated: June 2026_
_Designed for: Laravel app, 1-3 users/day, cost-optimized, security-conscious_
