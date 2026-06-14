# Amazon Route 53

## Mục tiêu bài học

Sau bài này, bạn sẽ hiểu Amazon Route 53 là gì, DNS hoạt động như thế nào, cách trỏ domain về các dịch vụ AWS như Application Load Balancer, CloudFront, S3 Website Endpoint, API Gateway, và cách thiết kế DNS an toàn, ổn định cho môi trường production.

---

# 1. Khái niệm và phép ẩn dụ

## 1.1. Amazon Route 53 là gì?

Amazon Route 53 là dịch vụ DNS được quản lý bởi AWS. Nói dễ hiểu, Route 53 giống như **“danh bạ điện thoại của Internet”**.

Người dùng không muốn nhớ địa chỉ IP như:

```text
13.248.169.48
76.223.54.146
```

Họ chỉ muốn gõ:

```text
example.com
www.example.com
api.example.com
```

Route 53 giúp ánh xạ tên miền tới địa chỉ IP hoặc dịch vụ AWS:

```text
example.com           → Application Load Balancer
www.example.com       → CloudFront
api.example.com       → API Gateway
internal.example.com  → Internal Load Balancer trong VPC
```

Nếu website là một cửa hàng, thì Route 53 giống như bảng chỉ đường giúp khách hàng tìm đúng cửa hàng cần đến.

---

## 1.2. DNS là gì?

**DNS — Domain Name System** là hệ thống dịch tên miền sang địa chỉ mà máy tính hiểu được.

Ví dụ:

```text
Người dùng gõ:
www.example.com

DNS trả về:
203.0.113.10
```

Sau đó trình duyệt mới biết cần gửi request đến server nào.

Không có DNS, người dùng phải nhớ IP của từng website. Điều này giống như bạn phải nhớ số điện thoại của từng người thay vì lưu tên trong danh bạ.

---

## 1.3. Vì sao gọi là Route 53?

Tên **Route 53** có 2 ý nghĩa:

| Thành phần | Ý nghĩa                                               |
| ---------- | ----------------------------------------------------- |
| `Route`    | Điều hướng request của người dùng đến đúng tài nguyên |
| `53`       | DNS mặc định sử dụng port `53`                        |

DNS thường dùng:

```text
UDP/53  → truy vấn DNS thông thường
TCP/53  → truy vấn DNS lớn hơn hoặc một số trường hợp đặc biệt
```

---

## 1.4. Route 53 khác gì với việc chỉ mua domain thông thường?

Các nhà cung cấp domain như GoDaddy, Namecheap, Mắt Bão, PA Việt Nam... có thể bán domain và cung cấp DNS cơ bản. Tuy nhiên, Route 53 mạnh hơn vì tích hợp sâu với AWS.

| Tiêu chí                                 | Nhà cung cấp domain thông thường | Amazon Route 53               |
| ---------------------------------------- | -------------------------------- | ----------------------------- |
| Mua domain                               | Có                               | Có                            |
| Quản lý DNS record                       | Có                               | Có                            |
| Trỏ tới ALB, CloudFront, S3, API Gateway | Làm được nhưng thủ công hơn      | Rất dễ với Alias Record       |
| Health Check                             | Thường hạn chế                   | Có tích hợp sẵn               |
| Failover DNS                             | Hạn chế                          | Có routing policy chuyên dụng |
| Private DNS trong VPC                    | Thường không có                  | Có Private Hosted Zone        |
| IAM phân quyền                           | Không sâu                        | Quản lý chặt bằng IAM         |
| CloudTrail audit                         | Không có hoặc hạn chế            | Có thể theo dõi thay đổi DNS  |

Nói ngắn gọn:

> Nếu chỉ mua domain cá nhân đơn giản, nhà cung cấp domain thông thường là đủ. Nếu chạy hệ thống production trên AWS, Route 53 sẽ an toàn, linh hoạt và chuẩn kiến trúc AWS hơn.

---

# 2. Các thành phần then chốt

## 2.1. Domain Registration

**Domain Registration** là chức năng đăng ký và quản lý tên miền.

Ví dụ bạn có thể mua:

```text
example.com
mycompany.jp
myapp.io
```

ngay trong Route 53.

Sau khi mua domain, AWS có thể tự tạo Hosted Zone tương ứng để bạn quản lý DNS.

---

## 2.2. Hosted Zone

**Hosted Zone** là nơi chứa toàn bộ DNS record của một domain.

Ví dụ domain:

```text
example.com
```

Hosted Zone sẽ quản lý các record như:

```text
example.com
www.example.com
api.example.com
mail.example.com
```

Có 2 loại Hosted Zone quan trọng:

| Loại                | Dùng cho                 | Ví dụ                          |
| ------------------- | ------------------------ | ------------------------------ |
| Public Hosted Zone  | DNS public trên Internet | `example.com`                  |
| Private Hosted Zone | DNS nội bộ trong VPC     | `service.internal`, `db.local` |

Có thể hiểu đơn giản:

```text
Hosted Zone = file cấu hình DNS của domain
DNS Record  = từng dòng cấu hình trong file đó
```

---

## 2.3. DNS Records phổ biến

| Record | Ý nghĩa                                      | Ví dụ                                   |
| ------ | -------------------------------------------- | --------------------------------------- |
| A      | Trỏ domain tới IPv4                          | `example.com → 203.0.113.10`            |
| AAAA   | Trỏ domain tới IPv6                          | `example.com → 2001:db8::1`             |
| CNAME  | Trỏ domain này sang domain khác              | `www.example.com → example.com`         |
| MX     | Mail server nhận email                       | Google Workspace, Microsoft 365         |
| TXT    | Lưu text dùng cho xác minh, SPF, DKIM, DMARC | `"v=spf1 include:_spf.google.com ~all"` |
| NS     | Name Server quản lý domain                   | `ns-123.awsdns-45.com`                  |
| SOA    | Thông tin gốc của DNS zone                   | Serial, admin, refresh time             |

Ví dụ record:

```text
example.com        A       203.0.113.10
www.example.com    CNAME   example.com
example.com        MX      10 mail.example.com
example.com        TXT     "v=spf1 include:_spf.google.com ~all"
```

---

## 2.4. Alias Record

**Alias Record** là record đặc biệt của Route 53 dùng để trỏ domain tới tài nguyên AWS.

Ví dụ:

```text
example.com → Application Load Balancer
example.com → CloudFront Distribution
example.com → S3 Static Website Endpoint
example.com → API Gateway
```

So sánh Alias và CNAME:

| Alias Record                                             | CNAME                           |
| -------------------------------------------------------- | ------------------------------- |
| Dùng được cho root domain `example.com`                  | Không dùng được cho root domain |
| Tích hợp tốt với AWS service                             | Không nhận biết AWS resource    |
| Không cần tự quản lý IP thay đổi phía AWS                | Phụ thuộc hostname              |
| Có thể dùng với ALB, CloudFront, S3 Website, API Gateway | Dùng chung cho mọi hostname     |

Ví dụ nên dùng:

```text
example.com      A Alias    my-alb-123.ap-northeast-1.elb.amazonaws.com
www.example.com  A Alias    my-alb-123.ap-northeast-1.elb.amazonaws.com
```

Không nên dùng CNAME cho root domain:

```text
example.com CNAME my-alb-123.ap-northeast-1.elb.amazonaws.com
```

Vì CNAME không dùng được ở zone apex/root domain.

---

## 2.5. TTL — Time To Live

**TTL** là thời gian DNS resolver được phép cache kết quả DNS.

Ví dụ:

```text
example.com A 203.0.113.10 TTL 300
```

Nghĩa là DNS resolver có thể cache IP này trong **300 giây**.

| TTL                     | Ưu điểm                 | Nhược điểm                      |
| ----------------------- | ----------------------- | ------------------------------- |
| Cao, ví dụ 86400 giây   | Ít query DNS hơn        | Khi đổi record sẽ cập nhật chậm |
| Thấp, ví dụ 60–300 giây | Dễ migration, đổi nhanh | Nhiều query DNS hơn             |

Khi chuẩn bị migration, nên giảm TTL trước vài giờ hoặc vài ngày.

Ví dụ:

```text
Trước migration:
TTL = 86400

Trước ngày migration:
TTL = 300 hoặc 60

Sau migration ổn định:
TTL = 300, 600, 3600 tùy hệ thống
```

---

## 2.6. Routing Policies

Routing Policy quyết định Route 53 trả lời DNS query như thế nào.

| Routing Policy    | Dùng khi nào                             | Ví dụ                                 |
| ----------------- | ---------------------------------------- | ------------------------------------- |
| Simple            | Một record đơn giản                      | `example.com → ALB`                   |
| Weighted          | Chia traffic theo tỷ lệ                  | 90% production, 10% canary            |
| Latency-based     | Gửi user tới region có latency thấp nhất | User Nhật → Tokyo, user Mỹ → Virginia |
| Failover          | Primary lỗi thì chuyển sang backup       | ALB chính lỗi → S3 maintenance page   |
| Geolocation       | Route theo vị trí địa lý của user        | User VN → site tiếng Việt             |
| Geoproximity      | Route theo khoảng cách địa lý, có bias   | Ưu tiên region gần user hơn           |
| Multivalue Answer | Trả về nhiều IP healthy                  | DNS dạng đơn giản có health check     |

Ví dụ Weighted Routing:

```text
api.example.com → ALB-v1 weight 90
api.example.com → ALB-v2 weight 10
```

Dùng cho canary release: chỉ cho 10% user vào version mới.

---

## 2.7. Health Checks

**Health Check** giúp Route 53 kiểm tra endpoint có còn hoạt động không.

Ví dụ kiểm tra:

```text
https://example.com/health
```

Nếu endpoint trả về `200 OK`, hệ thống được coi là healthy. Nếu timeout hoặc trả về lỗi, endpoint bị coi là unhealthy.

Kết hợp với Failover Routing:

```text
Primary: ALB production
Backup : S3 static maintenance page
```

Khi ALB lỗi, Route 53 có thể trả về backup endpoint.

---

## 2.8. Private Hosted Zone

**Private Hosted Zone** dùng cho DNS nội bộ trong VPC.

Ví dụ trong VPC:

```text
api.internal.company.local → 10.0.1.25
db.internal.company.local  → mydb.cluster-xxxxx.ap-northeast-1.rds.amazonaws.com
```

Use case thực tế:

```text
service-a gọi service-b bằng:
http://service-b.internal
```

Thay vì hardcode IP:

```text
http://10.0.2.15
```

Điều này rất hữu ích cho microservices, ECS, EC2 nội bộ, RDS, internal ALB.

---

## 2.9. DNSSEC

**DNSSEC — Domain Name System Security Extensions** giúp bảo vệ DNS khỏi việc bị giả mạo dữ liệu DNS.

Nói dễ hiểu:

```text
DNS thường:
Tôi hỏi example.com là IP nào?
Ai đó có thể giả mạo câu trả lời nếu hệ thống bị tấn công.

DNSSEC:
Câu trả lời DNS được ký số.
Resolver có thể kiểm tra câu trả lời có đúng từ chủ domain không.
```

DNSSEC giúp giảm rủi ro DNS spoofing và cache poisoning.

---

# 3. Ví dụ thực hành: Trỏ `example.com` về Application Load Balancer

## 3.1. Kiến trúc mục tiêu

```text
User
  ↓
example.com
  ↓
Route 53
  ↓
Application Load Balancer
  ↓
EC2 instances
```

Giả sử bạn có:

```text
Domain: example.com
ALB DNS Name:
my-app-alb-123456.ap-northeast-1.elb.amazonaws.com
```

---

## 3.2. Bước 1 — Tạo Hosted Zone

Vào AWS Console:

```text
Route 53
→ Hosted zones
→ Create hosted zone
```

Cấu hình:

```text
Domain name: example.com
Type: Public hosted zone
```

Sau khi tạo, Route 53 sẽ sinh ra các NS record dạng:

```text
ns-123.awsdns-45.com
ns-678.awsdns-90.net
ns-111.awsdns-22.org
ns-333.awsdns-44.co.uk
```

---

## 3.3. Bước 2 — Cập nhật Name Server nếu domain không mua ở AWS

Nếu domain mua ở nhà cung cấp khác, ví dụ Namecheap, GoDaddy, Mắt Bão..., bạn cần vào trang quản lý domain và thay Name Server sang NS của Route 53.

Ví dụ:

```text
Old Name Servers:
ns1.old-provider.com
ns2.old-provider.com

New Name Servers:
ns-123.awsdns-45.com
ns-678.awsdns-90.net
ns-111.awsdns-22.org
ns-333.awsdns-44.co.uk
```

Sau bước này, Route 53 mới trở thành nơi quản lý DNS chính thức cho domain.

---

## 3.4. Bước 3 — Tạo A Record dạng Alias cho root domain

Trong Hosted Zone `example.com`, tạo record:

```text
Record name: example.com
Record type: A
Alias: Yes
Route traffic to: Application and Classic Load Balancer
Region: ap-northeast-1
Load balancer: my-app-alb-123456.ap-northeast-1.elb.amazonaws.com
Routing policy: Simple
Evaluate target health: Yes hoặc No tùy thiết kế
```

Record mẫu:

```text
example.com.    A    Alias    my-app-alb-123456.ap-northeast-1.elb.amazonaws.com
```

---

## 3.5. Bước 4 — Tạo record cho `www.example.com`

Có 2 cách phổ biến.

### Cách 1: Dùng Alias trỏ thẳng tới ALB

```text
www.example.com.    A    Alias    my-app-alb-123456.ap-northeast-1.elb.amazonaws.com
```

### Cách 2: Dùng CNAME trỏ về root domain

```text
www.example.com.    CNAME    example.com
```

Trong AWS, nếu trỏ tới ALB hoặc CloudFront, nên dùng **Alias** cho đồng bộ và chuẩn AWS hơn.

---

## 3.6. Bước 5 — Kiểm tra DNS

Dùng `nslookup`:

```bash
nslookup example.com
nslookup www.example.com
```

Dùng `dig`:

```bash
dig example.com
dig www.example.com
```

Kiểm tra Name Server:

```bash
dig NS example.com
```

Kiểm tra trace DNS:

```bash
dig example.com +trace
```

Kiểm tra HTTP:

```bash
curl -I https://example.com
curl -I https://www.example.com
```

Kết quả mong muốn:

```text
HTTP/2 200
server: awselb/2.0
```

hoặc response từ application của bạn.

---

## 3.7. Khi DNS chưa cập nhật thì xử lý thế nào?

DNS có cache ở nhiều tầng:

```text
Browser cache
OS cache
Router cache
ISP DNS cache
Public resolver cache, ví dụ Google DNS, Cloudflare DNS
```

Nếu vừa thay record mà chưa thấy cập nhật, kiểm tra theo thứ tự:

```bash
dig example.com
dig example.com @8.8.8.8
dig example.com @1.1.1.1
dig NS example.com
```

Trên Windows có thể flush DNS:

```bash
ipconfig /flushdns
```

Trên macOS:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Nếu record TTL đang là 3600, có thể phải chờ khoảng 1 giờ. Nếu TTL là 86400, có thể phải chờ lâu hơn.

---

# 4. AWS Best Practices

## 4.1. Dùng Alias Record cho AWS services

Khi trỏ domain tới ALB, CloudFront, S3 Website Endpoint, API Gateway, nên dùng:

```text
A Record / AAAA Record + Alias
```

Thay vì cố dùng CNAME.

Đặc biệt với root domain:

```text
example.com
```

bạn cần Alias vì CNAME không dùng được cho zone apex.

---

## 4.2. Không đặt TTL quá cao trước migration

Trước khi migration domain, đổi ALB, đổi CloudFront hoặc chuyển traffic sang hệ thống mới, hãy giảm TTL trước.

Ví dụ:

```text
Trước migration 1 ngày:
TTL = 300

Trong lúc migration:
TTL = 60 hoặc 300

Sau khi ổn định:
TTL = 300 / 600 / 3600
```

Không nên để TTL `86400` rồi mới đổi record, vì khi đó quá trình cập nhật DNS có thể mất nhiều thời gian.

---

## 4.3. Dùng Health Check + Failover cho hệ thống quan trọng

Với production cần high availability:

```text
Primary endpoint: ALB production
Secondary endpoint: backup ALB hoặc S3 static maintenance page
```

Kết hợp:

```text
Health Check
+ Failover Routing Policy
```

để Route 53 tự chuyển hướng khi primary lỗi.

---

## 4.4. Quản lý quyền Route 53 theo Least Privilege

Không nên cho developer toàn quyền:

```json
"route53:*"
```

Nếu họ chỉ cần sửa record trong một hosted zone cụ thể, hãy giới hạn quyền theo hosted zone.

Ví dụ concept:

```json
{
  "Effect": "Allow",
  "Action": [
    "route53:ChangeResourceRecordSets",
    "route53:ListResourceRecordSets"
  ],
  "Resource": "arn:aws:route53:::hostedzone/Z1234567890ABC"
}
```

---

## 4.5. Bảo vệ domain quan trọng

Với domain production, nên có:

```text
MFA cho root account và IAM user quan trọng
Domain lock / transfer lock
Không chia sẻ quyền Route 53 bừa bãi
Không dùng access key cá nhân để automation DNS nếu không cần
CloudTrail để audit thay đổi
```

Một record DNS sai có thể làm website down, email lỗi, SSL fail hoặc traffic đi nhầm nơi.

---

## 4.6. Tách Public Hosted Zone và Private Hosted Zone rõ ràng

Không nên lẫn lộn:

```text
Public:
example.com
api.example.com

Private:
service.internal
db.internal
api.internal
```

Public Hosted Zone dành cho Internet.  
Private Hosted Zone dành cho nội bộ VPC.

---

## 4.7. Dùng naming convention và tagging

Ví dụ naming convention:

```text
prod-example-com
stg-example-com
dev-internal-local
```

Tag gợi ý:

```text
Environment = Production
Owner       = PlatformTeam
Project     = VideoApp
ManagedBy   = Terraform
```

Điều này giúp quản lý tốt hơn khi công ty có nhiều domain và nhiều hosted zone.

---

## 4.8. Theo dõi thay đổi DNS bằng CloudTrail

Bật CloudTrail để biết ai đã thay đổi DNS record.

Cần theo dõi các action như:

```text
ChangeResourceRecordSets
CreateHostedZone
DeleteHostedZone
UpdateHealthCheck
DeleteHealthCheck
```

Khi production lỗi do DNS, CloudTrail là nơi rất đáng xem đầu tiên.

---

## 4.9. Cẩn thận với MX/TXT record

Record MX/TXT thường liên quan đến email.

Sai các record này có thể làm:

```text
Không nhận được email
Email gửi vào spam
SPF/DKIM/DMARC fail
Không verify được domain với Google/Microsoft/AWS SES
```

Ví dụ TXT cho SPF:

```text
example.com TXT "v=spf1 include:_spf.google.com ~all"
```

Ví dụ TXT cho DMARC:

```text
_dmarc.example.com TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
```

---

## 4.10. Dùng Weighted Routing cho blue/green hoặc canary deployment

Ví dụ deploy version mới:

```text
api.example.com → ALB old version    weight 90
api.example.com → ALB new version    weight 10
```

Nếu version mới ổn:

```text
old: 50
new: 50
```

Sau đó:

```text
old: 0
new: 100
```

Nếu version mới lỗi, rollback rất nhanh:

```text
old: 100
new: 0
```

---

# 5. Các use case thực tế

## Kịch bản 1: Trỏ domain production tới ALB/CloudFront

### Bối cảnh

Công ty có web app chạy trên EC2 sau ALB.

```text
User → Route 53 → ALB → EC2
```

Hoặc nếu dùng CDN:

```text
User → Route 53 → CloudFront → ALB/S3
```

### Cấu hình DNS

```text
example.com      A Alias    ALB hoặc CloudFront
www.example.com  A Alias    ALB hoặc CloudFront
```

### Khi nào dùng ALB?

Dùng khi app backend chạy trên:

```text
EC2
ECS
EKS
Lambda target
```

### Khi nào dùng CloudFront?

Dùng khi muốn:

```text
Cache static content
Giảm latency cho user toàn cầu
Bảo vệ origin tốt hơn
Tích hợp WAF
Dùng HTTPS edge location
```

---

## Kịch bản 2: Failover DNS giữa primary site và backup site

### Bối cảnh

Production chạy ở Tokyo Region:

```text
Primary: ALB Tokyo
```

Backup chạy ở Singapore hoặc static maintenance page:

```text
Secondary: ALB Singapore hoặc S3 static website
```

### Kiến trúc

```text
User
  ↓
Route 53 Failover Routing
  ↓
Primary healthy? → ALB Tokyo
Primary failed?  → Backup site
```

### Record ví dụ

```text
example.com A Alias ALB-Tokyo     Failover: Primary
example.com A Alias ALB-Backup    Failover: Secondary
```

Health Check:

```text
https://example.com/health
```

Nếu primary fail, Route 53 chuyển sang secondary.

Use case này phù hợp với:

```text
E-commerce
SaaS production
Hệ thống đặt hàng
Ứng dụng cần uptime cao
```

---

## Kịch bản 3: DNS nội bộ cho microservices trong VPC

### Bối cảnh

Bạn có nhiều service nội bộ:

```text
user-service
order-service
payment-service
inventory-service
```

Không muốn hardcode IP vì IP có thể thay đổi.

### Dùng Private Hosted Zone

Domain nội bộ:

```text
company.internal
```

Record:

```text
user.company.internal       A/CNAME/Internal ALB
order.company.internal      A/CNAME/Internal ALB
payment.company.internal    A/CNAME/Internal ALB
```

Ứng dụng gọi nhau bằng DNS:

```bash
curl http://order.company.internal/orders
curl http://payment.company.internal/payments
```

Thay vì:

```bash
curl http://10.0.12.45/orders
```

Lợi ích:

```text
Dễ thay đổi backend
Không phụ thuộc IP cố định
Phù hợp microservices
Chỉ truy cập được trong VPC
An toàn hơn public DNS
```

---

# 6. Bảng tổng kết nhanh

| Thành phần          | Hiểu đơn giản               | Khi nào dùng                         |
| ------------------- | --------------------------- | ------------------------------------ |
| Route 53            | DNS service của AWS         | Quản lý domain/DNS                   |
| Domain Registration | Mua domain                  | Khi muốn mua domain qua AWS          |
| Hosted Zone         | Nơi chứa DNS records        | Mỗi domain thường có một hosted zone |
| A Record            | Domain → IPv4               | Trỏ tới IP                           |
| AAAA Record         | Domain → IPv6               | Trỏ tới IPv6                         |
| CNAME               | Domain → domain khác        | `www → example.com`                  |
| Alias               | Domain → AWS resource       | ALB, CloudFront, S3, API Gateway     |
| TTL                 | Thời gian cache DNS         | Điều chỉnh khi migration             |
| Routing Policy      | Cách Route 53 trả lời DNS   | Weighted, Failover, Latency...       |
| Health Check        | Kiểm tra endpoint sống/chết | Failover, HA                         |
| Private Hosted Zone | DNS nội bộ VPC              | Microservices, internal system       |
| DNSSEC              | Ký số DNS response          | Chống giả mạo DNS                    |

---

# 7. Checklist thực hành Route 53 cho production

```text
[ ] Domain đã trỏ đúng Name Server về Route 53
[ ] Hosted Zone đúng domain
[ ] Root domain dùng A/AAAA Alias
[ ] www domain dùng Alias hoặc CNAME hợp lý
[ ] ALB/CloudFront có certificate ACM hợp lệ
[ ] TTL phù hợp, không quá cao trước migration
[ ] MX/TXT/SPF/DKIM/DMARC không bị ghi sai
[ ] IAM Route 53 theo least privilege
[ ] CloudTrail bật để audit thay đổi DNS
[ ] Health Check + Failover nếu cần HA
[ ] Private Hosted Zone tách riêng với Public Hosted Zone
```

---

# 8. Ghi nhớ nhanh

Route 53 không chỉ là nơi “trỏ domain”.

Nó là lớp điều hướng đầu tiên trước khi request chạm tới hệ thống của bạn.

```text
User gõ domain
→ DNS resolve qua Route 53
→ Route 53 quyết định trả về endpoint nào
→ Request đi tới ALB/CloudFront/API Gateway/S3
→ Application xử lý request
```

Với hệ thống production trên AWS, Route 53 thường đi chung với:

```text
Route 53 + ACM + CloudFront + ALB + EC2/ECS
Route 53 + Health Check + Failover
Route 53 + Private Hosted Zone + VPC
```

Câu cần nhớ:

> **Route 53 trả lời câu hỏi: domain này nên đi tới đâu, theo điều kiện nào, và khi hệ thống lỗi thì chuyển hướng ra sao.**
