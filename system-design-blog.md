# System Design là gì? Tại sao AI càng phát triển thì kỹ năng này càng quan trọng?

## "AI viết code giỏi rồi, vậy lập trình viên cần học gì tiếp theo?"

Nếu vài năm trước, phỏng vấn kỹ sư phần mềm chủ yếu xoay quanh thuật toán và code, thì hiện nay mọi thứ đang thay đổi rất nhanh.

Với sự xuất hiện của ChatGPT, Claude, Copilot và các AI Agent, việc viết code không còn là kỹ năng tạo ra sự khác biệt lớn như trước nữa. Một đoạn CRUD API, một màn hình React hay một Lambda đơn giản giờ đây AI có thể tạo ra trong vài giây.

Nhưng có một thứ AI chưa thể thay thế hoàn toàn:

**Khả năng thiết kế hệ thống.**

Đó là lý do vì sao các công ty ngày càng chú trọng đến System Design trong phỏng vấn cũng như trong công việc thực tế.

---

# System Design là gì?

Hiểu đơn giản, System Design là cách chúng ta thiết kế một hệ thống phần mềm để đáp ứng các yêu cầu như:

* Phục vụ hàng triệu người dùng
* Đảm bảo hiệu năng tốt
* Dễ mở rộng trong tương lai
* Hạn chế tối đa việc hệ thống bị sập

Ví dụ:

Khi bạn mở Shopee, TikTok hay Facebook, phía sau không phải chỉ là một server duy nhất.

Đó là cả một hệ thống gồm:

* DNS
* Load Balancer
* Web Server
* Database
* Cache
* Message Queue
* CDN

và rất nhiều thành phần khác phối hợp với nhau.

System Design chính là việc hiểu cách những thành phần đó hoạt động và kết nối với nhau.

---

# Mọi hệ thống lớn đều bắt đầu từ một server nhỏ

Hãy tưởng tượng bạn vừa xây dựng một website bán hàng.

Ban đầu chỉ có vài chục khách truy cập mỗi ngày.

Kiến trúc lúc này rất đơn giản:

```text
User
   |
Internet
   |
Server
```

Server này chứa:

* Website
* API
* Database

Tất cả nằm trên cùng một máy.

Khi người dùng truy cập:

```text
shop.demo.com
```

Trình duyệt sẽ:

1. Hỏi DNS để tìm IP của server
2. Gửi request đến server
3. Server trả dữ liệu về

Mọi thứ hoạt động rất ổn.

---

# Chuyện gì xảy ra khi có 100.000 người dùng?

Vấn đề bắt đầu xuất hiện khi website phát triển.

Từ vài chục người dùng mỗi ngày tăng lên:

* 10.000 user
* 100.000 user
* 1 triệu user

Lúc này server bắt đầu gặp vấn đề:

* CPU tăng cao
* RAM đầy
* Database chậm
* Website phản hồi chậm

Một chiếc xe máy có thể chở 2 người rất tốt.

Nhưng nếu bắt nó chở 20 người thì chắc chắn sẽ gặp vấn đề.

Server cũng tương tự.

---

# Hai cách để tăng khả năng chịu tải

## Cách 1: Nâng cấp server (Scale Up)

Ví dụ:

```text
4 CPU -> 8 CPU
16GB RAM -> 64GB RAM
```

Ưu điểm:

* Nhanh
* Dễ thực hiện

Nhược điểm:

* Chi phí tăng nhanh
* Luôn có giới hạn phần cứng

Sẽ đến lúc bạn không thể nâng cấp thêm nữa.

---

## Cách 2: Thêm nhiều server (Scale Out)

Thay vì:

```text
1 Server
```

Bạn chuyển thành:

```text
Server 1
Server 2
Server 3
```

Lúc này câu hỏi xuất hiện:

**User sẽ truy cập vào server nào?**

---

# Load Balancer – Người điều phối giao thông của hệ thống

Để giải quyết vấn đề trên, chúng ta sử dụng Load Balancer.

```text
User
   |
Load Balancer
   |
-------------------
|        |        |
S1       S2       S3
```

Nhiệm vụ của Load Balancer là:

* Chia đều traffic
* Theo dõi tình trạng server
* Tự động bỏ qua server bị lỗi

Ví dụ:

Request 1 → Server 1

Request 2 → Server 2

Request 3 → Server 3

Nhờ vậy hệ thống có thể phục vụ nhiều người dùng hơn.

---

# SQL và NoSQL – Chọn database nào?

Đây là câu hỏi gần như luôn xuất hiện trong System Design.

## SQL

Ví dụ:

* MySQL
* PostgreSQL
* Oracle

Dữ liệu được lưu dưới dạng bảng.

Rất phù hợp với:

* Đơn hàng
* Thanh toán
* Ngân hàng
* Kế toán

Điểm mạnh:

* Dữ liệu chặt chẽ
* Hỗ trợ transaction
* Đảm bảo tính chính xác

---

## NoSQL

Ví dụ:

* MongoDB
* Redis
* Cassandra

Phù hợp khi:

* Dữ liệu lớn
* Cần tốc độ cao
* Cấu trúc dữ liệu linh hoạt

Ví dụ nổi tiếng nhất là Redis.

Redis thường được dùng làm cache để giảm tải cho database.

---

# Single Point of Failure – Kẻ thù của mọi hệ thống

Đây là khái niệm cực kỳ quan trọng.

Ví dụ:

```text
API Server
     |
Database
```

Nếu database duy nhất này bị lỗi:

```text
Database Down
```

Toàn bộ hệ thống ngừng hoạt động.

Database lúc này được gọi là:

**Single Point of Failure (SPOF)**

---

# Cách các công ty lớn tránh SPOF

Thay vì chỉ có một thành phần:

```text
1 Database
```

Họ xây dựng:

```text
Primary Database
       |
Replica Database
```

Hoặc:

```text
Load Balancer A
Load Balancer B
```

Nếu một thành phần gặp sự cố, thành phần còn lại sẽ tiếp tục hoạt động.

Đó gọi là Redundancy (dự phòng).

---

# Vì sao kỹ sư phần mềm nên học System Design?

Ngay cả khi bạn không phải Tech Lead hay Architect, việc hiểu System Design vẫn mang lại rất nhiều lợi ích:

### Hiểu hệ thống mình đang làm

Bạn biết:

* Request đi như thế nào
* Dữ liệu được lưu ở đâu
* Cache hoạt động ra sao

---

### Debug nhanh hơn

Khi website chậm, bạn sẽ biết cần kiểm tra:

* Load Balancer
* EC2
* Database
* Redis

thay vì đoán mò.

---

### Phỏng vấn dễ hơn

Hầu hết các vị trí:

* Senior Engineer
* Lead Engineer
* Staff Engineer

đều có vòng System Design.

---

### Chuẩn bị cho tương lai AI

AI có thể giúp bạn viết code.

Nhưng AI không thể tự quyết định:

* Dùng SQL hay NoSQL?
* Dùng Cache ở đâu?
* Scale hệ thống thế nào?
* Chấp nhận trade-off nào?

Đó vẫn là công việc của kỹ sư phần mềm.

---

# Kết luận

AI đang thay đổi cách chúng ta viết code.

Nhưng điều đó không làm kỹ sư phần mềm kém giá trị hơn.

Ngược lại, nó khiến khả năng hiểu hệ thống, thiết kế kiến trúc và đưa ra quyết định kỹ thuật trở nên quan trọng hơn bao giờ hết.

Nếu bạn muốn phát triển từ Junior lên Senior hoặc Tech Lead, System Design là một trong những kỹ năng đáng đầu tư nhất hiện nay.

Bởi vì cuối cùng, thứ tạo ra giá trị không phải là người viết được nhiều code nhất.

Mà là người hiểu được hệ thống đang vận hành như thế nào.
