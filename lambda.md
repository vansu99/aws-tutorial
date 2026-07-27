# AWS Lambda: Tính năng, Giới hạn và Khi nào nên sử dụng cho kỳ thi SAA

Bạn cần tạo một thumbnail mỗi khi người dùng tải lên một hình ảnh. Công việc này diễn ra theo từng đợt: đôi khi có vài nghìn hình ảnh được tải lên trong một phút, đôi khi lại không có gì trong suốt một giờ.

Theo cách cũ, bạn dựng một EC2 instance chạy liên tục để chờ công việc: bạn vẫn phải trả tiền cho cả những giờ server không làm gì, bạn phải tự vá lỗi và cập nhật hệ điều hành, và khi một lượng lớn hình ảnh được tải lên cùng lúc, bạn phải nhanh chóng bổ sung thêm tài nguyên để xử lý kịp thời.

Một câu hỏi tự nhiên được đặt ra: tại sao phải giữ một server chạy 24/7 cho một công việc chỉ chạy không thường xuyên?

Sẽ tốt hơn nhiều nếu chỉ cần đóng gói code xử lý hình ảnh, giao nó cho AWS và nói:

> “Bất cứ khi nào có hình ảnh mới, hãy chạy đoạn code này; khi không có hình ảnh nào, tôi không phải trả tiền.”

Đó chính xác là điều AWS Lambda mang lại.

Bài viết này là một bản đồ tổng quan về Lambda dành cho kỳ thi SAA. Chúng ta sẽ đi qua sáu phần:

1. Lambda là gì và nó mang lại điều gì.
2. Các giới hạn thường được kiểm tra trong kỳ thi.
3. Cách quản lý concurrency và throttling.
4. SnapStart để xử lý cold start.
5. Lambda@Edge và CloudFront Functions chạy ở edge.
6. Lambda khi được đặt bên trong VPC.

**Lưu ý:** Đây là phần tổng quan nhằm xây dựng cách hiểu tổng thể và giúp bạn nhanh chóng nhận diện đáp án trong kỳ thi. SAA hiếm khi hỏi về cú pháp của function; thay vào đó, kỳ thi hỏi về đặc điểm: giới hạn nào là giới hạn cứng, cơ chế nào phù hợp với vấn đề nào và ranh giới giữa những lựa chọn gần như giống nhau, ví dụ Lambda@Edge so với CloudFront Functions.

Bài viết này tập trung chính xác vào những điểm đó.

---

# 1. AWS Lambda là gì?

## 1.1. Mô hình Serverless và Function as a Service

AWS Lambda là một dịch vụ compute serverless cho phép bạn chạy code mà không cần dựng hoặc quản lý bất kỳ server nào.

Bạn không cần:

* Tạo EC2 instance.
* Cài đặt hệ điều hành.
* Vá lỗi hệ điều hành.
* Lo về scaling.

Bạn chỉ cần tải lên một function — một đoạn code thực hiện một nhiệm vụ — và khai báo event nào sẽ kích hoạt nó.

Vì đơn vị triển khai là một function nên mô hình này còn được gọi là:

**Function as a Service — FaaS.**

“Serverless” không có nghĩa là không có server.

Các server thực tế vẫn chạy bên dưới, nhưng AWS hoàn toàn ẩn chúng khỏi bạn. Việc cấp phát, vận hành và scale server là trách nhiệm của AWS.

Điều này khác đáng kể so với EC2, nơi bạn sở hữu và chịu trách nhiệm cho từng instance.

Lambda hoạt động theo mô hình **event-driven**:

* Code chỉ chạy khi có event kích hoạt.
* Code dừng khi xử lý hoàn tất.
* Không có event thì không có gì chạy.
* Khi không chạy, bạn không bị tính phí.

---

## 1.2. Các lợi ích cốt lõi

Có bốn điểm khiến Lambda trở nên hấp dẫn, và đây cũng là bốn điểm thường xuất hiện trong kỳ thi.

### Không cần quản lý server

Bạn chỉ cần quan tâm đến code.

Infrastructure, hệ điều hành và runtime là trách nhiệm của AWS.

### Tự động scale

Khi nhiều event đến cùng lúc, Lambda tạo thêm nhiều instance chạy song song để xử lý.

Khi công việc giảm xuống, Lambda tự scale trở lại về 0.

Bạn không cần cấu hình Auto Scaling Group như khi sử dụng EC2.

### Chỉ trả tiền cho những gì thực sự sử dụng

Bạn bị tính phí dựa trên:

* Số lần function được invoke.
* Thời gian thực tế function chạy.

Khi idle, chi phí bằng 0.

Đây là điểm khác biệt lớn nhất so với EC2, vì EC2 vẫn tính phí ngay cả khi máy không làm gì.

### High Availability được tích hợp sẵn

Lambda chạy phân tán trên nhiều Availability Zone trong một Region mà bạn không cần làm gì thêm.

---

## 1.3. Ngôn ngữ, Runtime và Container Image

Lambda chạy code của bạn bên trong một managed runtime.

AWS cung cấp runtime cho các ngôn ngữ phổ biến:

* Node.js
* Python
* Java
* .NET / C#
* Go
* Ruby

Khi bạn cần một ngôn ngữ hoặc phiên bản không có trong danh sách, có hai lựa chọn.

### Custom runtime

Bạn có thể cung cấp runtime của riêng mình, ví dụ:

* Rust
* Phiên bản cũ của một ngôn ngữ

Thông qua Lambda Runtime API.

### Container image

Bạn có thể đóng gói function thành một standard container image có kích thước tối đa 10 GB.

Đây là lựa chọn dành cho những function có nhiều thư viện nặng, chẳng hạn các dependency cho machine learning.

Một điểm dễ gây nhầm lẫn trong kỳ thi:

> Container image chạy trên Lambda vẫn là Lambda.

Nó vẫn:

* Event-driven.
* Có các giới hạn của Lambda.

Nó không trở thành container chạy theo cách ECS hoặc Fargate hoạt động.

---

## 1.4. Điều gì có thể trigger Lambda?

Sức mạnh thực sự của Lambda nằm ở việc nó tích hợp với gần như mọi dịch vụ AWS khác và có thể chạy khi các dịch vụ đó phát ra event.

Một số nguồn event thường gặp trong kỳ thi:

### API Gateway

Xây dựng REST API hoặc HTTP API dạng serverless, trong đó mỗi request gọi một function.

### S3

Chạy function khi một object được tạo hoặc xóa.

Ví dụ:

* Tạo thumbnail khi có ảnh mới được upload.

### SQS, SNS

* Xử lý message từ queue.
* Nhận notification.

### DynamoDB Streams, Kinesis

Phản ứng với:

* Từng thay đổi dữ liệu.
* Từng record trong stream.

### EventBridge

Chạy function:

* Theo lịch.
* Khi có system event.

Có thể dùng thay thế cron.

### ALB

Application Load Balancer có thể trỏ trực tiếp đến Lambda như một HTTP backend mà không cần EC2.

Để gọi các dịch vụ khác, mỗi Lambda function gắn với một IAM execution role chứa chính xác những permission mà nó cần.

Ví dụ:

* Permission ghi dữ liệu vào DynamoDB.

---

## 1.5. Mô hình tính phí

Lambda tính phí theo hai yếu tố.

Quan trọng nhất:

> Không có event thì không có phí thực thi.

### Theo số lần invocation

Phí được tính dựa trên tổng số lần function được kích hoạt.

### Theo thời gian chạy

Chi phí được tính theo GB-second.

Nghĩa là:

**Memory được cấp × thời gian function chạy**

Cấp nhiều memory hơn làm mỗi giây đắt hơn, nhưng function cũng nhận được nhiều CPU hơn và thường chạy nhanh hơn.

AWS cũng cung cấp một free tier hàng tháng khá lớn:

* 1 triệu lượt invocation.
* 400.000 GB-seconds.

Vì vậy, nhiều ứng dụng nhỏ gần như không tốn chi phí đáng kể cho Lambda.

Đối với kỳ thi, cần nhớ ý chính:

> Lambda là lựa chọn kinh tế cho workload có traffic theo đợt hoặc không đều.

Trong khi workload chạy liên tục với tải cao và ổn định có thể rẻ hơn khi sử dụng:

* EC2.
* Fargate.

---

# 2. Các giới hạn của Lambda

Đây là khu vực mà kỳ thi SAA hỏi nhiều nhất.

Nhiều câu hỏi thực chất là:

> Workload này có vượt quá giới hạn của Lambda không?

Nếu có, hãy chọn dịch vụ khác.

Các con số quan trọng:

| Giới hạn                    |                     Giá trị | Ghi chú cho kỳ thi                                   |
| --------------------------- | --------------------------: | ---------------------------------------------------- |
| Memory                      |            128 MB đến 10 GB | CPU tăng theo memory                                 |
| Thời gian chạy tối đa       |                     15 phút | Hard limit, không tăng được                          |
| Temporary `/tmp` storage    |            512 MB đến 10 GB | Scratch disk, dữ liệu mất khi environment bị thu hồi |
| Environment variables       |                 Tối đa 4 KB | Tổng kích thước tất cả biến môi trường               |
| Deployment package dạng ZIP | 50 MB nén / 250 MB giải nén | Bao gồm cả layers                                    |
| Container image             |                       10 GB | Dùng khi package vượt giới hạn 250 MB                |
| Synchronous payload         |                        6 MB | Kích thước request/response tối đa                   |
| Asynchronous payload        |                      256 KB | Cho event-driven invocation như SNS, S3              |

Một số điểm thường trở thành bẫy trong kỳ thi.

### 15 phút là giới hạn cứng

Nếu một task chạy lâu hơn 15 phút, ví dụ xử lý dữ liệu trong nhiều giờ, Lambda là đáp án sai.

Nên nghĩ đến:

* EC2.
* Fargate.
* AWS Batch.

### CPU không được cấu hình riêng

Bạn không chọn trực tiếp số CPU.

CPU được cấp tỷ lệ thuận với memory.

Nếu một function cần nhiều compute và chạy chậm, giải pháp thường là tăng memory để đồng thời nhận thêm CPU.

### `/tmp` chỉ là scratch space

Đây không phải storage lâu dài.

Nếu cần:

* Durable storage.
* Chia sẻ dữ liệu giữa các function.

Hãy dùng:

* S3.
* EFS.

### Payload 6 MB

Nếu cần truyền dữ liệu lớn hơn, hãy lưu dữ liệu trong S3 và chỉ truyền reference, ví dụ đường dẫn đến object.

---

# 3. Concurrency và Throttling

## 3.1. Cold Start và Warm Start

Để hiểu concurrency, trước tiên cần hiểu một Lambda instance hoạt động như thế nào.

Mỗi khi Lambda cần xử lý một event, nó tạo một execution environment.

Quá trình này gồm:

1. Download code.
2. Khởi tạo runtime.
3. Chạy phần initialization nằm ngoài main handler.
4. Có thể mở database connection.
5. Load library.

Quá trình tạo mới từ đầu này được gọi là:

**Cold Start.**

Cold start làm tăng thêm thời gian xử lý.

Nó đặc biệt đáng chú ý với runtime nặng như Java.

Sau khi Lambda chạy xong, AWS không hủy environment ngay lập tức.

AWS giữ environment đó lại trong một khoảng thời gian.

Nếu event mới đến trong khi environment vẫn còn tồn tại, Lambda sẽ reuse environment đó.

Đây gọi là:

**Warm Start.**

Warm start không cần chạy lại toàn bộ bước initialization nên nhanh hơn đáng kể.

Cold start là cái giá phải trả khi một execution environment hoàn toàn mới được tạo.

---

## 3.2. Concurrency là gì và giới hạn mặc định

Concurrency là số lượng function instance đang xử lý event cùng một lúc.

Ví dụ:

* 100 event đến đồng thời.
* Mỗi event mất 1 giây để xử lý.

Bạn cần 100 instance chạy song song.

Concurrency lúc này là:

**100**

Mỗi Region có quota concurrency mặc định được chia sẻ cho tất cả function:

**1.000 concurrent instances.**

Đây là soft limit.

Bạn có thể yêu cầu AWS tăng quota.

Khi tổng số concurrent instance đạt giới hạn, invocation vượt quá giới hạn sẽ bị throttle.

---

## 3.3. Reserved Concurrency

Reserved concurrency nghĩa là dành riêng một phần quota cho một function cụ thể.

Nó đồng thời có hai tác dụng.

### Bảo đảm

Function đó luôn có số lượng instance tương ứng sẵn trong quota.

Các function khác không thể sử dụng phần quota đó.

### Giới hạn tối đa

Con số đó đồng thời cũng là maximum concurrency mà function được phép sử dụng.

Function không thể vượt quá reserved concurrency của nó.

Reserved concurrency hữu ích trong hai tình huống thường xuất hiện trong kỳ thi.

### Bảo vệ một function quan trọng

Nếu một function khác sử dụng gần hết concurrency của Region, function quan trọng vẫn có phần quota riêng.

### Giới hạn áp lực lên downstream resource

Ví dụ database chỉ hỗ trợ số lượng connection hạn chế.

Bạn không muốn Lambda mở hàng nghìn database connection cùng lúc.

Reserved concurrency có thể giới hạn số Lambda chạy song song để bảo vệ database.

---

## 3.4. Provisioned Concurrency

Provisioned concurrency giải quyết vấn đề:

**Cold Start.**

Bạn yêu cầu Lambda khởi tạo trước một số execution environment và giữ chúng luôn warm.

Khi event đến, không cần tạo environment từ đầu.

Do đó không có cold start latency.

Provisioned concurrency phù hợp với application nhạy cảm về latency và có traffic có thể dự đoán được.

Ví dụ:

* API hướng đến người dùng cần response time ổn định.
* Chuẩn bị trước cho một traffic spike đã được lên lịch.

Không giống reserved concurrency, provisioned concurrency có thêm chi phí.

Lý do:

AWS phải duy trì các execution environment luôn sẵn sàng.

Cách phân biệt nhanh:

**Reserved concurrency**

→ Quan tâm đến số lượng instance.

**Provisioned concurrency**

→ Quan tâm đến latency.

Nếu đề hỏi:

> Loại bỏ cold start cho workload nhạy cảm với latency.

Đáp án thường là:

**Provisioned Concurrency.**

---

## 3.5. Throttling

Khi số concurrent instance vượt quá giới hạn, Lambda thực hiện throttling.

Điều này có thể xảy ra khi vượt:

* Shared quota của Region.
* Reserved concurrency của function.

Lambda từ chối một số invocation và trả về:

**429 TooManyRequestsException**

Điều xảy ra tiếp theo tùy thuộc vào kiểu invocation.

### Synchronous invocation

Ví dụ thông qua API Gateway.

Error được trả trực tiếp cho caller.

Caller quyết định có retry hay không.

### Asynchronous invocation

Ví dụ từ:

* S3.
* SNS.

Lambda tự retry trong một khoảng thời gian.

Do đó những đợt tải tăng ngắn thường có thể được hấp thụ mà không làm mất event.

Nếu thường xuyên gặp throttling, bạn có thể:

* Yêu cầu tăng Region concurrency quota.
* Thiết lập reserved concurrency hợp lý.

---

# 4. SnapStart

Như đã đề cập ở phần cold start, các runtime nặng như Java có thể mất khá nhiều thời gian để initialize.

Đôi khi mất vài giây.

Điều này có thể gây trải nghiệm không tốt cho API cần phản hồi nhanh.

Một giải pháp là:

**Provisioned Concurrency.**

Nhưng nó tốn chi phí để giữ environment luôn warm.

Lambda SnapStart là một giải pháp khác.

Đối với Java, SnapStart miễn phí.

Ý tưởng của SnapStart là:

Thay vì initialize environment từ đầu trong mỗi cold start, Lambda:

1. Chạy initialization một lần.
2. Chụp snapshot của environment đã initialize hoàn chỉnh.
3. Cache snapshot đó.

Các invocation sau chỉ cần restore từ snapshot đã có thay vì build lại toàn bộ environment.

Điều này giúp cold start nhanh hơn đáng kể.

Các điểm cần nhớ:

### Runtime hỗ trợ

* Java 11 trở lên.
* Python 3.12 trở lên.
* .NET 8 trở lên.

Không hỗ trợ:

* Node.js.
* Ruby.
* Custom runtime.
* Container image.

### Không thể dùng cùng Provisioned Concurrency

SnapStart và Provisioned Concurrency loại trừ lẫn nhau.

Bạn phải chọn một.

SnapStart cũng không dùng cùng:

* EFS.
* `/tmp` lớn hơn 512 MB.

### Chi phí

Đối với Java:

* SnapStart miễn phí.

Đối với Python và .NET:

* Có thêm chi phí cho việc cache và restore snapshot.

Trong kỳ thi, keyword nhận biết SnapStart là:

> Giảm cold start cho Java function mà không trả phí để giữ function luôn warm như Provisioned Concurrency.

---

# 5. Lambda@Edge và CloudFront Functions

## 5.1. Tại sao chạy code ở Edge?

CloudFront là CDN của AWS.

Nó cache nội dung tại những địa điểm gần người dùng nhằm giảm latency.

Đôi khi bạn muốn chạy một phần logic ngay tại edge trước khi request đến origin.

Ví dụ:

* Rewrite URL.
* Kiểm tra authentication token.
* Thêm hoặc xóa HTTP header.
* Phục vụ nội dung khác nhau dựa trên loại thiết bị.

Việc chạy logic tại edge cho phép xử lý gần người dùng hơn thay vì request phải đi đến Region rồi quay lại.

AWS có hai công cụ:

* CloudFront Functions.
* Lambda@Edge.

Kỳ thi SAA rất thường yêu cầu phân biệt hai dịch vụ này.

---

## 5.2. CloudFront Functions

CloudFront Functions là các JavaScript function cực kỳ nhẹ chạy trực tiếp tại edge location.

Execution time dưới:

**1 millisecond.**

Nó được thiết kế cho volume cực lớn, có thể đạt hàng triệu request mỗi giây.

Phù hợp với những tác vụ rất ngắn:

* Thay đổi HTTP header.
* Rewrite URL.
* Redirect URL.
* Normalize cache key.
* Kiểm tra token hoặc simple authentication.

Đổi lại cho tốc độ cao và chi phí thấp, nó có một số hạn chế:

* Chỉ hỗ trợ JavaScript.
* Rất ít memory.
* Không được gọi network.
* Chỉ trigger tại hai thời điểm.

Hai trigger point:

* Viewer request.
* Viewer response.

Viewer request xảy ra trước khi CloudFront kiểm tra cache.

Viewer response xảy ra trước khi response được trả về người dùng.

---

## 5.3. Lambda@Edge

Lambda@Edge là Lambda function chạy tại Regional Edge Cache.

Hỗ trợ:

* Node.js.
* Python.

Nó nặng hơn và mạnh hơn CloudFront Functions.

Nó có:

* Thời gian chạy lâu hơn.
* Nhiều memory hơn.
* Có thể thực hiện network call.

Điểm quan trọng trong kỳ thi:

Lambda@Edge có thể trigger tại bốn giai đoạn trong vòng đời CloudFront:

* Viewer request.
* Viewer response.
* Origin request.
* Origin response.

Trong đó:

**Origin request**

→ trước khi request đến origin.

**Origin response**

→ sau khi origin trả response.

Do có thể can thiệp vào origin phase và gọi network, Lambda@Edge phù hợp với logic phức tạp hơn.

Ví dụ:

* Authentication và authorization nâng cao.
* Routing có điều kiện tới các origin khác nhau.
* Transform response cần truy vấn dữ liệu bên ngoài.

---

## 5.4. So sánh và cách chọn

| Tiêu chí       | CloudFront Functions                     | Lambda@Edge                                        |
| -------------- | ---------------------------------------- | -------------------------------------------------- |
| Ngôn ngữ       | Chỉ JavaScript                           | Node.js, Python                                    |
| Nơi chạy       | Edge Locations                           | Regional Edge Caches                               |
| Trigger point  | Viewer request/response                  | Viewer + origin request/response                   |
| Thời gian chạy | Dưới 1 millisecond                       | Lên đến vài giây                                   |
| Network call   | Không                                    | Có                                                 |
| Scale          | Hàng triệu request/giây                  | Thấp hơn nhiều                                     |
| Phù hợp        | Header, redirect, cache key, simple auth | Logic phức tạp, gọi external service, origin phase |

Cách chọn nhanh trong kỳ thi:

Nếu cần:

* Logic cực nhẹ.
* Cực nhanh.
* Quy mô cực lớn.
* Chỉ xử lý viewer request/response.

Chọn:

**CloudFront Functions.**

Nếu cần:

* Network call.
* Can thiệp origin phase.
* Logic nặng hơn.

Chọn:

**Lambda@Edge.**

---

# 6. Lambda bên trong VPC

## 6.1. Mặc định Lambda nằm ngoài VPC của bạn

Theo mặc định, Lambda function chạy trong AWS-managed VPC, tách biệt khỏi VPC của bạn.

Một hệ quả quan trọng:

Ở chế độ mặc định, function có thể truy cập Internet, ví dụ gọi public API.

Nhưng nó không thể truy cập các private resource nằm trong VPC của bạn.

Ví dụ:

* RDS database trong private subnet.
* ElastiCache cluster trong private subnet.

---

## 6.2. Gắn Lambda vào VPC

Khi function cần truy cập các private resource, bạn cấu hình Lambda chạy bên trong VPC.

Bạn chỉ định:

* Subnet.
* Security Group.

Sau đó Lambda sử dụng ENI trong VPC để kết nối với các resource.

Ngày nay AWS sử dụng:

**Hyperplane ENI.**

Đây là các shared ENI được tạo trước ở cấp function configuration thay vì tạo một ENI cho mỗi invocation.

Vì vậy, việc gắn Lambda vào VPC hiện nay không còn gây ra cold start lớn như trước đây.

Đối với kỳ thi, chỉ cần nhớ:

> Lambda giao tiếp với resource trong VPC thông qua ENI.

---

## 6.3. Vào VPC đồng nghĩa mất default Internet route

Đây là một trong những bẫy kinh điển nhất của Lambda trong kỳ thi SAA.

Khi gắn function vào VPC:

**Lambda mất default Internet access.**

Việc đặt function trong public subnet cũng không giải quyết được vấn đề.

Lý do:

Lambda function không có public IP.

Nếu function trong VPC cần:

* Truy cập private resource.
* Đồng thời truy cập Internet hoặc AWS public API.

Kiến trúc đúng là:

1. Đặt Lambda trong private subnet.
2. Private subnet có route tới NAT Gateway.
3. NAT Gateway nằm trong public subnet.
4. NAT Gateway xử lý outbound Internet traffic.

---

## 6.4. Gọi S3 và DynamoDB mà không cần NAT

Nếu Lambda trong VPC chỉ cần gọi:

* S3.
* DynamoDB.

Bạn không nhất thiết phải tạo NAT Gateway.

NAT Gateway có chi phí.

S3 và DynamoDB hỗ trợ:

**Gateway VPC Endpoint.**

Bạn thêm route vào route table để traffic đến S3 hoặc DynamoDB đi qua endpoint này.

Traffic:

* Ở hoàn toàn trong AWS network.
* Không đi qua Internet.
* Không phát sinh NAT charge.

Nếu đề thi hỏi:

> Lambda trong VPC cần truy cập S3 với chi phí thấp nhất.

Đáp án:

**Gateway VPC Endpoint.**

Không phải:

**NAT Gateway.**

---

# Kết luận

Quay lại ví dụ thumbnail ở đầu bài.

Bạn không cần giữ một server chạy 24/7 chỉ để chờ một workload xảy ra không thường xuyên.

Bạn chỉ cần:

1. Đóng gói code.
2. Kết nối code với event “có hình ảnh mới trên S3”.
3. Để AWS xử lý phần còn lại.

AWS Lambda sẽ:

* Scale theo tải.
* Tính phí theo mức sử dụng.
* Cung cấp High Availability có sẵn.

Đó chính là tinh thần của Lambda.

Phần còn lại của bài viết là tập hợp các đặc điểm bạn cần biết để sử dụng Lambda đúng trường hợp và chọn đáp án đúng trong kỳ thi.

---

# Các keyword thường gặp trong kỳ thi

| Keyword trong câu hỏi                                                              | Hướng trả lời                                      |
| ---------------------------------------------------------------------------------- | -------------------------------------------------- |
| Bursty, event-driven, không muốn quản lý server, pay per use                       | Lambda                                             |
| Task chạy lâu hơn 15 phút                                                          | Không dùng Lambda — chọn EC2 / Fargate / AWS Batch |
| Bảo đảm và giới hạn số instance của function                                       | Reserved Concurrency                               |
| Loại bỏ cold start cho workload nhạy cảm với latency và chấp nhận trả phí giữ warm | Provisioned Concurrency                            |
| Giảm cold start cho Java function mà không trả phí giữ warm                        | SnapStart                                          |
| Logic cực nhẹ, cực nhanh để xử lý header/URL ở edge với quy mô rất lớn             | CloudFront Functions                               |
| Logic ở edge cần network call hoặc can thiệp origin phase                          | Lambda@Edge                                        |
| Lambda trong VPC cần Internet access                                               | Private subnet + NAT Gateway                       |
| Lambda trong VPC cần gọi S3/DynamoDB với chi phí thấp nhất                         | Gateway VPC Endpoint                               |

---

# Những điểm cần nhớ cho kỳ thi

Lambda là:

* Serverless.
* Event-driven.
* Pay-per-use.

Phù hợp cho workload:

* Bursty.
* Không đều.

Chi phí bằng 0 khi idle.

Các giới hạn cứng cần nhớ:

* Memory tối đa: 10 GB.
* CPU tăng theo memory.
* Thời gian chạy tối đa: 15 phút.
* Synchronous payload: 6 MB.
* Deployment package: 50 MB nén / 250 MB giải nén.
* Container image: 10 GB.

### Reserved và Provisioned

**Reserved Concurrency**

→ Liên quan đến “bao nhiêu instance”.

Nó vừa là:

* Guarantee.
* Ceiling.

**Provisioned Concurrency**

→ Liên quan đến latency.

Execution environment được giữ warm để tránh cold start và có tính phí.

### SnapStart

Giảm cold start cho:

* Java.
* Python.
* .NET.

Không thể dùng cùng Provisioned Concurrency.

### CloudFront Functions và Lambda@Edge

**CloudFront Functions**

* Nhẹ.
* Nhanh.
* JavaScript.
* Viewer phase.

**Lambda@Edge**

* Nặng hơn.
* Có thể gọi network.
* Node.js hoặc Python.
* Có thể can thiệp origin phase.

### Lambda trong VPC

Lambda khi vào VPC sẽ mất default Internet access.

Muốn truy cập Internet:

**Private Subnet → NAT Gateway**

Nếu chỉ gọi S3 hoặc DynamoDB:

**Gateway VPC Endpoint**

sẽ tiết kiệm hơn vì không cần NAT.
