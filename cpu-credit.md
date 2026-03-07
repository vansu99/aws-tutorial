# AWS EC2 CPU Credit Explained: Vì sao T3 instance đôi khi làm bạn tốn tiền hơn?

Nhiều team khi deploy hệ thống trên AWS thường chọn **T3 instance** vì
giá rẻ.

Ví dụ:

t3.large \~ \$60 / tháng\
m6i.large \~ \$70 / tháng

Nhìn qua thì thấy **T3 rẻ hơn khá nhiều**.

Nhưng trong nhiều hệ thống production, sau vài tháng team mới phát hiện:

-   CPU thường xuyên 100%
-   Server bị chậm
-   AWS billing tăng bất ngờ

Nguyên nhân thường nằm ở **CPU Credit**.

Trong bài viết này chúng ta sẽ hiểu:

-   CPU Credit là gì
-   Vì sao T3 có thể rẻ... nhưng cũng có thể đắt
-   Khi nào nên dùng T3
-   Case study production thực tế
-   Best practice để tối ưu chi phí AWS

------------------------------------------------------------------------

# CPU Credit là gì?

CPU Credit là cơ chế của AWS dành cho **burstable instances**:

-   T2
-   T3
-   T4g

Mục tiêu của AWS là:

> Cho phép instance **burst CPU tạm thời** nhưng không chạy CPU cao liên
> tục.

Định nghĩa chính thức:

1 CPU Credit = 1 vCPU chạy 100% trong 1 phút

Ví dụ:

  CPU Usage                       Credit tiêu thụ
  ------------------------------- -----------------
  1 vCPU chạy 100% trong 1 phút   1 credit
  2 vCPU chạy 100% trong 1 phút   2 credits
  1 vCPU chạy 50% trong 1 phút    0.5 credit

------------------------------------------------------------------------

# Baseline CPU (điều rất nhiều người bỏ qua)

Instance dòng T không chạy full CPU liên tục.

Mỗi instance có một **baseline CPU**.

  Instance    vCPU   Baseline
  ----------- ------ ----------
  t3.nano     2      5%
  t3.micro    2      10%
  t3.small    2      20%
  t3.medium   2      40%
  t3.large    2      60%

Ví dụ:

t3.large baseline = 60%

Điều đó nghĩa là:

CPU \<= 60% → không tốn CPU credit\
CPU \> 60% → bắt đầu tiêu CPU credit

------------------------------------------------------------------------

# CPU Credit hoạt động như thế nào?

Hãy tưởng tượng:

> CPU Credit giống như **pin dự phòng cho CPU**

## Khi server idle

CPU thấp → tích lũy CPU credit

Ví dụ:

CPU = 10%\
baseline = 40%

Phần CPU không dùng:

40% - 10% = 30%

AWS sẽ **tích credit cho bạn**.

------------------------------------------------------------------------

## Khi traffic tăng

CPU = 80%\
baseline = 40%

Phần vượt:

80% - 40% = 40%

Instance bắt đầu **tiêu CPU credit** để burst.

------------------------------------------------------------------------

# Điều gì xảy ra khi CPU Credit = 0 ?

Có 2 chế độ.

## Standard mode

CPU bị **throttle về baseline**.

Ví dụ:

t3.large\
baseline = 60%

CPU sẽ bị khóa:

max = 60%

Kết quả:

-   API chậm
-   queue backlog
-   latency tăng

------------------------------------------------------------------------

## Unlimited mode

Instance vẫn được burst.

Nhưng AWS sẽ **tính tiền thêm**.

Chi phí:

\$0.05 / vCPU / giờ

------------------------------------------------------------------------

# Ví dụ chi phí CPU Credit

Giả sử server:

t3.large\
2 vCPU\
baseline = 60%

Server chạy:

CPU = 100%

Burst vượt:

100% - 60% = 40%

Chi phí thêm:

2 vCPU × 0.4 × \$0.05 ≈ \$0.04 / giờ

Một tháng:

≈ \$29

Trong khi upgrade instance:

m6i.large ≈ \$70

Chênh:

:   \$10

Nhưng:

-   CPU ổn định
-   không credit
-   performance tốt hơn

------------------------------------------------------------------------

# Case Study Production (thực tế)

Một hệ thống SaaS chạy:

-   Laravel API
-   Redis
-   MySQL RDS

Ban đầu deploy:

t3.large

Metrics sau 2 tuần:

CPU avg = 75%\
CPU peak = 95%\
CPUCreditBalance ≈ 0

Triệu chứng:

-   API response chậm
-   worker queue backlog
-   billing tăng

## Root Cause

CPU \> baseline (60%) liên tục

Instance luôn:

burst mode

CPU credit bị tiêu hết mỗi ngày.

Sau đó instance chạy:

unlimited burst

AWS bắt đầu charge **surplus credit**.

------------------------------------------------------------------------

# Giải pháp

Team chuyển instance:

t3.large → m6i.large

Kết quả:

  Metric         Before           After
  -------------- ---------------- --------
  CPU avg        75%              45%
  latency        250ms            120ms
  credit usage   high             none
  cost           \~\$60 + burst   \~\$70

Chi phí gần như **không đổi** nhưng:

-   performance tăng
-   system ổn định

------------------------------------------------------------------------

# Khi nào nên dùng T3

T3 phù hợp khi workload:

-   dev environment
-   web nhỏ
-   microservice idle nhiều
-   spike traffic ngắn

Ví dụ:

CPU bình thường 5%\
đôi khi spike 70%

------------------------------------------------------------------------

# Khi nào KHÔNG nên dùng T3

Nếu:

CPU \> 60% liên tục

Tránh dùng T3 cho:

-   API server traffic lớn
-   background worker
-   batch processing
-   data pipeline

Nên dùng:

-   m6i
-   c6i
-   m7g

------------------------------------------------------------------------

# Best Practice AWS

## 1. Monitor CPU Credit

CloudWatch metrics:

-   CPUUtilization
-   CPUCreditBalance
-   CPUSurplusCreditBalance

Nếu thấy:

CPU \> baseline\
credit gần 0

→ instance đang **không tối ưu chi phí**

------------------------------------------------------------------------

## 2. Right-size instance

Ví dụ:

t3.large → m6i.large

------------------------------------------------------------------------

## 3. Dùng Graviton

Nếu app support ARM:

m7g.large

Savings:

20--40%

------------------------------------------------------------------------

## 4. Scale horizontally

Thay vì:

1 instance lớn

Có thể dùng:

2 instance nhỏ

kèm:

-   Auto Scaling
-   ALB

------------------------------------------------------------------------

# Quy tắc đơn giản

CPU \< 20% → T3 rất phù hợp\
CPU 20--60% → cân nhắc\
CPU \> 60% → chuyển M hoặc C series

------------------------------------------------------------------------

# Kết luận

CPU Credit là cơ chế rất hay của AWS để tối ưu chi phí cho workload
burst.

Nhưng nếu dùng sai cách, nó có thể:

-   làm server chậm
-   làm bill AWS tăng

Hiểu rõ **CPU Credit + Baseline CPU** sẽ giúp bạn:

-   chọn đúng instance
-   tiết kiệm chi phí
-   tránh các vấn đề performance production.

------------------------------------------------------------------------

Nếu bạn đang vận hành hệ thống trên AWS, hãy kiểm tra ngay:

CloudWatch → CPUCreditBalance

Bạn có thể phát hiện ra **instance đang âm thầm đốt tiền mỗi ngày**.
