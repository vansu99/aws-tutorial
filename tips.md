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
42. Khi cần kết nối môi trường On-premise với nhiều AWS account thông qua DirectConnect thì nghĩ đến Transit Gateway
43. DB dạng quan hệ có khả năng tự scale để đáp ứng traffic thì nghĩ đến Aurora Serverless
44. Transfer Acceleration dùng cho việc tăng tốc upload trực tiếp lên S3
45. S3 cần copy sang region khác thì nghĩ đến S3-cross region replication
46. Inspector chỉ giúp phát hiện các lỗ hổng bảo mật (vulnerabilities) trong EC2 instances và container images, không có khả năng giám sát và phát hiện các hoạt động bất thường.
47. Config chỉ giúp ghi lại lịch sử thay đổi setting của resource và đặt ra quy định để đảm bảo tuần thủ, không có khả năng giám sát và phát hiện các hoạt động bất thường.
48. Phát hiện hoạt động bất thường, đáng ngờ "malicious" thì nghĩ đến GuardDuty
49. Tổng hợp và quản lý các phát hiện security (security findings) thì nghĩ đến Security Hub
50. Keyword "search, real-time update" thì thường sẽ nghĩ đến OpenSearch
51. Keyword thời gian thực "real-time" thì thường sẽ nghĩ đến Kinesis Data Streams
52. Keyword hiển thị hoá "visualization" thì thường sẽ nghĩ đến QuickSight
53. "encrypt before storing" - tức là data phải được mã hóa trước khi gửi lên S3. Do đó sử dụng Client-side encryption để đảm bảo dữ liệu được mã hóa tại client trước khi upload.
54. server-side encryption - data được mã hóa SAU KHI đã đến S3, không phải "before storing"
55. Khi cần solution storage cho local mà support dạng block storage thì sẽ nghĩ đến Volume Gateway. Keyword: iSCSI, Block Storage, Volume
56. Với lượng data hàng trăm TB hay hàng PB thì việc migrate hầu như sẽ thực hiện qua Snowball Edge
57. WAF có thể hạn chế tấn công DDoS thông qua Rate-based rule
58. Intelligent-Tiering chỉ tối ưu chi phí storage S3 dựa trên tần suất truy cập, không giải quyết vấn đề cross-region transfer cost.
59. Nhiều RDS DB instances chạy trong development account. Tất cả instances đều có gắn tags để nhận diện (development resources). Muốn instance chỉ chạy theo lịch trong giờ hành chính => Dùng State Manager. Chỉ cần tạo association với schedule expression, chọn target bằng tags, và State Manager sẽ tự động thực hiện start/stop theo thời gian định sẵn. Ít operational overhead nhất vì không cần viết code Lambda hay setup monitoring phức tạp.
60. Trusted Advisor là service đưa ra các gợi ý về mặt vận hành, security trong account, không có chức năng vận hành RDS theo lịch.
61. Giảm thời gian failover cho RDS hoặc Aurora thì nghĩ đến RDS Proxy
62. Chạy job mà cần tính ổn định, Cannot be interrupted thì sẽ tránh đáp án Spot Instances
63. Auto Scaling Group cần scale tự động để đáp ứng traffic thì nghĩ đến target tracking scaling
64. Athena là dịch vụ serverless cho phép query data trực tiếp trên S3. Vì là serverless nên sẽ thích hợp cho việc truy vấn occasional (không thường xuyên). Athena chỉ cần trả tiền theo lượng dữ liệu scan thực tế, không có chi phí cố định. Ngoài ra có performance cao tương thích với Parquet và có support mã hoá client CSE-KMS encryption.
65. Việc query data trên S3 thỉnh thoảng, không thường xuyên thì nghĩ ngay đến Amazon Athena
66. Đảm bảo được việc chỉ xử lý message đúng 1 lần duy nhất (exactly-once processing) thì nghĩ đến SQS FIFO
67. Cách lựa chọn các tầng lưu trữ cho S3:
  + Truy cập nhiều, thường xuyên → Standard
  + Ít truy cập + khi cần truy cập phải lấy được file ngay (immediate access) → Standard-IA (độ bền cao và đắt hơn) hoặc One Zone-IA (độ bền thấp nhưng rẻ)
  + Tiết kiệm chi phí hơn nữa, chấp nhận việc phải chờ mới lấy được file → Glacier Family (Instant Retrieval, Flexible Retrieval, Deep Archive), trong đó:
    + Trả phí để được lấy file ngay lập tức: Glacier Instant Retrieval
    + Có thể đợi từ vài phút đến vài giờ: Glacier Flexible Retrieval
    + Chỉ để lưu trữ lâu dài: Glacier Deep Archive
  + Tần suất truy cập không cố định, không biết trước (Unknown access pattern) → Intelligent-Tiering
69. "LEAST effort" + "backup" + "hundreds of instances" → AWS Backup
70. "Centralized backup" → AWS Backup
71. AWS Backup là lựa chọn hàng đầu cho việc quản lý backup tập trung.
72. Khi đề bài nhấn mạnh "LEAST effort" với large-scale backup, ưu tiên managed services thay vì manual solutions
73. "Database credentials" + "rotation" → AWS Secrets Manager.
74. "Encryption + on-premises to VPC" → Site-to-Site VPN (có IPSec encryption tự động)
75. Transit Gateway chủ yếu để kết nối nhiều VPC hoặc on-premises networks, không tạo private connection với third-party SaaS.
76. "Third-party SaaS + private connectivity" → AWS PrivateLink.
77. Kiến trúc tham khảo về luồng kết nối từ Lambda đến RDS
    - Lambda request vào Secrets Manager để lấy thông tin đăng nhập DB
    - Lambda connect vào DB
    - RDS có liên kết trực tiếp với Secrets Manager để rotation định kì
78. Khi đề bài yêu cầu traffic cân bằng tải trên một nhóm replica cụ thể → nghĩ đến Custom Endpoint
79. Networking inspection -> Thường nghĩ đến Gateway Load Balancer
80. Multi-AZ RDS: đảm bảo database chạy trên 2 AZ, có failover tự động khi có sự cố, tăng tính khả dụng
81. Đối với EC2 để multi-az thì sẽ sử dụng Auto Scaling Group để scale trên nhiều zone, có thể kết hợp cân bằng tải với ALB nếu cần thiết
82. "Cost breakdown by application" → Cost allocation tags + Cost Explorer
83. Cost allocation tags cho phép quản lý và phân chia chi phí theo tag, và Cost Explorer cung cấp dashboard trực quan với khả năng tạo report, cập nhật thường xuyên tự động - tất cả đều hoàn toàn miễn phí.
84. "Automatic key rotation" → KMS (chỉ KMS mới có automatic rotation)
85. "Long-running workloads" → Reserved Instances/Savings Plans với cam kết sử dụng dài hạn
86. Payment Options theo mức discount:
  - All Upfront (discount cao nhất)
  - Partial Upfront (discount trung bình)
  - No Upfront (discount thấp nhất)
87. "Millions of objects" → Nghĩ đến S3 Batch Operations (thao tác hàng loạt)
88. "Each time upload" → S3 Event Notification (real-time trigger)
89. "Performance analysis" → Câu hỏi về performance analysis nên sẽ cần metrics (CloudWatch Metrics), không phải CloudWatch Logs
90. "Granularity 1-2 minutes" → CloudWatch Detailed Monitoring (1-minute intervals)
91. Keyword "Reports" → Nghĩ đến Read replica
92. Read-heavy workload gây chậm → Read replica là giải pháp đầu tiên
93. "Accidentally deleted" + "Prevent data loss" → Nghĩ đến Recycle Bin
94. "TCP/UDP traffic" → chỉ có Network Load Balancer & AWS Global Accelerator
95. "TCP/UDP traffic" → Loại hết các đáp án có ALB, API GW, CloudFront vì chỉ không support TCP/UDP
96. "Granular control + same bucket + different prefixes" → S3 Access Points
97. "Restrict application to specific prefix" → Access Point Policies
98. "Third-party CA" → phải import certificate vào ACM (không thể tạo và sign trực tiếp)
99. ECS Fargate là lựa chọn phù hợp cho việc chạy job trong thời gian dài vì có tính ổn định, trong quá trình chạy sẽ không bị gián đoạn.
100. Đối với các câu hỏi về việc chạy job thì các solution thường nghĩ đến đó là AWS Lambda, ECS Fargate, Batch, EC2 Spot Instances.
101. Đầu tiên cần xem thời gian chạy job là bao lâu, nếu trên 15 phút sẽ loại ngay Lambda, ưu tiên chọn các solution managed, serverless như ECS Fargate, Batch.
102. "Aurora" + "DR" → nghĩ đến Aurora Global Database
103. Best Practice cho Static Website:
    Dùng Cache-Control max-age dài để tối ưu bandwidth
    Kết hợp CloudFront invalidation khi deployment để đảm bảo fresh content
    Tránh TTL = 0 vì mất lợi ích caching
104. Cost and Usage Report → S3 → Athena → QuickSight
105. "Peak usage + timeouts" → Caching (ElastiCache) + Decoupling (SQS)
106. "Scale cost-effectively" → Auto Scaling + managed services
107. "Application timeouts" → Database bottleneck → cần caching + async processing
108. "DDoS attacks" → AWS Shield
109. "Layer 7 attacks" → WAF
110. "Layer 3/4 attacks" → Shield
111. "Low traffic" + "cost-effective" → Serverless (Lambda + API Gateway)
112. "Cost-effectively" + "High Availability" → Auto Scaling (pay for what you use) + Load Balancer
113. High Availability pattern: Multi-AZ + Auto Scaling + Load Balancer
114. NAT Gateway = public internet routing → vi phạm compliance requirements
115. "Private subnet" + "S3 access" + "not public internet" → VPC Endpoints
116. Security Hub là central security dashboard tổng hợp từ nhiều sources, nhưng không chuyên về vulnerability assessment chi tiết như Inspector.
117. Automate OS updates/patching → AWS Systems Manager Patch Manager
118. Vulnerability assessment + monthly reports → Amazon Inspector
119. "Multiple AWS accounts" + "centralized control" → AWS Organizations + SCP
120. "Low login latency" → Lambda@Edge (runs at edge locations)
121. "Serverless auth/authz" → Amazon Cognito + Lambda@Edge
122. Transfer Acceleration = faster uploads, không phải web serving
123. Lambda@Edge = low latency, regular Lambda = higher latency
124. "Cross-account access" → IAM Role với Trust Policy
125. "Development account users" → Trust policy specify Development account
126. Cross-account pattern: Role in target account + Trust policy specify source account
127. "Order processing" → SQS + Lambda pattern
128. ".csv files" + "SQL queries" → Glue crawler + Athena
129. "SAP" + "high memory utilization" → Memory optimized instances
130. "SQL Server database" + "high memory" → Memory optimized instances
131. "Aurora scaling" → Aurora Read Replicas là standard solution
132. DAX only works với DynamoDB, không phải Aurora
133. Redshift = analytics/data warehouse, không phải operational reads
134. "Write performance degradation" + "read traffic" → Read replicas
135. "Daily backup" + "RDS" → RDS automated backups (đã có sẵn daily)
136. "Retention period" → Modify RDS backup retention setting
137. RDS automated backups default enabled với configurable retention (1-35 days)
138. "HPC workload" + "low latency" + "high throughput" → Cluster placement groups
139. Spot instances = cost optimization, không phải performance optimization
140. "External consultant" + "temporary access" → Presigned URLs
141. "CloudFront" + "SSL certificate" → ACM certificate trong us-east-1
142. "Public certificate" = free từ ACM
143. "Threat detection" + "suspicious behavior" → Amazon GuardDuty
144. "Automated response" + "WAF rules" → EventBridge + Lambda integration
145. "Shared storage" + "multiple EC2 instances" → Amazon EFS
146. "Holiday each year" → Recurring scheduled actions
147. "HPC" + "tightly coupled" → Elastic Fabric Adapter (EFA)
148. "HPC" + "storage" → FSx for Lustre
149. SSM là service để quản lý và vận hành EC2, không có cơ chức năng hạn chế hay ngăn chặn người dùng tạo EC2.
150. Service Catalog là service để tạo và quản lý các solution được build dựa trên CloudFormation, không có cơ chức năng hạn chế hay ngăn chặn người dùng tạo EC2.
151. "Multiple AWS accounts" + "centrally restrict/control" → AWS Organizations + SCP . Bài toán nhắc đến kiến trúc multi-account và cần chặn một action nào đó → luôn nghĩ đến SCP
152. Cognito User Pool là service chuyên dụng cho việc quản lý và xác thực người dùng (authentication) với đầy đủ tính năng: sign-up, sign-in, password recovery, MFA
153. API Gateway REST APIs + Cognito Authorizer là pattern chuẩn cho RESTful APIs với authentication
154. Lưu ý Cognito User Pool dùng để quản lý và xác thực đăng nhập người dùng (authentication), Cognito Identity Pool dùng để cấp quyền cho người dùng thao tác với AWS Service (authorization)
155. "Lambda + environment variables + encrypt" → sử dụng AWS KMS
156. "Control Tower" + "Prevent/Block deployment" → Proactive Controls
157. "Occasionally" + "SQL query" + "S3" → Athena (serverless, pay-per-query)
158. "Audit all API calls + logins" → AWS CloudTrail (using with Control Tower)
159. "Collect data/usage/configuration on-premises servers" → AWS Application Discovery Service
160. "No duplicate" / "Exactly-once" → SQS FIFO Queue






























