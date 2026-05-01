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
22. Cho phép application trong VPC access đến các service khác một cách an toàn, không đi qua internet thì nghĩ đến VPC Endpoint
23. DB dạng quan hệ có khả năng tự scale để đáp ứng traffic thì nghĩ đến Aurora Serverless
24. Company phát triển application với HTTP API trong API Gateway. Chỉ muốn cho phép truy cập từ danh sách IP giới hạn (limited set of trusted IP addresses) thuộc mạng nội bộ công ty (company's internal network). Create a resource policy for the API that denies access to any IP address that is not specifically allowed. Resource Policy cũng là một cách không mất tiền để định nghĩa quyền kiểm soát truy cập API Gateway theo IP. Policy có thể định nghĩa chính xác các IP được phép và từ chối tất cả IP khác. Đây là best practice cho IP whitelisting trong API Gateway.
25. API Gateway không sử dụng security groups như EC2. Security groups không áp dụng cho managed services như API Gateway.
26. Private integration chỉ là cơ chế cho phép API Gateway kết nối với backend services trong VPC, không liên quan đến việc kiểm soát client access theo IP.
27. Đối với các câu hỏi về việc chạy job thì các solution thường nghĩ đến đó là AWS Lambda, ECS Fargate, Batch, EC2 Spot Instances.
28. Đầu tiên cần xem thời gian chạy job là bao lâu, nếu trên 15 phút sẽ loại ngay Lambda, ưu tiên chọn các solution managed, serverless như ECS Fargate, Batch. Nếu thời gian dưới 15 phút thì thường sẽ lựa chọn Lambda.
29. Lưu ý RDS hầu như không bao giờ để trong Public subnet
30. Để kết nối on-premise đến VPC an toàn, có mã hoá trên đường truyền thì có thể sử dụng Site-to-Site VPN
31. Đối với yêu cầu cần buffer request giúp giảm quá tải cho server / db thì có thể sử dụng SQS
32. DynamoDB chỉ là service database, không có có cơ chế queue. Do đó vẫn có thể bị quá tải như thường.
33. Cần bắt buộc việc gắn tag và gán giá trị cho tag trong Organization thì nghĩ đến Tag policy
34. Cần chặn một hành động (action) nào đó trong Organization thì sẽ nghĩ đến SCP
35. Khi thấy từ khóa NFS trên AWS thì thường sẽ nghĩ đến EFS
36. Fileshare cho Linux instance thì thường sẽ nghĩ đến EFS
37. Sử dụng S3 không biết trước tần suất, chỉ muốn tiết kiệm chi phí thì sẽ nghĩ đến Amazon S3 Intelligent-Tiering
38. Amazon S3 Standard: Không tiết kiệm chi phí cho data ít được access, Giá cao nhất trong các S3 storage class
39. Amazon S3 Glacier Deep Archive: Đây là tầng lưu trữ thiết kế để lưu data lâu dài, Không phù hợp với tần suất truy cập không biết trước
40. Amazon S3 One Zone-IA: Không đáp ứng yêu cầu "highest durability" vì chỉ lưu data trong 1 AZ
41. S3 chỉ giúp host static content, không thể host dynamic web application.














