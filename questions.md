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

###
