### Pricing and Support

- **Free Tier**: Try AWS services free for 12 months.
- **Pricing Calculator**: Estimate monthly costs.
- **Support Plans**: From basic to enterprise-level.

### Abstracted Services / Managed Services

Thường sẽ dẫn đến các dịch vụ như:
- DynamoDB (cơ sở dữ liệu NoSQL)
- Aurora (cơ sở dữ liệu quan hệ)
- S3 (lưu trữ dữ liệu)
- Lambda (tài nguyên điện toán không máy chủ)
Lợi ích: Giảm thiểu công việc vận hành, dễ dàng đáp ứng các yêu cầu.

### Fault Tolerant / Disaster Recovery (Chịu lỗi / Khả năng khôi phục)
- Sử dụng nhiều Availability Zone (AZ)
- Có khả năng chuyển đổi dự phòng (failover) sang vùng khác (Region)

### Highly Available
- Triển khai ứng dụng hoặc cơ sở dữ liệu trên nhiều AZ
- Ví dụ: EC2 kết hợp với Load Balancer, cơ sở dữ liệu cấu hình multi-AZ deployment

### Long Term Storage
- Dành cho dữ liệu ít truy cập, thường dùng các giải pháp như:
  + Glacier hoặc Glacier Deep Archive
  + S3 Intelligent-Tiering hoặc S3 Infrequent Access (nếu cần thời gian truy xuất nhanh hơn)

### Real-Time Processing
- Thường liên quan đến việc tích hợp với Amazon Kinesis hoặc các dịch vụ tương tự.

### Scalable 
- Dùng Auto Scaling cho EC2 hoặc Read Replicas cho cơ sở dữ liệu để đáp ứng nhu cầu tải cao.

### Elastic 
- Môi trường có khả năng mở rộng hoặc thu hẹp tự động tùy theo nhu cầu sử dụng thực tế (on-demand).

### Virtual Private Cloud (VPC) và Networking
- VPC có thể mở rộng qua nhiều Availability Zones trong một region và có thể chứa nhiều public subnet và private subnet.
- Một public subnet chứa route đến Internet Gateway (cần được cấu hình thủ công).
- Một private subnet thường không có quyền truy cập internet. Nếu cần truy cập internet, bạn phải thiết lập NAT Gateway hoặc NAT Instance và cấu hình whitelist cho request truy cập.
- Nếu bạn cần truy cập SSH từ internet đến một tài nguyên trong private subnet, bạn cần cấu hình một Bastion Host trong public subnet, đồng thời điều chỉnh Security Groups và Network Access Control Lists (NACLs) để cho phép chuyển tiếp request trên port 22.

### Disaster Recovery Plans
- **Backup and Restore** – có RTO (Recovery Time Objective) và RPO (Recovery Point Objective) cao nhất nhưng chi phí thấp nhất.
- **Pilot Light** – lưu trữ các hệ thống quan trọng ở trạng thái tối thiểu làm mẫu, từ đó có thể mở rộng nhanh chóng khi xảy ra issue.
- **Warm Standby** – duy trì phiên bản sao của các hệ thống quan trọng luôn chạy sẵn, sẵn sàng tiếp quản khi cần.
- **Multi-Site** – có RTO và RPO thấp nhất nhưng chi phí cao nhất.

### Route Tables
- Xác định cách request mạng sẽ di chuyển trong VPC.
- Mỗi route bao gồm đích đến (destination) và mục tiêu (target), ví dụ: 0.0.0.0/0 (CIDR) và igw-1234567890 (Internet Gateway).
- CIDR block đại diện cho tất cả địa chỉ IPv4 trong subnet và chuyển tiếp chúng đến Internet Gateway.
- Route tables được gắn vào subnet cụ thể
- Main route table không thể bị xóa, nhưng bạn có thể thêm, sửa, hoặc xóa route trong đó
- Một subnet chỉ được gắn với một route table, nhưng một route table có thể được gắn với nhiều subnet.
- Route tables cũng có thể gắn với Virtual Private Gateway hoặc Internet Gateway để xác định hướng đi của request vào VPC.
- Mỗi VPC luôn có router ngầm định (implicit router), nơi các route table sẽ được gắn vào.

### Virtual Private Gateway (VPC Gateway)
- Cần thiết khi bạn muốn kết nối VPC của AWS với hệ thống mạng tại chỗ (on-premise).

### Network Access Control Lists (NACLs)
- Hoạt động ở cấp độ subnet, và là stateless
- Có thể cấu hình các rule cho phép và chặn request.
- Mặc định cho phép tất cả request đi và đến trên mọi port.
- Request phản hồi cần được cho phép rõ ràng, không được tự động xử lý như trong Security Groups.

### Security Groups (SGs)
- SG hoạt động ở cấp độ instance (ví dụ: EC2 instance) và có tính chất "stateful" (có ghi nhớ trạng thái kết nối).
- Chỉ định nghĩa các luật "cho phép" (allow), không có luật "chặn" (deny) như ở NACLs.
- cho phép các instance trong cùng một SG giao tiếp với nhau, cho phép toàn bộ lưu lượng đi ra ngoài, và chặn toàn bộ lưu lượng đến từ bên ngoài.
- Có thể gán hoặc gỡ SG khỏi instance bất kỳ lúc nào mà không cần dừng máy (EC2 instance).
- Luôn chỉ định rõ ràng dải IP qua CIDR, không dùng IP đơn lẻ (nếu muốn IP đơn thì dùng /32).

### CIDR (Classless Inter-Domain Routing)
- Là cách xác định dải IP bằng việc dùng subnet mask.
- Được sử dụng rộng rãi trong AWS, như SGs, VPCs, Subnets,...
- Ví dụ: 10.0.0.0/24 → chứa 256 địa chỉ IP (từ 10.0.0.0 đến 10.0.0.255) ; 10.0.0.0/16 → chứa 65,536 IP
- Lên kế hoạch dải IP kỹ lưỡng, tránh bị trùng khi dùng VPC Peering hoặc mở rộng hệ thống.

### VPC Endpoints
- Giúp truy cập dịch vụ AWS từ trong VPC mà không cần truy cập internet công cộng.
- **Gateway Endpoint**:
  + Dành cho S3 và DynamoDB.
  + Tạo một "cổng" trong VPC để truy cập dịch vụ, không cần dùng Internet Gateway hay NAT.
- **Interface Endpoint**:
  + Dùng cho các dịch vụ khác (EC2, SSM, SNS,...).
  + Sử dụng AWS PrivateLink để đảm bảo kết nối riêng tư qua ENI (Elastic Network Interface).
- Luôn sử dụng VPC Endpoint nếu muốn tăng bảo mật bằng cách không cho lưu lượng ra internet mà vẫn gọi được các dịch vụ AWS.

### NAT Gateway & NAT Instance – Kết nối subnet private ra internet
- Khi bạn có subnet riêng (private subnet) và muốn các instance trong đó truy cập internet (ví dụ tải package) mà không muốn chúng bị truy cập từ bên ngoài.
- **NAT Instance**:
  + Tự quản lý (cài đặt, cập nhật, scale), không có auto-scaling mặc định.
  + Dễ cấu hình nhưng tốn công quản lý, ít được khuyến khích.
- **NAT Gateway**:
  + Dịch vụ do AWS quản lý, tự động scale, hiệu suất cao và đáng tin cậy hơn.
  + Khuyến nghị sử dụng trong môi trường production.
- Dùng NAT Gateway thay vì NAT Instance nếu không có nhu cầu đặc biệt cần tự quản lý.
  
### VPC Peering – Kết nối VPC với VPC
- Cho phép hai VPC giao tiếp trực tiếp với nhau thông qua IP.
- Có thể kết nối giữa các tài khoản AWS khác nhau.
- CIDR của 2 VPC không được trùng nhau.
- Kết nối không có tính "transitive": A <– Peering –> B <– Peering –> C ⇒ A không thể giao tiếp với C
- Dùng khi bạn chỉ cần kết nối một vài VPC cụ thể.
- Đảm bảo không trùng CIDR trước khi tạo Peering.

### Transit Gateway – Trung tâm kết nốGắn được 1 EC2 tại 1 thời điểm (không phải ổ đĩa mạng)i nhiều VPC 
- Giải pháp thay thế khi có nhiều VPC cần giao tiếp với nhau (kể cả on-premise).
- Có tính transitive: A → TGW ← B ← TGW → C ⇒ A có thể nói chuyện với C qua Transit Gateway.
- Dùng Transit Gateway thay vì Peering khi có từ 3 VPC trở lên để giảm độ phức tạp kết nối.

### Elastic IP Address (EIP)
- IP tĩnh do AWS cấp phát, có thể di chuyển giữa các instance khác nhau trong cùng region.
- Không gán thì không mất tiền, gán rồi không dùng thì bị tính phí.
- Ví dụ: Dùng EIP cho máy chủ chính để giữ IP không đổi ngay cả khi phải tạo lại EC2 mới.
- Chỉ sử dụng khi bạn thật sự cần một IP cố định, ví dụ để cấu hình DNS công khai hoặc firewall cố định

### Amazon Machine Image (AMI)
- Là mẫu (template) để tạo EC2 instance. AMI chứa:
  + Hệ điều hành (OS)
  + Ứng dụng máy chủ (web server, database, v.v.)
  + Cấu hình phần mềm khác
  + Thông tin quyền khởi chạy
- Loại AMI:
  + EBS-backed: lưu dữ liệu ổ đĩa trong EBS (dữ liệu sẽ giữ lại khi tắt máy)
  + Instance Store-backed: lưu trong ổ đĩa vật lý tạm thời (bị mất khi tắt máy)
  + Ví dụ: Bạn tạo một EC2, cài Apache + PHP, sau đó tạo một custom AMI để tái sử dụng cấu hình cho lần sau.

### Elastic Block Store (EBS)
- Ổ đĩa gắn vào từng EC2 instance (giống như ổ cứng trong máy tính).
- Gắn được 1 EC2 tại 1 thời điểm (không phải ổ đĩa mạng)
- Có thể tạo snapshot để sao lưu
- Root volume (ổ chứa OS) mặc định sẽ bị xóa khi EC2 bị terminate
- Dùng GP3 cho web/app phổ thông. Dùng IO2 khi cần performance cao như database sản xuất (PostgreSQL, Oracle...).
- Dùng AWS Data Lifecycle Manager để tự động tạo snapshot định kỳ cho backup.

### Auto Scaling Policies
- Cơ chế tự động thêm/bớt EC2 instance dựa trên tải hệ thống (CPU, Network, …)
- **Target Tracking**:
  + Bạn chọn một metric (VD: CPU = 50%)
  + Auto Scaling sẽ tự động tạo CloudWatch alarm và scale theo.
  + Có thể định nghĩa thời gian "warm-up" để tránh scale ảo do EC2 mới tạo có CPU cao.
- **Step Scaling**:
  + Cho phép scale theo "step", tùy theo mức độ vi phạm threshold.
  + Ví dụ: Nếu CPU vượt 20% thì thêm 1 instance, nếu vượt 40% thì thêm 3 instance.
- Ví dụ thực tế 1:
  + Desired capacity: 10 EC2
  + CPU tăng từ 50% → 60% → scale out +1 instance
  + Sau cooldown, CPU tăng tiếp đến 70% → scale out +3 instance nữa
- Ví dụ thực tế 2:
  + Trường hợp sử dụng trong Thương mại điện tử
  + Giả sử bạn có một website bán hàng giống Shopee hay Tiki, chạy trên EC2, và có traffic truy cập cao vào buổi trưa hoặc khung giờ Flash Sale 20h tối.
  + Vấn đề:
    + Nếu bạn chạy cố định 10 EC2, ban ngày thì lãng phí tiền, vì ít người truy cập.
    + Nếu bạn chỉ chạy 2 EC2, lúc cao điểm có thể bị sập website.
  + Giải pháp Auto Scaling:
    + Tăng thêm EC2 khi CPU vượt quá 70%
    + Giảm bớt EC2 khi CPU thấp hơn 30%
- Ví dụ cấu hình ***Step Scaling Policy***:
  + Giả sử bạn có:
    + Desired: 10 instances
    + Policy:
      - Nếu CPU tăng 10–20% → Tăng 1 instance
      - Nếu tăng 20–30% → Tăng 2 instances
      - Nếu tăng >30% → Tăng 3 instances
    + CPU tăng từ 50% → 70% (tức +20%) → tăng 2 instances
    + Nếu sau đó tăng tiếp lên 90% → tăng thêm 3 instances
    + Scale-in cũng hoạt động tương tự: nếu CPU giảm nhiều, bạn có thể giảm 1–2–3 instances theo từng mức.
- Không nên scale nhiều lần liên tục → nên thiết lập cooldown hợp lý
- Luôn sử dụng CloudWatch để giám sát metric và kết hợp Auto Scaling
- Không thể scale trên nhiều region
- Cách tiếp cận nhẹ nhàng:
  + Dùng Launch Template
  + Gắn với Application Load Balancer
  + Bật Auto Scaling Group với Target Tracking Policy
  + Giám sát bằng CloudWatch Dashboards
  + Thiết lập cảnh báo (Alarm) nếu số instance lên quá cao
- ***Scaling based on SQS (Amazon Simple Queue Service)***
  + Tự động scale (tăng/giảm) số lượng EC2 instances hoặc container dựa trên số lượng tin nhắn chờ xử lý trong SQS, ví dụ như hệ thống xử lý đơn hàng, email, video encode, v.v.
  + Nếu có nhiều tin nhắn trong queue, hệ thống cần thêm worker (EC2 instance, Lambda, ECS Task…) để xử lý nhanh hơn.
  + Nếu không còn tin nhắn, bạn nên giảm số lượng instance để tiết kiệm chi phí.
  + Metric chính thường dùng `ApproximateNumberOfMessagesVisible`
  + Cách thiết lập Auto Scaling theo SQS:
    + Trường hợp 1: Auto Scaling EC2 bằng SQS (ví dụ: worker xử lý tin nhắn)
    + Tạo Auto Scaling Group (ASG) cho các EC2 worker
    + Trong CloudWatch, tạo Alarm cho metric: SQS > Queue Metrics > ApproximateNumberOfMessagesVisible
    + Thiết lập Alarm như sau:
      + Nếu > 100 tin nhắn trong queue → Scale out (tăng instance)
      + Nếu < 10 tin nhắn → Scale in (giảm instance)
    + Gắn các alarm này vào Scaling Policy trong ASG
    + Ví dụ thực tế 1:
      + Hệ thống gửi email marketing:
        + Mỗi lần người dùng gửi chiến dịch, hệ thống đưa email vào SQS
        + EC2 worker sẽ nhận email từ queue và gửi đi
        + Nếu số email tồn trong queue vượt 100 → scale thêm 2 EC2 worker
        + Khi còn dưới 10 → giảm số worker xuống còn 1
      + Giải pháp:
        + Tăng hoặc giảm số lượng instance tại thời điểm cố định trong ngày hoặc tuần, ví dụ như:
        + Tăng EC2 vào ban ngày (giờ cao điểm)
        + Giảm EC2 vào ban đêm để tiết kiệm chi phí
        + Vào **Auto Scaling Group** trong EC2 console
        + Chọn tab **Automatic Scaling** > **Add Scheduled Action**
        + Nhập các thông tin:
          + Name: scale-up-morning
          + Start time: 08:00 AM hàng ngày
          + Min/Max/Desired capacity: ví dụ set desired = 6
          + Recurrence: ví dụ 0 8 * * * (theo cron expression)
        + Có thể tạo thêm 1 hành động để scale down vào 22:00 với desired = 1
    + Ví dụ thực tế 2 dùng Scaling based on a schedule:
      + Hệ thống đặt hàng online:
        + Giờ cao điểm là từ 9:00 đến 22:00
        + Bạn đặt lịch:
          + 09:00: Scale lên 8 instance
          + 22:00: Scale xuống 2 instance
        + Không cần đợi CloudWatch metric phản hồi

### Route 53
- Route 53 là một dịch vụ quản lý tên miền (DNS) được AWS quản lý hoàn toàn, cho phép bạn định tuyến người dùng đến các tài nguyên trên AWS hoặc bên ngoài AWS một cách nhanh chóng và đáng tin cậy.
- Cấu hình Failover:
  + Active-Active Configuration (Cấu hình hoạt động song song)
    + Khi bạn muốn tất cả tài nguyên cùng hoạt động liên tục, chia tải với nhau.
    + Cách hoạt động: Route 53 định tuyến đến nhiều tài nguyên đồng thời. Nếu một trong số đó không còn hoạt động (unhealthy), Route 53 tự động loại bỏ tài nguyên đó khỏi kết quả DNS.
    + Ví dụ: 2 web server đặt ở 2 vùng khác nhau (US-East-1, US-West-1) cùng phục vụ một website.
    + Best Practice: Sử dụng health checks để theo dõi trạng thái của từng tài nguyên.
  + Active-Passive Configuration (Cấu hình hoạt động-dự phòng)
    + Khi bạn có một nhóm tài nguyên chính hoạt động thường xuyên và một nhóm dự phòng chỉ sử dụng khi nhóm chính gặp sự cố.
    + Cách hoạt động: Bình thường chỉ định tuyến đến nhóm chính. Khi tất cả nhóm chính đều hỏng, Route 53 bắt đầu chỉ định tuyến đến nhóm dự phòng.
    + Ví dụ: Một web app chạy ở US-East-1 và backup dự phòng ở EU-West-1 chỉ hoạt động khi khu vực chính có sự cố.
    + Best Practice: Cấu hình health check và cấu hình weight/priority để định tuyến phù hợp.


### Key Management Service - KMS
- KMS là "người giữ chìa khóa" an toàn nhất, bạn dùng dịch vụ khác (S3, EBS, RDS...) thì "gửi đồ" cho KMS mã hóa/giải mã giúp, nhưng bạn kiểm soát ai được dùng chìa.
- Mục đích: giữ an toàn cho dữ liệu của bạn, cả khi lưu trữ và truyền tải.
- Mã hóa AMI và EBS Volume:
  + Có thể chia sẻ khóa với nhiều tài khoản AWS bằng cách thêm quyền cho các tài khoản khác trên khóa chính (CMK).
- Tích hợp sẵn với các dịch vụ như S3, SQS, DynamoDB để tự động mã hóa dữ liệu.
- Cách hoạt động đơn giản:
  + Bạn tạo một CMK (Customer Master Key) trong KMS.
  + Khi mã hóa một file, KMS sinh ra một "data key", dùng để mã hóa file.
  + KMS mã hóa luôn cái "data key" đó bằng CMK bạn đã tạo.
  + Khi giải mã, KMS dùng CMK để giải mã lại data key, sau đó dùng data key để giải mã file.
  + Như vậy khóa chính không bao giờ ra ngoài, AWS chỉ trả về kết quả mã hóa/giải mã.
- Khi nào cần dùng? Khi cần mã hóa các dữ liệu sau: 
  + Dữ liệu lưu trong S3 (hình ảnh, tài liệu,...)
  + EBS chứa hệ điều hành hoặc database
  + Cơ sở dữ liệu RDS
  + Tin nhắn trong SQS
  + Hoặc bạn muốn mã hóa chuỗi nhạy cảm như mật khẩu, token...
- Ví dụ:
  + Mã hóa dữ liệu lưu trữ trong S3 bằng KMS để bảo vệ nội dung nhạy cảm như thông tin khách hàng.
  + Tạo CMK riêng trong KMS và phân quyền sử dụng cho team hoặc dịch vụ cụ thể.
- Lưu ý:
  + Mã hóa với KMS không miễn phí: có phí theo số lần gọi API và số khóa bạn tạo.
  + Nếu dùng sai quyền IAM/KMS Policy, ứng dụng có thể bị lỗi không giải mã được → phải test kỹ.


### RDS
- Amazon RDS cung cấp cơ sở dữ liệu có thể mở rộng và được quản lý hoàn toàn.
- Multi-AZ deployments: sao lưu đồng bộ dữ liệu giữa nhiều AZ, tăng tính sẵn sàng.
- Tính giá theo mức sử dụng.
- Hỗ trợ scale chiều ngang (read replicas) và dọc (tăng cấu hình).
- Minimal downtime khi scale.
- Các engine được hỗ trợ: MySQL, MariaDB, PostgreSQL, Oracle, SQL Server (không hỗ trợ read replicas ở vùng khác), Aurora
- Best Practice:
  + Dùng Aurora Serverless cho ứng dụng có tải biến động.
  + Kết hợp với IAM authentication để quản lý truy cập an toàn hơn.


### Lambda - Serverless
- AWS Lambda cho phép bạn chạy code mà không cần quản lý máy chủ.
- Tự động scale theo số lượng yêu cầu.
- Tích hợp với các dịch vụ như S3, SQS, SNS, API Gateway.
- Có thể gắn vào VPC mà không cần lo về ENI/Private IP như trước.
- Lambda Layer: quản lý dependency dùng chung giữa các function.
- Destination: Sau khi chạy xong, có thể chỉ định một service khác nhận kết quả (thành công hoặc thất bại).
- Best Practice:
  + Dùng Graviton2 nếu cần hiệu năng cao và tiết kiệm chi phí.
  + Đặt timeout, bộ nhớ hợp lý để tránh lỗi OutOfMemory hoặc timeout.
  + Tránh đặt quá nhiều logic vào một function → chia nhỏ chức năng.


### CloudFront - CDN
- CloudFront giúp phân phối nội dung tĩnh hoặc động gần với người dùng hơn, cải thiện tốc độ tải và giảm tải server gốc.
- Có hơn 225 edge location + 13 mid-tier caches trên toàn thế giới.
- Multiple origins: một phân phối có thể truy xuất dữ liệu từ nhiều nguồn.
- Cache policy tùy chỉnh: kiểm soát phần nào của request dùng làm khóa cache.
- Lambda@Edge / CloudFront Functions: chạy code tại edge location (thường dùng để rewrite URL, xử lý xác thực nhẹ,...).
- Geo restriction: giới hạn truy cập theo quốc gia.
- Tích hợp WAF: kiểm soát request HTTP/S và bảo vệ nội dung khỏi tấn công.
- Ví dụ:
  + Website toàn cầu → dùng CloudFront phân phối ảnh và nội dung tĩnh từ S3, giảm độ trễ.
  + Dùng WAF để chặn bot hoặc IP xấu.
- Best Practice:
  + Thiết lập TTL hợp lý để tận dụng caching nhưng vẫn đảm bảo cập nhật nhanh khi nội dung thay đổi.
  + Dùng Origin Groups để cấu hình failover cho nguồn gốc (giống như S3 + EC2 fallback).
  + Tích hợp với Lambda@Edge để xử lý URL redirect, header injection.


### IAM (Identity and Access Management)
- Hệ thống quản lý truy cập và định danh – bảo mật trong AWS.
- Không dùng tài khoản root để làm việc hằng ngày – tạo IAM user riêng, bật MFA.
- Các thành phần chính:
  + User: Người dùng, đăng nhập console hoặc API.
  + Group: Nhóm user chia sẻ quyền.
  + Policy: Tập hợp quyền, dạng JSON.
  + Role: Vai trò có thể được assume bởi user/dịch vụ để có quyền tạm thời.
- Mặc định mọi hành động đều bị từ chối, cần cho phép cụ thể trong policy.
- Explicit deny (từ chối rõ ràng) sẽ ưu tiên hơn allow.
- IAM Policy → Định nghĩa quyền, nhưng phải gắn vào User/Group/Role.
- IAM Group → Nhóm Users, không gắn trực tiếp vào EC2. Muốn cấp quyền thì phải gắn Policy vào Group.
- IAM User → Dành cho người/dịch vụ, không dành cho EC2.
- Cách đúng để cấp quyền cho EC2 instance? → IAM Role
- Best practice:
  + Dùng policy nguyên tắc “least privilege” – chỉ cấp đúng quyền cần thiết.
  + Gắn role vào EC2 hoặc Lambda thay vì dùng access key thủ công.

### CloudWatch
- Log Events: Dòng log có thời gian, từ các dịch vụ như Lambda, ECS.
- Log Groups: Nhóm các log liên quan đến cùng 1 dịch vụ.
- Log Streams: Mức chi tiết hơn, ví dụ log của 1 Lambda instance cụ thể.
- Metrics: Số liệu hệ thống (CPU, RAM...), được thu thập mặc định và không tính phí.
- X-Ray: Theo dõi yêu cầu phân tán (distributed tracing) – xem request đi qua các dịch vụ nào.
- Alarms: Thiết lập cảnh báo, có thể:
  + Gửi email qua SNS.
  + Kích hoạt autoscaling khi vượt ngưỡng.
- Đặt cảnh báo cho lỗi, độ trễ cao, hoặc chi phí bất thường.
- Kết hợp với Lambda để xử lý sự kiện real-time.

### CloudFormation
- Cho phép bạn định nghĩa và triển khai hạ tầng AWS như EC2, S3, VPC, RDS... thông qua file cấu hình (template) dạng YAML hoặc JSON.
- **Template**: là bản mô tả đầy đủ các tài nguyên cần triển khai.
- **Stack**: là một đơn vị triển khai của template – khi bạn deploy template, CloudFormation sẽ tạo một stack gồm tất cả resource được khai báo.
- **Change Sets**: CloudFormation tự động phân tích sự khác biệt giữa template mới và stack hiện tại, để quyết định cần tạo, cập nhật, hay xóa resource nào.
- **Outputs**: cho phép xuất các giá trị (ID, ARN, URL) để các stack khác hoặc dịch vụ khác có thể sử dụng.
- **Lợi ích chính**:
  + Triển khai hạ tầng nhất quán và lặp lại
  + Quản lý thay đổi rõ ràng
  + Rollback khi deploy lỗi
  + Tự động hóa CI/CD
- Khi nào nên dùng AWS CloudFormation?
| Tình huống | Lý do |
|-----------|-------|
| muốn **quản lý hạ tầng như code (IaC)** | Dễ kiểm soát, lưu Git, kiểm tra thay đổi |
| cần **tái sử dụng hoặc triển khai lặp lại nhiều lần** | Viết 1 template → deploy ở nhiều region/account |
| triển khai **ứng dụng có nhiều dịch vụ liên kết** | Kiểm soát thứ tự tạo resource (VPC → EC2 → ELB...) |
| cần **CI/CD cho hạ tầng** | Kết hợp với CodePipeline để tự động triển khai |
| muốn **rollback khi deploy lỗi** | CloudFormation hỗ trợ rollback stack khi có lỗi |
- Ví dụ thực tế:
  + Tạo 1 file template để triển khai:
    - VPC + Subnet
    - ECS Cluster + Task
    - Application Load Balancer
    - IAM Role
    - Auto Scaling Group  
    → Sau đó chỉ cần đổi biến là có thể triển khai cho dev/test/prod.
- **Best Practices**:
  + Sử dụng **modular templates** (nested stacks)
  + Sử dụng **Parameters**, **Mappings**, **Conditions**, và **Outputs** hợp lý và để tái sử dụng
  + Kiểm tra thay đổi bằng **Change Sets** trước khi apply
  + Gắn tag cho tài nguyên để dễ quản lý chi phí
  
### ELB (Elastic Load Balancer)
- Dịch vụ cân bằng tải giúp phân phối lưu lượng đến các compute resource.
- Có 4 loại ELB:
  + Classic (CLB): Đã cũ, không còn dùng nhiều.
  + Application (ALB): Layer 7, định tuyến dựa vào nội dung HTTP/HTTPS (VD: path, host).
  + Network (NLB): Layer 4, định tuyến theo IP/port.
  + Gateway (GLB): Layer 3/4, dùng trước firewall hoặc NAT Gateway.
- Cân bằng tải giúp tăng độ sẵn sàng (high availability) bằng cách chuyển request đến các healthy targets, có thể nằm ở nhiều AZ khác nhau.
- Có 2 chế độ:
  + Internet-facing: Có IP công cộng, dùng cho người dùng ngoài.
  + Internal-only: Chỉ dùng trong VPC nội bộ, không public IP.
- Best practice:
  + Dùng ALB cho ứng dụng web (HTTP).
  + Dùng NLB cho TCP/UDP cần độ trễ thấp, như IoT hoặc game server.

### SNS (Simple Notification Service)
- Dịch vụ gửi thông báo được quản lý hoàn toàn, theo mô hình publish/subscribe.
- Chủ đề (topic): Nơi gửi thông báo.
- Người đăng ký (subscriber): Nhận thông báo từ chủ đề (VD: email, Lambda, SQS).
- Có hai loại:
  + Standard: Không đảm bảo thứ tự.
  + FIFO: Đảm bảo thứ tự và loại bỏ trùng lặp.
- Có thể lưu trữ thông điệp bằng cách chuyển đến Kinesis Firehose rồi đưa vào S3 hoặc Redshift để phân tích.


### CloudTrail 
- CloudTrail là dịch vụ giám sát và ghi lại mọi hoạt động trong tài khoản AWS của bạn dưới dạng JSON (so sánh với CloudWatch, CloudTrail tập trung vào ghi log hoạt động quản lý tài khoản).
- Có thể quyết định sự kiện nào được theo dõi bằng cách tạo "trails" (đường dẫn log). Mỗi trail sẽ chuyển tiếp sự kiện đến:
  + S3 bucket (để lưu trữ lâu dài)
  + CloudWatch Logs group (để phân tích real-time)
- Các loại sự kiện CloudTrail ghi lại:
  + Management Events
    + Các thao tác quản lý hạ tầng AWS
    + Ví dụ:
        + Thay đổi IAM policy
        + Tạo mới VPC subnet
        + Khởi tạo EC2 instance
  + Data Events
    + Các thao tác đọc/ghi/xóa dữ liệu trong tài khoản
    + Ví dụ:
        + CRUD trên DynamoDB
        + GET object trong S3
        + Upload file lên S3
  + Insight Events
    + Ghi nhận các bất thường trong cách sử dụng API dựa trên lịch sử hoạt động
    + Ví dụ:
        + Đột ngột có nhiều request xóa dữ liệu S3
        + Số lần login IAM tăng đột biến
- Customize tracking:
  + Chỉ theo dõi thao tác modify/delete trên DynamoDB, bỏ qua các thao tác đọc
  + Chỉ ghi lại các thay đổi security group
- Best Practice:
  + Bật CloudTrail trên toàn bộ regions
    + Kích hoạt multi-region trail để đảm bảo ghi lại mọi hoạt động dù xảy ra ở region nào
  + Tích hợp với S3 và CloudWatch Logs
    + Lưu trữ log vào S3 với versioning và bucket policy nghiêm ngặt
    + Forward log sang CloudWatch để cảnh báo real-time
  + Mã hóa log file
    + Sử dụng KMS để mã hóa log file trước khi lưu vào S3
  + Kích hoạt Data Events cho các dịch vụ quan trọng
    + Ví dụ: S3, DynamoDB, Lambda nếu cần audit chi tiết
  + Thiết lập cảnh báo cho các hoạt động nguy hiểm
  + Kích hoạt CloudTrail Insights
  + Giới hạn truy cập log: Áp dụng principle of least privilege cho IAM user/role truy cập log
  + Thiết lập retention policy phù hợp


### AWS CloudWatch vs CloudTrail: So sánh

| Feature          | CloudWatch                          | CloudTrail                          |
|-------------------|-------------------------------------|-------------------------------------|
| **Purpose**      | Giám sát hiệu năng hệ thống        | Audit hoạt động quản lý            |
| **Data Collected**   | Metrics, logs, events              | API calls, thay đổi cấu hình       |
| **Data Format**     | Số liệu/metrics và logs text       | Events JSON                        |
| **Pricing Model**     | Pay per metric/log/alert                 | Pay per event delivered                     |
| **Storage**          | CloudWatch Logs (retention configurable)| S3 bucket (infinite retention possible) |


### GuardDuty
- là một dịch vụ phát hiện mối đe dọa (threat detection) chứ không phải dịch vụ kiểm tra hoặc lọc lưu lượng mạng (traffic inspection/filtering).
- GuardDuty giống như camera an ninh (phát hiện kẻ đột nhập) → Báo cho bạn biết để xử lý.
- Khi phát hiện nguy cơ, nó gửi cảnh báo (alert) qua AWS Console hoặc Amazon EventBridge để bạn xử lý.
- Bạn cần dùng kết hợp GuardDuty với các dịch vụ khác (như AWS WAF, Shield, Network ACL) để vừa phát hiện vừa ngăn chặn mối đe dọa.
- GuardDuty KHÔNG làm gì?
  + Không chặn lưu lượng xấu (như tường lửa - Firewall).
  + Không kiểm tra từng gói tin (như IDS/IPS).
  + Không tự động ngăn tấn công (bạn cần kết hợp với AWS WAF, Shield, hoặc NACL để chặn).
  






