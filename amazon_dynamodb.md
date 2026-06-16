# Amazon DynamoDB

Amazon DynamoDB là một trong những database quan trọng nhất trong hệ sinh thái AWS, đặc biệt khi xây dựng ứng dụng **serverless**, **mobile/web app**, **hệ thống traffic lớn**, hoặc các workload cần đọc/ghi cực nhanh.

---

# 1. Khái niệm và Phép ẩn dụ

## 1.1. Amazon DynamoDB là gì?

**Amazon DynamoDB** là dịch vụ **NoSQL Database được quản lý hoàn toàn bởi AWS**.

Hiểu đơn giản:

> DynamoDB giống như **một kho dữ liệu siêu nhanh dạng bảng key-value/document**, nơi ứng dụng có thể đọc/ghi dữ liệu với tốc độ rất cao mà không cần tự cài server database, không cần tự scale, không cần backup thủ công, không cần lo patch OS/database engine.

Ví dụ dễ hiểu:

Bạn có một hệ thống web/mobile cần lưu:

- Session đăng nhập
- Token tạm thời
- Giỏ hàng
- User preference
- Notification status
- Tracking event
- IoT sensor data
- Game leaderboard

Những dữ liệu này thường có đặc điểm:

- Cấu trúc linh hoạt
- Đọc/ghi nhiều
- Cần phản hồi nhanh
- Không nhất thiết cần join phức tạp như SQL
- Có thể tăng traffic rất nhanh

Đây chính là “đất diễn” của DynamoDB.

---

## 1.2. NoSQL là gì?

**NoSQL** là nhóm database không bắt buộc lưu dữ liệu theo kiểu bảng quan hệ truyền thống như MySQL/PostgreSQL.

Trong database quan hệ, ta thường có:

```text
users table
orders table
order_items table
products table
```

Sau đó truy vấn bằng SQL, dùng `JOIN`, `GROUP BY`, `FOREIGN KEY`.

Trong NoSQL như DynamoDB, dữ liệu thường được thiết kế xoay quanh **cách ứng dụng truy vấn dữ liệu**, gọi là **access pattern**.

Ví dụ:

Thay vì hỏi:

> “Tôi có những bảng nào?”

Ta hỏi trước:

> “Ứng dụng sẽ cần lấy dữ liệu theo userId, sessionId, email, createdAt, status hay orderId?”

Đây là điểm cực kỳ quan trọng khi học DynamoDB.

---

## 1.3. Vì sao doanh nghiệp dùng NoSQL?

Doanh nghiệp dùng NoSQL khi cần:

- Scale lớn, traffic tăng nhanh.
- Đọc/ghi tốc độ cao.
- Dữ liệu linh hoạt, không cố định schema.
- Không muốn vận hành database server.
- Kiến trúc event-driven hoặc serverless.
- Dữ liệu dạng key-value/document.
- Truy vấn đơn giản nhưng tần suất cực cao.

Ví dụ thực tế:

Một app mobile có 5 triệu user, mỗi user mở app nhiều lần mỗi ngày. Mỗi lần mở app cần lấy user preference, trạng thái thông báo, session token. Nếu dùng database quan hệ và thiết kế không tốt, hệ thống có thể bị nghẽn ở database. DynamoDB sinh ra để xử lý kiểu workload này rất tốt.

---

## 1.4. DynamoDB giúp lưu trữ, truy vấn, scale và bảo vệ dữ liệu như thế nào?

| Nhu cầu          | DynamoDB hỗ trợ như thế nào                                                 |
| ---------------- | --------------------------------------------------------------------------- |
| Lưu trữ dữ liệu  | Lưu dữ liệu trong **Table**, mỗi bản ghi là **Item**                        |
| Truy vấn nhanh   | Truy vấn bằng **Primary Key**, **Partition Key**, **Sort Key**, **GSI/LSI** |
| Scale            | Hỗ trợ **On-Demand Mode**, **Provisioned Mode**, Auto Scaling               |
| Bảo mật          | Tích hợp **IAM**, mã hóa bằng **AWS KMS**                                   |
| Event-driven     | Dùng **DynamoDB Streams** để trigger Lambda                                 |
| Dữ liệu tạm thời | Dùng **TTL** để tự động xóa item hết hạn                                    |
| Cache            | Dùng **DAX** để tăng tốc read query                                         |
| Multi-region     | Có thể dùng **Global Tables** cho ứng dụng global                           |

---

## 1.5. DynamoDB khác gì so với RDS/MySQL/PostgreSQL?

| Tiêu chí             | DynamoDB                                  | RDS / MySQL / PostgreSQL                         |
| -------------------- | ----------------------------------------- | ------------------------------------------------ |
| Loại database        | NoSQL key-value/document                  | Relational Database                              |
| Schema               | Linh hoạt                                 | Cố định hơn                                      |
| Truy vấn             | Theo key/index                            | SQL linh hoạt                                    |
| JOIN                 | Không phù hợp                             | Rất mạnh                                         |
| Transaction phức tạp | Có hỗ trợ nhưng không phải thế mạnh chính | Rất phù hợp                                      |
| Scale                | Scale ngang rất tốt                       | Thường scale dọc trước, scale ngang phức tạp hơn |
| Quản lý server       | Serverless/managed                        | Managed nhưng vẫn cần chọn instance              |
| Use case chính       | Session, cart, event, IoT, serverless app | ERP, accounting, order system, relational data   |
| Tư duy thiết kế      | Access pattern first                      | Data model/entity relationship first             |

Nói gọn:

> **RDS phù hợp khi dữ liệu có quan hệ phức tạp. DynamoDB phù hợp khi cần đọc/ghi cực nhanh theo key với quy mô lớn.**

---

## 1.6. Khi nào nên dùng DynamoDB?

Nên dùng DynamoDB khi:

- Ứng dụng cần đọc/ghi nhanh, latency thấp.
- Biết rõ access pattern.
- Dữ liệu truy vấn chủ yếu theo key.
- Không cần JOIN phức tạp.
- Traffic khó đoán hoặc có thể tăng đột biến.
- Làm serverless với Lambda/API Gateway.
- Lưu session, token, cart, notification, tracking event.
- Cần TTL tự động xóa dữ liệu tạm thời.

Ví dụ rất hợp:

```text
Get session by userId + sessionId
Get cart by userId
Get user preference by userId
Get notification status by userId
Get game score by gameId + score
```

---

## 1.7. Khi nào không nên dùng DynamoDB?

Không nên dùng DynamoDB nếu:

- Cần nhiều JOIN giữa các bảng.
- Cần query ad-hoc linh hoạt như SQL.
- Cần báo cáo phức tạp kiểu BI trực tiếp trên database.
- Access pattern chưa rõ.
- Team chưa quen thiết kế NoSQL.
- Dữ liệu có quan hệ chặt chẽ và transaction phức tạp.
- Bạn muốn search full-text như Elasticsearch/OpenSearch.

Ví dụ không phù hợp:

```sql
SELECT users.name, orders.total, products.name
FROM users
JOIN orders ON users.id = orders.user_id
JOIN order_items ON orders.id = order_items.order_id
JOIN products ON order_items.product_id = products.id
WHERE orders.created_at BETWEEN ...
GROUP BY users.id
```

Query kiểu này nên dùng RDS, Redshift, Athena hoặc OpenSearch tùy bài toán.

---

# 2. Các thành phần then chốt

## 2.1. Tổng quan nhanh

| Thành phần    | Giải thích dễ hiểu                                    |
| ------------- | ----------------------------------------------------- |
| Table         | Nơi lưu dữ liệu                                       |
| Item          | Một bản ghi trong table                               |
| Attribute     | Thuộc tính của item                                   |
| Primary Key   | Khóa chính để xác định/truy vấn item                  |
| Partition Key | Khóa dùng để phân phối dữ liệu và truy vấn            |
| Sort Key      | Khóa dùng để sắp xếp/range query trong cùng partition |
| GSI           | Index phụ để query theo pattern khác                  |
| LSI           | Index phụ cùng Partition Key nhưng khác Sort Key      |
| RCU/WCU       | Đơn vị capacity đọc/ghi                               |
| On-Demand     | Tự scale theo request, trả theo dùng                  |
| Provisioned   | Khai báo trước capacity đọc/ghi                       |
| Streams       | Ghi lại thay đổi item                                 |
| TTL           | Tự xóa item hết hạn                                   |
| DAX           | Cache in-memory cho DynamoDB                          |
| Encryption    | Mã hóa dữ liệu bằng AWS KMS                           |

---

## 2.2. Table

**Table** là nơi lưu trữ dữ liệu trong DynamoDB.

Ví dụ:

```text
UserSessions
Users
Orders
Notifications
GameScores
DeviceEvents
```

Trong DynamoDB, một table có thể chứa rất nhiều item. Table có primary key, có thể có rất nhiều item, và mỗi item có attributes linh hoạt.

---

## 2.3. Item

**Item** là một bản ghi trong table, tương tự một row trong SQL.

Ví dụ một item trong table `UserSessions`:

```json
{
  "userId": "user-1001",
  "sessionId": "sess-abc123",
  "createdAt": "2026-06-16T10:00:00Z",
  "expiresAt": 1792130400,
  "ipAddress": "203.0.113.10",
  "userAgent": "Chrome on Windows"
}
```

Điểm khác SQL:

- Item này có thể có `ipAddress`.
- Item khác có thể không có `ipAddress`.
- Item khác nữa có thể có thêm `deviceId`, `location`, `loginMethod`.

DynamoDB không bắt buộc mọi item phải có cùng bộ attribute giống nhau.

---

## 2.4. Attribute

**Attribute** là thuộc tính của item, tương tự column trong SQL nhưng linh hoạt hơn.

Ví dụ:

```text
userId
sessionId
createdAt
expiresAt
ipAddress
userAgent
```

DynamoDB hỗ trợ nhiều kiểu dữ liệu như String, Number, Binary, Boolean, Null, List, Map và Set. Item trong DynamoDB có thể thêm attribute theo thời gian và schema có thể tiến hóa nhanh.

---

## 2.5. Primary Key

**Primary Key** là khóa chính của table. Đây là phần cực kỳ quan trọng vì DynamoDB truy vấn hiệu quả nhất thông qua key.

Có 2 kiểu Primary Key:

### Kiểu 1: Chỉ có Partition Key

Ví dụ:

```text
Table: Users
Partition Key: userId
```

Mỗi `userId` là duy nhất.

Phù hợp khi mỗi user chỉ có một record chính.

### Kiểu 2: Partition Key + Sort Key

Ví dụ:

```text
Table: UserSessions
Partition Key: userId
Sort Key: sessionId
```

Một user có thể có nhiều session.

```text
user-1001 + sess-001
user-1001 + sess-002
user-1001 + sess-003
```

Kiểu này rất phổ biến và linh hoạt hơn.

---

## 2.6. Partition Key

**Partition Key** quyết định dữ liệu được phân phối như thế nào bên trong DynamoDB.

Hãy tưởng tượng DynamoDB có rất nhiều “ngăn kho”. Partition Key giúp AWS biết item nên nằm ở ngăn nào.

Ví dụ:

```text
Partition Key = userId
```

Khi query:

```text
Lấy tất cả session của user-1001
```

DynamoDB biết cần tìm trong partition của `user-1001`, thay vì quét toàn bộ table.

### Partition Key tốt là gì?

Partition Key tốt nên:

- Có độ phân tán cao.
- Không dồn quá nhiều request vào một giá trị.
- Phù hợp với access pattern chính.

Ví dụ tốt:

```text
userId
orderId
deviceId
tenantId
gameId
```

Ví dụ dễ gây hot partition:

```text
status = ACTIVE
country = JP
type = LOGIN
```

Nếu hàng triệu request đều query `status = ACTIVE`, một partition có thể bị quá tải.

---

## 2.7. Sort Key

**Sort Key** dùng để sắp xếp và query theo range trong cùng Partition Key.

Ví dụ table `UserSessions`:

```text
Partition Key: userId
Sort Key: createdAt
```

Ta có thể query:

```text
Lấy tất cả session của user-1001
Lấy session của user-1001 trong 7 ngày gần nhất
Lấy session mới nhất của user-1001
```

Một số Sort Key hay dùng:

```text
createdAt
orderDate
sessionId
status#createdAt
score
timestamp
```

Ví dụ Sort Key dạng composite:

```text
ACTIVE#2026-06-16T10:00:00Z
EXPIRED#2026-06-15T09:00:00Z
```

Cách này giúp query theo status và thời gian hiệu quả hơn.

---

## 2.8. Global Secondary Index — GSI

**GSI** là index phụ cho phép query table theo một access pattern khác với Primary Key chính.

Ví dụ table chính:

```text
Table: UserSessions
Partition Key: userId
Sort Key: sessionId
```

Access pattern chính:

```text
Lấy session theo userId
```

Nhưng sau này bạn cần:

```text
Tìm session theo sessionId
```

Nếu `sessionId` không phải Partition Key chính, bạn có thể tạo GSI:

```text
GSI name: SessionIdIndex
Partition Key: sessionId
```

Khi đó có thể query nhanh theo `sessionId`.

### Khi nào dùng GSI?

Dùng khi cần query theo thuộc tính khác Primary Key chính:

- Query user theo email.
- Query order theo status.
- Query session theo sessionId.
- Query notification theo userId + status.
- Query product theo categoryId + price.

---

## 2.9. Local Secondary Index — LSI

**LSI** là index phụ dùng **cùng Partition Key với table chính**, nhưng khác Sort Key.

Ví dụ table chính:

```text
Partition Key: userId
Sort Key: sessionId
```

LSI:

```text
Partition Key: userId
Sort Key: createdAt
```

Khi đó với cùng `userId`, bạn có thể query session theo `createdAt`.

Điểm cần nhớ:

| Tiêu chí                     | GSI                            | LSI                                  |
| ---------------------------- | ------------------------------ | ------------------------------------ |
| Partition Key                | Có thể khác table chính        | Bắt buộc giống table chính           |
| Sort Key                     | Có thể khác                    | Khác table chính                     |
| Tạo sau khi table đã tồn tại | Có thể                         | Thường phải tạo lúc tạo table        |
| Dùng khi                     | Query theo access pattern khác | Query khác sort trong cùng partition |

---

## 2.10. Read/Write Capacity

DynamoDB tính capacity đọc/ghi bằng:

- **RCU**: Read Capacity Unit
- **WCU**: Write Capacity Unit

Nói dễ hiểu:

```text
RCU = năng lực đọc
WCU = năng lực ghi
```

Bạn không cần nhớ công thức quá sâu ngay từ đầu. Quan trọng là hiểu:

- Đọc nhiều thì tốn read capacity.
- Ghi nhiều thì tốn write capacity.
- Item càng lớn thì càng tốn capacity.
- Query đúng key tiết kiệm hơn scan toàn table.

---

## 2.11. On-Demand Mode và Provisioned Mode

| Mode                       | Cách hoạt động            | Khi nào dùng                         |
| -------------------------- | ------------------------- | ------------------------------------ |
| On-Demand                  | AWS tự scale theo request | Giai đoạn đầu, traffic khó đoán      |
| Provisioned                | Bạn khai báo RCU/WCU      | Traffic ổn định, muốn tối ưu chi phí |
| Provisioned + Auto Scaling | Tự tăng/giảm theo ngưỡng  | Production đã hiểu pattern           |

Khuyến nghị thực tế:

```text
Dev / MVP / giai đoạn đầu: On-Demand
Production ổn định: Provisioned + Auto Scaling
```

---

## 2.12. DynamoDB Streams

**DynamoDB Streams** ghi lại các thay đổi trên item:

```text
INSERT
MODIFY
REMOVE
```

Ví dụ:

Khi user tạo account mới:

```text
DynamoDB item created
→ DynamoDB Stream
→ Lambda trigger
→ Send welcome email
```

Use case hay gặp:

- Gửi email welcome.
- Đồng bộ dữ liệu sang OpenSearch.
- Ghi audit log.
- Tính analytics realtime.
- Trigger Lambda xử lý sau khi có thay đổi.

---

## 2.13. TTL — Time To Live

**TTL** cho phép DynamoDB tự động xóa item sau một thời điểm nhất định.

Ví dụ session hết hạn sau 24 giờ:

```json
{
  "userId": "user-1001",
  "sessionId": "sess-abc123",
  "expiresAt": 1792130400
}
```

`expiresAt` là Unix timestamp. Khi đến thời điểm đó, DynamoDB sẽ tự động xóa item.

Rất hợp cho:

- Session
- Login token
- OTP
- Temporary verification code
- Cache data
- Short-lived logs

---

## 2.14. DAX — DynamoDB Accelerator

**DAX** là cache in-memory được quản lý bởi AWS dành riêng cho DynamoDB.

Hiểu đơn giản:

```text
App → DAX → DynamoDB
```

Nếu dữ liệu đã có trong cache, DAX trả về rất nhanh.

Dùng DAX khi:

- App đọc rất nhiều.
- Dữ liệu được đọc lặp lại.
- Muốn giảm tải read vào DynamoDB.
- Cần latency cực thấp.

Không cần DAX khi:

- App chủ yếu ghi.
- Dữ liệu thay đổi liên tục.
- Traffic nhỏ.
- Query chưa tối ưu key/index.

---

## 2.15. Encryption

DynamoDB hỗ trợ mã hóa dữ liệu at-rest bằng AWS KMS.

Bạn có thể dùng:

- AWS owned key
- AWS managed key
- Customer managed KMS key

Best practice:

- Dữ liệu nhạy cảm nên dùng KMS key phù hợp.
- Phân quyền KMS key cẩn thận.
- Không lưu secret/password plain text nếu không cần.
- Field cực kỳ nhạy cảm có thể cân nhắc client-side encryption.

---

# 3. Ví dụ thực hành: Tạo DynamoDB Table lưu session/login token

## 3.1. Bài toán

Ta cần lưu session đăng nhập của user.

Yêu cầu:

- Một user có thể có nhiều session.
- Mỗi session có thời gian tạo.
- Session tự hết hạn.
- App có thể ghi session mới, đọc session, query danh sách session của user.
- Dữ liệu không cần JOIN.
- Traffic ban đầu chưa đoán được.

Thiết kế table:

```text
Table name: UserSessions
Partition Key: userId
Sort Key: sessionId
TTL attribute: expiresAt
Capacity Mode: On-Demand
```

Attributes:

| Attribute | Kiểu   | Ý nghĩa                |
| --------- | ------ | ---------------------- |
| userId    | String | ID của user            |
| sessionId | String | ID của session         |
| createdAt | String | Thời gian tạo          |
| expiresAt | Number | Unix timestamp để TTL  |
| ipAddress | String | IP đăng nhập           |
| userAgent | String | Browser/device         |
| status    | String | ACTIVE/EXPIRED/REVOKED |

---

## 3.2. Access Pattern

Trước khi tạo table, cần xác định app sẽ query gì.

| Access Pattern              | Cách xử lý                         |
| --------------------------- | ---------------------------------- |
| Tạo session mới             | `PutItem`                          |
| Lấy session cụ thể của user | `GetItem` với `userId + sessionId` |
| Lấy tất cả session của user | `Query` theo `userId`              |
| Xóa session khi logout      | `DeleteItem`                       |
| Tự xóa session hết hạn      | TTL theo `expiresAt`               |

Lưu ý hay:

Nếu bạn cần tìm session chỉ bằng `sessionId` mà không có `userId`, nên thêm GSI:

```text
GSI: SessionIdIndex
Partition Key: sessionId
```

---

## 3.3. Tạo table bằng AWS CLI

```bash
aws dynamodb create-table \
  --table-name UserSessions \
  --attribute-definitions \
      AttributeName=userId,AttributeType=S \
      AttributeName=sessionId,AttributeType=S \
  --key-schema \
      AttributeName=userId,KeyType=HASH \
      AttributeName=sessionId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region ap-northeast-1
```

Giải thích:

| Tham số                                   | Ý nghĩa             |
| ----------------------------------------- | ------------------- |
| `AttributeName=userId,AttributeType=S`    | userId là String    |
| `AttributeName=sessionId,AttributeType=S` | sessionId là String |
| `KeyType=HASH`                            | Partition Key       |
| `KeyType=RANGE`                           | Sort Key            |
| `PAY_PER_REQUEST`                         | On-Demand Mode      |
| `ap-northeast-1`                          | Tokyo Region        |

---

## 3.4. Bật TTL

Giả sử TTL dùng attribute `expiresAt`.

```bash
aws dynamodb update-time-to-live \
  --table-name UserSessions \
  --time-to-live-specification "Enabled=true, AttributeName=expiresAt" \
  --region ap-northeast-1
```

Kiểm tra TTL:

```bash
aws dynamodb describe-time-to-live \
  --table-name UserSessions \
  --region ap-northeast-1
```

---

## 3.5. Put item — ghi session mới

Ví dụ `expiresAt` là Unix timestamp.

```bash
aws dynamodb put-item \
  --table-name UserSessions \
  --item '{
    "userId": {"S": "user-1001"},
    "sessionId": {"S": "sess-abc123"},
    "createdAt": {"S": "2026-06-16T10:00:00Z"},
    "expiresAt": {"N": "1792130400"},
    "ipAddress": {"S": "203.0.113.10"},
    "userAgent": {"S": "Chrome on Windows"},
    "status": {"S": "ACTIVE"}
  }' \
  --region ap-northeast-1
```

---

## 3.6. Get item — lấy đúng một session

```bash
aws dynamodb get-item \
  --table-name UserSessions \
  --key '{
    "userId": {"S": "user-1001"},
    "sessionId": {"S": "sess-abc123"}
  }' \
  --region ap-northeast-1
```

---

## 3.7. Query item — lấy tất cả session của user

```bash
aws dynamodb query \
  --table-name UserSessions \
  --key-condition-expression "userId = :uid" \
  --expression-attribute-values '{
    ":uid": {"S": "user-1001"}
  }' \
  --region ap-northeast-1
```

---

## 3.8. Query session theo prefix hoặc range

Nếu Sort Key là `createdAt` thay vì `sessionId`, bạn có thể query theo thời gian dễ hơn.

Ví dụ thiết kế khác:

```text
Partition Key: userId
Sort Key: createdAt
```

Query session sau một thời điểm:

```bash
aws dynamodb query \
  --table-name UserSessions \
  --key-condition-expression "userId = :uid AND createdAt >= :fromTime" \
  --expression-attribute-values '{
    ":uid": {"S": "user-1001"},
    ":fromTime": {"S": "2026-06-01T00:00:00Z"}
  }' \
  --region ap-northeast-1
```

Vì vậy, chọn Sort Key phụ thuộc vào cách app cần query.

---

## 3.9. IAM Policy cho ứng dụng

Policy dưới đây cho phép app chỉ đọc/ghi/xóa/query table `UserSessions`.

Thay:

```text
ACCOUNT_ID
ap-northeast-1
```

bằng thông tin AWS account thật.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UserSessionsTableAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ],
      "Resource": [
        "arn:aws:dynamodb:ap-northeast-1:ACCOUNT_ID:table/UserSessions"
      ]
    }
  ]
}
```

Nếu có GSI, cần thêm index ARN:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UserSessionsTableAndIndexAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ],
      "Resource": [
        "arn:aws:dynamodb:ap-northeast-1:ACCOUNT_ID:table/UserSessions",
        "arn:aws:dynamodb:ap-northeast-1:ACCOUNT_ID:table/UserSessions/index/*"
      ]
    }
  ]
}
```

Best practice: gắn policy này vào **IAM Role của EC2/Lambda/ECS Task**, không hardcode Access Key trong source code.

---

## 3.10. Ví dụ code Python với boto3

```python
import time
import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource("dynamodb", region_name="ap-northeast-1")
table = dynamodb.Table("UserSessions")


def create_session(user_id: str, session_id: str, ip_address: str, user_agent: str):
    now = int(time.time())
    expires_at = now + 24 * 60 * 60  # 24 giờ

    item = {
        "userId": user_id,
        "sessionId": session_id,
        "createdAt": now,
        "expiresAt": expires_at,
        "ipAddress": ip_address,
        "userAgent": user_agent,
        "status": "ACTIVE"
    }

    try:
        table.put_item(Item=item)
        return item
    except ClientError as e:
        print(f"Failed to create session: {e}")
        raise


def get_session(user_id: str, session_id: str):
    try:
        response = table.get_item(
            Key={
                "userId": user_id,
                "sessionId": session_id
            }
        )
        return response.get("Item")
    except ClientError as e:
        print(f"Failed to get session: {e}")
        raise


def query_user_sessions(user_id: str):
    try:
        response = table.query(
            KeyConditionExpression="userId = :uid",
            ExpressionAttributeValues={
                ":uid": user_id
            }
        )
        return response.get("Items", [])
    except ClientError as e:
        print(f"Failed to query sessions: {e}")
        raise


if __name__ == "__main__":
    create_session(
        user_id="user-1001",
        session_id="sess-abc123",
        ip_address="203.0.113.10",
        user_agent="Chrome on Windows"
    )

    session = get_session("user-1001", "sess-abc123")
    print(session)

    sessions = query_user_sessions("user-1001")
    print(sessions)
```

---

# 4. AWS Best Practices — Quy tắc vàng khi dùng DynamoDB

## 4.1. Thiết kế access pattern trước khi thiết kế table

Đây là quy tắc quan trọng nhất.

Với RDS, bạn thường thiết kế entity trước:

```text
users
orders
products
payments
```

Với DynamoDB, hãy bắt đầu bằng câu hỏi:

```text
Ứng dụng sẽ cần query dữ liệu như thế nào?
```

Ví dụ:

| Câu hỏi                            | Thiết kế key/index            |
| ---------------------------------- | ----------------------------- |
| Lấy session theo user              | PK = userId                   |
| Lấy session cụ thể                 | PK = userId, SK = sessionId   |
| Lấy order theo user và thời gian   | PK = userId, SK = orderDate   |
| Lấy order theo status              | GSI PK = status               |
| Lấy event theo device và timestamp | PK = deviceId, SK = timestamp |

---

## 4.2. Chọn Partition Key tốt để tránh hot partition

Partition Key không tốt có thể làm một phần table bị quá tải.

Không nên chọn key có quá ít giá trị:

```text
status = ACTIVE / INACTIVE
country = JP / VN / US
type = LOGIN / LOGOUT
```

Nên chọn key có độ phân tán cao:

```text
userId
deviceId
orderId
tenantId
sessionId
```

Ví dụ xấu:

```text
Partition Key = status
```

Nếu 90% item có `status = ACTIVE`, request sẽ dồn vào một partition.

Ví dụ tốt hơn:

```text
Partition Key = userId
Sort Key = status#createdAt
```

---

## 4.3. Dùng Sort Key để hỗ trợ query theo thời gian/trạng thái/thứ tự

Sort Key giúp bạn query range rất tiện.

Ví dụ:

```text
PK = userId
SK = createdAt
```

Query:

```text
Lấy order của user trong tháng này
Lấy session gần nhất
Lấy event từ thời điểm A đến B
```

Composite Sort Key rất mạnh:

```text
SK = STATUS#TIMESTAMP
```

Ví dụ:

```text
ACTIVE#2026-06-16T10:00:00Z
EXPIRED#2026-06-15T10:00:00Z
```

---

## 4.4. Sử dụng GSI khi cần query theo thuộc tính khác Primary Key

Ví dụ table `Orders`:

```text
PK = userId
SK = orderId
```

Nếu cần query:

```text
Lấy tất cả order đang PENDING
```

Bạn có thể tạo GSI:

```text
GSI PK = status
GSI SK = createdAt
```

Nhưng nhớ: GSI cũng tốn chi phí lưu trữ và capacity. Đừng tạo index “cho vui”.

---

## 4.5. Không scan toàn bộ table trong production nếu dữ liệu lớn

`Scan` đọc toàn bộ table. Với table nhỏ thì không sao, nhưng với table lớn có thể:

- Chậm.
- Tốn tiền.
- Tốn capacity.
- Gây ảnh hưởng production.

Ưu tiên:

```text
Query theo Partition Key
Query qua GSI
Thiết kế lại access pattern
Đưa dữ liệu sang OpenSearch/Athena nếu cần search/report
```

Một câu dễ nhớ:

> DynamoDB mạnh khi **Query đúng key**, yếu khi bạn bắt nó làm việc như SQL analytics database.

---

## 4.6. Dùng On-Demand Mode khi traffic khó dự đoán

Giai đoạn đầu hoặc workload chưa rõ traffic:

```text
Billing Mode = On-Demand
```

Phù hợp cho:

- MVP
- Startup app
- Campaign ngắn hạn
- Traffic spike bất ngờ
- Dev/staging

Ưu điểm:

- Không cần tính RCU/WCU trước.
- AWS tự scale theo request.
- Ít rủi ro throttling do cấu hình capacity thấp.

Nhược điểm:

- Có thể đắt hơn nếu traffic đều và lớn.

---

## 4.7. Dùng Provisioned Mode + Auto Scaling khi traffic ổn định

Khi production đã có pattern rõ:

```text
Provisioned Mode + Auto Scaling
```

Phù hợp cho:

- Traffic ổn định.
- Có dự đoán tải.
- Muốn tối ưu chi phí.
- Hệ thống chạy 24/7 với lượng request đều.

---

## 4.8. Bật TTL cho dữ liệu tạm thời

Rất nên bật TTL cho:

```text
session
login token
otp
verification code
temporary cache
short-term logs
cart expire data
```

Ví dụ:

```json
{
  "userId": "user-1001",
  "sessionId": "sess-abc123",
  "expiresAt": 1792130400
}
```

Lợi ích:

- Giảm dung lượng lưu trữ.
- Giảm chi phí.
- Tự động cleanup.
- Tránh giữ dữ liệu hết giá trị.

---

## 4.9. Dùng IAM Role thay vì hardcode Access Key

Không nên làm:

```env
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
```

Nên làm:

- EC2: gắn IAM Role vào instance.
- Lambda: dùng Execution Role.
- ECS: dùng Task Role.
- EKS: dùng IRSA.
- Local dev: dùng AWS profile hoặc SSO.

---

## 4.10. Bật encryption và áp dụng Least Privilege

Best practice bảo mật:

- Chỉ cấp quyền đúng table cần dùng.
- Chỉ cấp action cần thiết: `GetItem`, `PutItem`, `Query`, không cấp `dynamodb:*` nếu không cần.
- Dùng KMS phù hợp.
- Tách role cho từng service.
- Không dùng root account.
- Bật CloudTrail để audit API activity.

---

## 4.11. Theo dõi CloudWatch Metrics

Các metrics nên monitor:

| Metric                          | Ý nghĩa                     |
| ------------------------------- | --------------------------- |
| `ConsumedReadCapacityUnits`     | Read capacity đã dùng       |
| `ConsumedWriteCapacityUnits`    | Write capacity đã dùng      |
| `ProvisionedReadCapacityUnits`  | Read capacity đã provision  |
| `ProvisionedWriteCapacityUnits` | Write capacity đã provision |
| `ThrottledRequests`             | Request bị throttle         |
| `ReadThrottleEvents`            | Sự kiện throttle khi đọc    |
| `WriteThrottleEvents`           | Sự kiện throttle khi ghi    |
| `SuccessfulRequestLatency`      | Latency request             |
| `SystemErrors`                  | Lỗi phía service            |
| `UserErrors`                    | Lỗi do request/app          |

Nên tạo alarm cho:

```text
ThrottledRequests > 0
SystemErrors tăng bất thường
Latency tăng bất thường
Consumed capacity gần chạm provisioned capacity
```

---

## 4.12. Dùng DynamoDB Streams + Lambda cho event-driven architecture

Ví dụ flow:

```text
User đăng ký
→ PutItem vào DynamoDB Users table
→ DynamoDB Stream phát event INSERT
→ Lambda nhận event
→ Gửi welcome email qua SES
```

Hoặc:

```text
Order status changed
→ DynamoDB Stream
→ Lambda
→ Push notification
→ Update analytics table
```

Đây là pattern rất phổ biến trong serverless architecture.

---

# 5. Các Use Case thực tế

## 5.1. Kịch bản 1: Lưu session, token, cart hoặc user preference

### Bài toán

Một web/mobile app cần lưu session đăng nhập và giỏ hàng tạm thời.

Yêu cầu:

- Đọc/ghi nhanh.
- Dữ liệu theo user.
- Không cần JOIN.
- Session/cart có thể hết hạn.
- Traffic có thể tăng bất ngờ.

### Thiết kế gợi ý

```text
Table: UserSessions
PK: userId
SK: sessionId
TTL: expiresAt
```

```text
Table: UserCarts
PK: userId
SK: productId
Attributes: quantity, addedAt, expiresAt
```

### Vì sao DynamoDB phù hợp?

- Query theo user rất nhanh.
- TTL tự cleanup session/cart cũ.
- On-Demand phù hợp traffic khó đoán.
- Không cần quản lý database server.

---

## 5.2. Kịch bản 2: Serverless API với API Gateway, Lambda và DynamoDB

### Kiến trúc

```text
Client / Mobile App
        ↓
API Gateway
        ↓
Lambda
        ↓
DynamoDB
```

### Ví dụ use case

Ứng dụng Todo List:

- User tạo todo.
- User xem danh sách todo.
- User update trạng thái todo.
- User xóa todo.

### Thiết kế table

```text
Table: UserTodos
PK: userId
SK: todoId
Attributes:
- title
- status
- createdAt
- updatedAt
```

### Vì sao DynamoDB phù hợp?

- Serverless end-to-end.
- Không cần EC2.
- Không cần RDS instance.
- Scale theo request.
- Chi phí tốt khi traffic nhỏ/vừa.
- Dễ tích hợp Lambda.

---

## 5.3. Kịch bản 3: Tracking event, IoT data, game leaderboard, notification status

### Ví dụ tracking event

```text
Table: UserEvents
PK: userId
SK: timestamp
Attributes:
- eventType
- page
- device
- metadata
```

Query:

```text
Lấy event của user trong 24 giờ gần nhất
```

### Ví dụ IoT data

```text
Table: DeviceEvents
PK: deviceId
SK: timestamp
Attributes:
- temperature
- humidity
- battery
```

Query:

```text
Lấy dữ liệu sensor của device trong 1 giờ gần nhất
```

### Ví dụ game leaderboard

```text
Table: GameScores
PK: gameId
SK: score
Attributes:
- userId
- displayName
- createdAt
```

Có thể thiết kế sort key theo score để lấy top players.

### Ví dụ notification status

```text
Table: UserNotifications
PK: userId
SK: notificationId
Attributes:
- status
- sentAt
- readAt
```

GSI nếu cần query theo trạng thái:

```text
GSI PK: status
GSI SK: sentAt
```

---

# 6. DynamoDB Design Checklist cho Developer

Trước khi tạo table, hãy trả lời 10 câu này:

| Câu hỏi                                | Ví dụ                                |
| -------------------------------------- | ------------------------------------ |
| 1. Dữ liệu chính là gì?                | Session, Order, Event, Notification  |
| 2. App sẽ query theo gì?               | userId, orderId, deviceId            |
| 3. Có cần sort/range không?            | createdAt, timestamp, score          |
| 4. Có cần query theo field khác không? | email, status, category              |
| 5. Có dữ liệu tạm thời không?          | session/token/OTP                    |
| 6. Traffic có đoán được không?         | Không → On-Demand                    |
| 7. Có nguy cơ hot key không?           | status=ACTIVE                        |
| 8. Có cần event trigger không?         | Streams + Lambda                     |
| 9. Có cần cache không?                 | DAX                                  |
| 10. IAM role nào được truy cập?        | EC2 role, Lambda role, ECS task role |

---

# 7. Tóm tắt cực ngắn

| Ý chính          | Ghi nhớ                                         |
| ---------------- | ----------------------------------------------- |
| DynamoDB là gì?  | NoSQL database managed/serverless của AWS       |
| Mạnh ở đâu?      | Đọc/ghi nhanh, scale lớn, latency thấp          |
| Tư duy thiết kế  | Access pattern first                            |
| Thành phần chính | Table, Item, Attribute, PK, SK, GSI, LSI        |
| Capacity         | On-Demand hoặc Provisioned                      |
| Dữ liệu tạm thời | Dùng TTL                                        |
| Event-driven     | Dùng DynamoDB Streams + Lambda                  |
| Cache            | DAX                                             |
| Bảo mật          | IAM Role, Least Privilege, KMS                  |
| Tránh gì?        | Scan table lớn, hot partition, thiết kế như SQL |

---

# 8. Kết luận

Amazon DynamoDB không phải là “MySQL phiên bản NoSQL”. Nó là một database được thiết kế cho các hệ thống cần **tốc độ cao, scale lớn, vận hành đơn giản và tích hợp sâu với serverless AWS**.

Điểm khó nhất khi học DynamoDB không phải là tạo table, mà là **thiết kế key đúng**.

Một câu cần nhớ:

> Với DynamoDB, đừng bắt đầu bằng “tôi cần những bảng nào”, hãy bắt đầu bằng “ứng dụng sẽ truy vấn dữ liệu như thế nào”.

Với ví dụ `UserSessions`, bạn đã có một use case rất thực tế để bắt đầu: tạo table, dùng `userId + sessionId`, bật TTL, dùng On-Demand, cấp IAM least privilege, rồi thực hành `put-item`, `get-item`, `query`. Đây là nền tảng đủ tốt để bạn tiếp tục học các pattern nâng cao như single-table design, GSI optimization, Streams + Lambda và Global Tables.
