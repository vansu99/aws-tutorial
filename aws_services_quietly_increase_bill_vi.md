# AWS Services That Quietly Increase Your Cloud Bill

Dễ hiểu nhất: **AWS tính tiền theo kiểu “dùng bao nhiêu trả bấy nhiêu”, nhưng nhiều service vẫn tính phí dù bạn tưởng là không dùng nữa.** Vì vậy nếu tạo xong rồi quên xóa, bill vẫn tăng âm thầm mỗi ngày.

---

## 1. NAT Gateway

**Dùng để EC2 trong private subnet đi ra Internet.**

Vấn đề: NAT Gateway tính phí **theo giờ chạy** và **theo dung lượng data đi qua**.

Ví dụ:  
Bạn tạo NAT Gateway để server private subnet update package. Sau đó không dùng nữa nhưng không xóa. Dù không có traffic nhiều, nó vẫn tính phí theo giờ.

Cách xử lý:  
Chỉ dùng khi thật sự cần. Nếu môi trường dev/test ít dùng, có thể tắt/xóa khi không cần.

---

## 2. Elastic Load Balancer

**Dùng để phân phối traffic vào nhiều EC2.**

Vấn đề: ALB/NLB vẫn tính phí theo giờ, kể cả lúc không có user truy cập.

Ví dụ:

```text
User → ALB → EC2
```

Nếu website đã ngưng chạy nhưng ALB vẫn còn, bạn vẫn bị tính tiền.

Cách xử lý:  
Kiểm tra Load Balancer nào không còn target hoặc không còn traffic thì xóa.

---

## 3. EBS Snapshots

**Snapshot là bản sao lưu của EBS Volume.**

Vấn đề: Snapshot lưu trong AWS vẫn tính phí theo dung lượng.

Ví dụ:  
Bạn backup EC2 mỗi ngày, sau 6 tháng có hàng trăm snapshot cũ. EC2 có thể đã xóa rồi nhưng snapshot vẫn còn, bill storage vẫn tăng.

Cách xử lý:  
Dùng lifecycle policy để tự động xóa snapshot cũ, ví dụ giữ 7 ngày hoặc 30 ngày.

---

## 4. EC2 Instances

**Máy chủ ảo trên AWS.**

Vấn đề: EC2 đang chạy là tính phí liên tục. Nhiều người tạo server test rồi quên tắt.

Ví dụ:  
Bạn tạo EC2 để test Laravel trong 2 giờ, nhưng để chạy cả tháng. Bill vẫn tính như server production.

Cách xử lý:

```text
Không dùng → Stop hoặc Terminate
```

Lưu ý: Stop EC2 thì không tính tiền compute nữa, nhưng **EBS volume vẫn tính phí**.

---

## 5. Unattached EBS Volumes

**EBS Volume là ổ đĩa gắn vào EC2.**

Vấn đề: EC2 bị xóa nhưng ổ EBS còn lại, gọi là unattached volume. Ổ này vẫn tính tiền.

Ví dụ:  
Bạn terminate EC2 nhưng volume data 100GB vẫn còn. Không gắn vào server nào nhưng AWS vẫn tính phí lưu trữ.

Cách xử lý:  
Vào **EC2 → Volumes** → lọc trạng thái `available`. Nếu không cần thì xóa.

---

## 6. Data Transfer

**Chi phí truyền dữ liệu ra Internet hoặc giữa các Region.**

Vấn đề: Data transfer thường là khoản khó nhìn thấy nhưng có thể tăng rất nhanh.

Ví dụ:  
Server ở Tokyo gửi dữ liệu sang Singapore hoặc user tải nhiều video từ S3/EC2 ra Internet. Data out sẽ phát sinh phí.

Cách xử lý:  
Dùng CloudFront để cache nội dung gần user, tránh origin phải gửi dữ liệu lặp lại nhiều lần.

Ví dụ kiến trúc tốt hơn:

```text
User
 ↓
CloudFront
 ↓
S3 / ALB / EC2
```

---

## 7. CloudWatch Logs

**Dùng để lưu log ứng dụng, EC2, Lambda, container.**

Vấn đề: Log lưu càng nhiều, càng lâu thì càng tốn tiền.

Ví dụ:  
Ứng dụng Laravel ghi log debug liên tục. CloudWatch Logs giữ log mãi mãi không xóa. Sau vài tháng dung lượng log tăng rất lớn.

Cách xử lý:  
Set retention period, ví dụ:

```text
Dev/Test: giữ log 7–14 ngày
Production: giữ 30–90 ngày tùy yêu cầu
```

---

## 8. Elastic IP Address

**IP tĩnh public của AWS.**

Vấn đề: Elastic IP không gắn vào resource đang chạy vẫn có thể bị tính phí.

Ví dụ:  
Bạn tạo Elastic IP cho EC2, sau đó xóa EC2 nhưng quên release IP. IP đó vẫn nằm trong account và phát sinh cost.

Cách xử lý:  
Vào **EC2 → Elastic IPs** → kiểm tra IP nào không attached thì release.

---

## 9. S3 Storage

**Dùng để lưu file, ảnh, video, backup, log.**

Vấn đề: S3 rẻ, nhưng nếu dữ liệu tăng liên tục mà không có lifecycle thì bill sẽ tăng đều.

Ví dụ:  
Hệ thống upload ảnh/video lên S3 mỗi ngày. File cũ không ai truy cập nhưng vẫn nằm ở S3 Standard.

Cách xử lý:  
Dùng lifecycle policy:

```text
File mới → S3 Standard
Sau 30 ngày → S3 Standard-IA
Sau 90/180 ngày → Glacier
Không cần nữa → Delete
```

---

## 10. Auto Scaling Group cấu hình sai

**Auto Scaling giúp tự tăng/giảm EC2 theo traffic.**

Vấn đề: Nếu cấu hình sai min/desired capacity, hệ thống có thể giữ nhiều EC2 chạy dù không cần.

Ví dụ:  
Bạn set:

```text
Min = 5
Desired = 5
Max = 10
```

Dù ban đêm không có traffic, hệ thống vẫn giữ ít nhất 5 EC2 chạy.

Cách xử lý:  
Set min/desired hợp lý. Với môi trường dev/test có thể dùng schedule scaling để giảm server ngoài giờ làm việc.

---

## Tóm tắt dễ nhớ

| Service | Vì sao tăng bill? | Cách xử lý |
|---|---|---|
| NAT Gateway | Tính theo giờ và data | Xóa nếu không dùng |
| Load Balancer | Không có traffic vẫn tính phí | Xóa ALB/NLB không dùng |
| EBS Snapshot | Backup cũ vẫn tính storage | Set lifecycle/delete snapshot cũ |
| EC2 | Chạy liên tục là tính tiền | Stop/terminate khi không dùng |
| EBS Volume | Volume rời vẫn tính phí | Xóa volume trạng thái available |
| Data Transfer | Data out/cross-region tốn phí | Dùng CloudFront, tránh transfer dư |
| CloudWatch Logs | Log lưu lâu/tăng nhiều | Set retention |
| Elastic IP | IP không dùng vẫn tốn phí | Release IP dư |
| S3 | Dữ liệu tăng dần | Lifecycle sang IA/Glacier |
| Auto Scaling | Min/desired quá cao | Điều chỉnh scaling policy |

---

## Ví dụ thực tế

Bạn tạo một môi trường test:

```text
ALB
 ↓
2 EC2
 ↓
EBS Volume
Logs → CloudWatch
Files → S3
Private subnet → NAT Gateway
```

Sau khi test xong, bạn chỉ stop EC2 nhưng quên:

```text
ALB vẫn còn
NAT Gateway vẫn còn
EBS volume vẫn còn
Snapshot vẫn còn
CloudWatch Logs vẫn lưu mãi
S3 file test vẫn còn
Elastic IP vẫn chưa release
```

Kết quả: dù app không còn chạy, AWS vẫn có thể phát sinh bill.

---

## Best practice nên làm

Hàng tuần nên kiểm tra:

```text
Billing / Cost Explorer
EC2 running instances
Unattached EBS Volumes
EBS Snapshots cũ
NAT Gateways
Load Balancers không dùng
Elastic IP chưa gắn
CloudWatch Logs retention
S3 lifecycle
```

---

## Cách nhớ ngắn gọn

> AWS không đắt vì tạo resource. AWS đắt vì tạo xong rồi quên xóa.
