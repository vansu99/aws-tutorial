1. Khi câu hỏi nhắc đến việc liên kết người dùng Active Directory để cho phép truy cập account trong Organization thì thường nghĩ ngay đến IAM Identity Center
2. API Gateway là managed service nên không thể đặt bên trong subnet VPC. Kể cả Private REST APIs thì cũng chỉ là cơ chế cho phép access đến API Gateway thông qua endpoint từ VPC, chứ không phải bản thân API GW được đặt trong VPC
3. Hệ thống global cần hạn chế quyền truy cập theo khu vực hoặc quốc gia, có thể nghĩ đến Route53 Geolocation Routing Policy
4. Access đến các service từ bên trong VPC mà không đi qua internet, nghĩ ngay đến Gateway Endpoint (áp dụng cho S3, DynamoDB)
5. Các câu hỏi liên quan đến web tĩnh, content tĩnh mà cần tối ưu hoá độ trễ hay tăng tốc upload thì thường sẽ nghĩ đến CloudFront, S3 Transfer Acceleration
6. Xuất hiện keyword Workflow thì thường sẽ nghĩ đến Step Functions
7. Solution cần ít effort vận hành thì các đáp án sẽ không ưu tiên chọn EC2
8. Các yêu cầu về chạy job mà tốn ít effort thì thường sẽ nghĩ đến các service như Lambda, ECS Fargate, AWS Batch
9. Để cho phép các user trong Active Directory có thể access môi trường aws của một account đơn lẻ, không phải AWS Organizations thì sẽ nghĩ đến liên kết SAML 2.0 & IAM Role
10. Khi bài toán có EC2 xử lí SQS mà gặp vấn đề bottleneck thì nghĩ đến việc sử dụng kết hợp Auto Scaling Group và scale dựa trên metric số lượng message có trong queue
11. Việc scale EC2 thì thường sẽ sử dụng Target Tracking policy vì có tính tự động hoá cao, giúp đáp ứng khi gặp traffic lớn hoặc thất thường
12. Để cấp quyền cho các user bên ngoài hệ thống có thể truy cập các AWS Service thì nghĩ đến Cognito Identity Pool
13. Cho phép application trong VPC access đến các service khác một cách an toàn thì nghĩ đến VPC Endpoint
14. Nhắc đến việc migrate lift-and-shift hay migrate server nói chung thì thường sẽ nghĩ đến Application Migration Service (MGN)
15. Với các application stateless và có khả năng chịu lỗi tốt mà cần cơ chế scale thì thường sẽ nghĩ đến Auto Scaling Group & Spot Instances
16. Bài toán nhắc đến use case ETL (extract-transform-load) và yêu cầu tốn ít effort (LEAST operational effort) thì thường sẽ nghĩ đến AWS Glue
17. Các câu hỏi liên quan đến use case host web tĩnh thì thường sẽ nghĩ đến CloudFront và S3
18. Đề bài nhắc đến DB có khả năng linh hoạt đáp ứng thay đổi schema (rapidly evolve schema) thì sẽ nghĩ đến DynamoDB
19. Phân tích thái độ người dùng (sentiment analysis) thì thường nghĩ ngay đến Comprehend
20. Các bài toán cần tăng tốc kết nối đến hệ thống thì thường nghĩ đến Global Accelerator
21. Cần solution về storage cho phía on-premise cho phép kết nối và đồng bộ lên s3 thì sẽ nghĩ đến S3 File Gateway




