## 1. Ý chính 

Video giải thích cách **AWS WAF tích hợp trực tiếp với CloudFront** để bảo vệ website ở **Layer 7 – Application Layer**.

Cách nhớ đơn giản:

> **CloudFront = cửa trước của website**
> **AWS WAF = bảo vệ đứng trước cửa, kiểm tra từng request**

Luồng request:

```text
User
  ↓
CloudFront
  ↓
AWS WAF kiểm tra request
  ├─ Request tốt → Allow
  ├─ Request đáng ngờ → Count / Challenge / CAPTCHA
  └─ Request xấu → Block
  ↓
ALB / EC2 / ECS / S3
```

Khi WAF kết hợp với CloudFront, request xấu có thể bị chặn trước khi tới origin, giúp giảm tải cho hệ thống backend. 

---

# 2. AWS WAF là gì?

AWS WAF là **Web Application Firewall**.

Nó kiểm tra HTTP/HTTPS request và có thể quyết định:

```text
ALLOW      → Cho qua
BLOCK      → Chặn
COUNT      → Chỉ ghi nhận, chưa chặn
CAPTCHA    → Bắt giải CAPTCHA
CHALLENGE  → Kiểm tra client trước khi cho qua
```

WAF có thể kiểm tra các thành phần như:

* IP address
* HTTP header
* URI path
* Query string
* Request body
* SQL Injection
* Cross-Site Scripting
* Request rate

Tài liệu SAA cũng nhấn mạnh WAF có thể allow, block, rate-limit hoặc monitor request dựa trên các điều kiện ở tầng ứng dụng. 

---

# 3. Điểm quan trọng nhất: Monitor trước, Block sau

Đây là phần nên nhớ nhất.

## Monitor Mode

Ví dụ bạn vừa bật WAF:

```text
User bình thường       → COUNT
Bot                     → COUNT
Scanner                 → COUNT
SQL Injection attempt   → COUNT
```

Không ai bị block.

Mục tiêu là quan sát:

* request nào thường xuyên xuất hiện;
* quốc gia nào gửi traffic;
* IP nào bất thường;
* rule nào match nhiều;
* liệu có request hợp lệ bị nhận diện nhầm không.

### Tại sao phải làm như vậy?

Giả sử website Laravel có API:

```text
POST /api/search
```

Request hợp lệ:

```json
{
  "keyword": "select table for office"
}
```

Một rule quá nhạy có thể hiểu chữ `select` là dấu hiệu SQL Injection.

Nếu bật `BLOCK` ngay:

```text
Khách hàng thật
      ↓
Request hợp lệ
      ↓
WAF false positive
      ↓
403 Forbidden
```

Vì vậy cách triển khai production tốt hơn là:

```text
COUNT
  ↓
Quan sát
  ↓
Phân tích false positive
  ↓
Tinh chỉnh rule
  ↓
BLOCK
```

---

# 4. Ví dụ thực tế rất dễ hiểu

Giả sử bạn có website bán vé:

```text
example.com
```

Kiến trúc:

```text
Internet
    ↓
CloudFront
    ↓
AWS WAF
    ↓
ALB
    ↓
EC2 Auto Scaling
    ↓
RDS
```

## Trường hợp A: Người dùng bình thường

Người dùng mở:

```text
GET /products/123
```

Luồng:

```text
CloudFront
    ↓
WAF kiểm tra
    ↓
Không có vấn đề
    ↓
ALLOW
    ↓
Backend
```

---

## Trường hợp B: Hacker thử SQL Injection

Request:

```text
/login?id=1 OR 1=1
```

Luồng:

```text
CloudFront
    ↓
AWS WAF
    ↓
SQL Injection rule match
    ↓
BLOCK
```

Backend không cần xử lý request đó.

---

## Trường hợp C: Bot spam API

Một IP gọi:

```text
/api/login
```

10.000 lần trong thời gian ngắn.

Bạn có thể sử dụng **Rate-based rule**:

```text
Nếu một nguồn gửi quá nhiều request
        ↓
Block tạm thời
```

Rate-based rule đặc biệt hữu ích khi cần kiểm soát request volume theo nguồn; study guide cũng dùng đây làm pattern cho việc hạn chế lượng request bất thường. 

---

# 5. AWS Managed Rules trong video là gì?

Video đề cập các nhóm rule như:

```text
Amazon IP Reputation List
Common Rule Set
Known Bad Inputs
Bot Control
```

Hiểu đơn giản:

## Common Rule Set

Bảo vệ các kiểu attack phổ biến.

Ví dụ:

```text
SQL Injection
XSS
Path Traversal
Local File Inclusion
```

---

## Amazon IP Reputation List

AWS duy trì danh sách các IP có lịch sử đáng ngờ.

Ví dụ:

```text
IP 203.x.x.x
```

đã từng tham gia hoạt động scanning hoặc tấn công.

Request:

```text
203.x.x.x
    ↓
CloudFront
    ↓
WAF IP Reputation rule
    ↓
BLOCK
```

Điểm mạnh là bạn không cần tự duy trì toàn bộ danh sách IP xấu.

---

## Known Bad Inputs

Phát hiện những request có pattern bất thường hoặc payload thường dùng trong tấn công.

Ví dụ:

```text
../../../etc/passwd
```

hoặc payload request có đặc điểm nguy hiểm.

---

# 6. Bot Control là gì?

Không phải bot nào cũng xấu.

Ví dụ:

### Bot tốt

```text
Googlebot
Bingbot
Search engine crawler
Monitoring bot
```

### Bot xấu

```text
Credential stuffing bot
Scraping bot
Spam bot
Inventory bot
Vulnerability scanner
```

WAF Bot Control giúp quan sát và phân loại bot.

Bạn có thể áp dụng:

```text
Known search bot → Allow

Unknown bot
       ↓
Challenge

Scraping bot
       ↓
Block
```

## Ví dụ thực tế

Website bán hàng có:

```text
100.000 requests/ngày
```

Phân tích thấy:

```text
Human users      60%
Search bots       5%
Unknown bots     20%
Scraping bots    15%
```

Không nên làm:

```text
Block tất cả bot
```

vì có thể chặn luôn Googlebot.

Tốt hơn:

```text
Googlebot       → Allow
Unknown bot     → Challenge
Scraper         → Block
```

---

# 7. CloudFront Security Dashboard dùng để làm gì?

Trong CloudFront Console, Security dashboard giúp bạn xem trực quan:

```text
Total requests
Allowed requests
Blocked requests
Bot traffic
Top countries
Top IPs
Top URI paths
Rule matches
```

Ví dụ bạn thấy:

```text
US        40%
Japan     35%
Vietnam   15%
Others    10%
```

Sau đó phát hiện:

```text
IP: 1.2.3.4
Requests: 500.000
URI: /login
```

Bạn có thể điều tra tiếp:

```text
Đây là user thật?
Bot?
Brute force?
Load test?
Internal monitoring?
```

Điểm quan trọng: **không nên nhìn thấy traffic bất thường là block ngay lập tức**.

---

# 8. WAF Logs dùng để điều tra như thế nào?

Ví dụ bạn muốn điều tra request từ một quốc gia.

Tư duy:

```text
Filter Country = US
```

Sau đó xem:

```text
Client IP
URI
HTTP method
Rule matched
Action
User-Agent
Timestamp
```

Ví dụ:

```text
Country: US
IP: 10.x.x.x
URI: /wp-login.php
Action: COUNT
```

Trong khi website của bạn là Laravel và hoàn toàn không dùng WordPress.

Đây là tín hiệu khá rõ của:

```text
Automated vulnerability scanner
```

Bạn có thể quyết định:

```text
COUNT → quan sát thêm
```

hoặc:

```text
BLOCK → nếu pattern rõ ràng
```

---

# 9. CloudFront Console và WAF Console khác nhau thế nào?

## CloudFront Console

Dành cho người muốn:

```text
Bật bảo vệ nhanh
Xem dashboard dễ hiểu
Monitor traffic
Bật blocking
```

## WAF Console

Dành cho cấu hình sâu hơn:

```text
Custom Web ACL
Custom rule
Rule priority
Rate-based rules
IP Set
Regex
Header inspection
URI inspection
Bot Control tuning
Logging
Rule exclusions
```

Dễ nhớ:

```text
CloudFront Security
        =
Simple view

WAF Console
        =
Advanced control
```

---

# 10. Một điểm rất quan trọng trong video

Khi bật security protection từ CloudFront, AWS có thể tạo và liên kết WAF Web ACL cho distribution.

Sau đó bạn vẫn có thể vào:

```text
AWS WAF
  ↓
Web ACLs
  ↓
CloudFront scope
```

để xem và chỉnh cấu hình chi tiết hơn.

Nhưng về quản trị production, nên tránh thay đổi một cách ngẫu hứng giữa nhiều nơi. Cấu hình bảo mật nên được quản lý bằng IaC như:

```text
Terraform
CloudFormation
AWS CDK
```

để đảm bảo traceability và hạn chế configuration drift. Điều này phù hợp với nguyên tắc automation, monitoring và Infrastructure as Code trong hướng dẫn kiến trúc của project. 

---

# 11. Cách triển khai tôi khuyến nghị cho production

Không nên:

```text
Enable WAF
   ↓
Bật toàn bộ rules thành BLOCK
   ↓
Production lỗi 403
```

Nên:

```text
Phase 1
CloudFront + WAF
        ↓
Managed Rules = COUNT

Phase 2
Thu thập dữ liệu
        ↓
Phân tích Top IP / URI / Country / Rule match

Phase 3
Xử lý false positive
        ↓
Scope-down statement / exception

Phase 4
Chuyển rule an toàn sang BLOCK

Phase 5
Rate limit các endpoint nhạy cảm
/login
/register
/password-reset
/api/*
```

Đây là cách tiếp cận an toàn hơn cho production vì cân bằng giữa **Security** và **Reliability**.

---

# 12. Sơ đồ để ghi nhớ

```text
                    Internet
                       │
                       ▼
                ┌────────────┐
                │ CloudFront │
                │    CDN     │
                └──────┬─────┘
                       │
                       ▼
                ┌────────────┐
                │  AWS WAF   │
                ├────────────┤
                │ SQLi       │
                │ XSS        │
                │ Bad IP     │
                │ Rate Limit │
                │ Bot        │
                └──────┬─────┘
                       │
              Good request only
                       │
                       ▼
                     ALB
                       │
                       ▼
                  ECS / EC2
                       │
                       ▼
                      RDS
```

---

# Cách nhớ trong 20 giây

> **CloudFront đưa website tới gần người dùng.**
> **WAF kiểm tra request ở Layer 7.**
> **Monitor trước để hiểu traffic và tránh false positive.**
> **Sau đó mới Block.**
> **Managed Rules xử lý tấn công phổ biến, Rate-based rule chống spam request, Bot Control xử lý bot, WAF Logs dùng để điều tra.**

Công thức:

```text
CloudFront = Delivery
WAF        = Protection
COUNT      = Observe
BLOCK      = Enforce
Logs       = Investigate
Bot Control = Control automation traffic
```

