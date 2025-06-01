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
- Backup and Restore – có RTO (Recovery Time Objective) và RPO (Recovery Point Objective) cao nhất nhưng chi phí thấp nhất.
- Pilot Light – lưu trữ các hệ thống quan trọng ở trạng thái tối thiểu làm mẫu, từ đó có thể mở rộng nhanh chóng khi xảy ra issue.
- Warm Standby – duy trì phiên bản sao của các hệ thống quan trọng luôn chạy sẵn, sẵn sàng tiếp quản khi cần.
- Multi-Site – có RTO và RPO thấp nhất nhưng chi phí cao nhất.

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
- Gateway Endpoint:
  + Dành cho S3 và DynamoDB.
  + Tạo một "cổng" trong VPC để truy cập dịch vụ, không cần dùng Internet Gateway hay NAT.
- Interface Endpoint:
  + Dùng cho các dịch vụ khác (EC2, SSM, SNS,...).
  + Sử dụng AWS PrivateLink để đảm bảo kết nối riêng tư qua ENI (Elastic Network Interface).
- Luôn sử dụng VPC Endpoint nếu muốn tăng bảo mật bằng cách không cho lưu lượng ra internet mà vẫn gọi được các dịch vụ AWS.

### NAT Gateway & NAT Instance – Kết nối subnet private ra internet
- Khi bạn có subnet riêng (private subnet) và muốn các instance trong đó truy cập internet (ví dụ tải package) mà không muốn chúng bị truy cập từ bên ngoài.
- NAT Instance:
  + Tự quản lý (cài đặt, cập nhật, scale), không có auto-scaling mặc định.
  + Dễ cấu hình nhưng tốn công quản lý, ít được khuyến khích.
- NAT Gateway:
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
- Target Tracking:
  + Bạn chọn một metric (VD: CPU = 50%)
  + Auto Scaling sẽ tự động tạo CloudWatch alarm và scale theo.
  + Có thể định nghĩa thời gian "warm-up" để tránh scale ảo do EC2 mới tạo có CPU cao.
- Step Scaling:
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
- Ví dụ cấu hình Step Scaling Policy:
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

