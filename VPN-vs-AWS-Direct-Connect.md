# VPN vs AWS Direct Connect

## 1. Tổng quan dễ hiểu

**VPN** và **AWS Direct Connect** đều dùng để kết nối hệ thống **On-Premises** của công ty với **AWS**.

**On-Premises** nghĩa là hệ thống đang đặt tại văn phòng công ty hoặc data center riêng, ví dụ:

- Server nội bộ
- Database nội bộ
- Hệ thống kế toán
- ERP
- File server
- Camera system
- Data center riêng

Mục tiêu là giúp hệ thống nội bộ có thể kết nối an toàn với AWS VPC.

```text
Văn phòng / Data Center công ty
↓
Kết nối mạng
↓
AWS VPC
↓
EC2 / RDS / S3 / Private Service
```

---

## 2. Vì sao công ty cần kết nối On-Premises với AWS?

Các lý do phổ biến:

- Công ty muốn **migrate dần lên AWS**
- Server nội bộ cần truy cập EC2/RDS trong AWS
- Ứng dụng trên AWS cần truy cập database vẫn còn ở công ty
- Backup dữ liệu từ công ty lên AWS
- Xây dựng mô hình **Hybrid Cloud**: một phần ở công ty, một phần trên AWS
- Đồng bộ dữ liệu giữa data center và AWS

Ví dụ:

Công ty có database kế toán đang đặt ở văn phòng, nhưng ứng dụng web mới chạy trên EC2 trong AWS. Khi đó, EC2 cần kết nối về database nội bộ.

---

# 3. AWS VPN là gì?

**AWS VPN** là kết nối bảo mật giữa mạng công ty và AWS thông qua **Internet public**, nhưng dữ liệu được **mã hóa**.

Nói đơn giản:

> VPN giống như tạo một “đường hầm bảo mật” đi qua Internet.

```text
Office Network
↓
Internet public
↓
VPN tunnel mã hóa
↓
AWS VPC
```

Dù dữ liệu đi qua Internet, dữ liệu vẫn được mã hóa nên người ngoài không đọc được nội dung.

---

## 4. Các loại VPN trong AWS

## 4.1 Site-to-Site VPN

**Site-to-Site VPN** dùng để nối **cả mạng công ty** với **AWS VPC**.

Ví dụ:

Công ty có văn phòng tại Việt Nam, muốn server trong văn phòng truy cập private EC2 trong AWS.

```text
Mạng công ty
↓
Router / Firewall / VPN Device
↓
AWS Site-to-Site VPN
↓
AWS VPC
↓
Private EC2 / RDS
```

Phù hợp khi:

- Công ty cần kết nối mạng nội bộ với AWS
- Cần kết nối nhanh
- Traffic không quá lớn
- Muốn tiết kiệm chi phí

---

## 4.2 Client VPN

**Client VPN** dùng cho **người dùng cá nhân** kết nối vào AWS.

Ví dụ:

Nhân viên làm việc từ xa cần truy cập server private trong AWS.

```text
Laptop nhân viên
↓
AWS Client VPN
↓
AWS VPC
↓
Private Server
```

Phù hợp khi:

- Nhân viên remote cần truy cập hệ thống private
- Admin cần truy cập EC2 private
- Không muốn mở SSH/RDP public ra Internet

---

# 5. AWS Direct Connect là gì?

**AWS Direct Connect** là kết nối mạng riêng, chuyên dụng từ công ty hoặc data center đến AWS.

Khác với VPN, Direct Connect **không đi qua Internet public**.

Nói đơn giản:

> Direct Connect giống như thuê một đường truyền riêng từ công ty đến AWS.

```text
Data Center công ty
↓
Đường truyền riêng
↓
AWS Direct Connect Location
↓
AWS VPC
```

Direct Connect thường ổn định hơn, độ trễ thấp hơn, tốc độ đều hơn so với VPN.

---

# 6. Lợi ích của Direct Connect

AWS Direct Connect phù hợp với hệ thống lớn vì:

- Độ trễ thấp hơn
- Hiệu năng ổn định hơn
- Không phụ thuộc Internet public
- Phù hợp traffic lớn
- Phù hợp production workload quan trọng
- Dễ dự đoán network performance hơn VPN

Ví dụ:

Một công ty tài chính cần đồng bộ dữ liệu liên tục giữa data center nội bộ và AWS. Nếu dùng Internet public, tốc độ có thể dao động. Direct Connect giúp kết nối ổn định hơn.

---

# 7. So sánh VPN và AWS Direct Connect

| Tiêu chí | AWS VPN | AWS Direct Connect |
|---|---|---|
| Đường truyền | Đi qua Internet public | Đường truyền riêng |
| Bảo mật | Có mã hóa VPN | Private connection, có thể kết hợp mã hóa thêm |
| Tốc độ triển khai | Nhanh | Chậm hơn |
| Chi phí | Rẻ hơn | Đắt hơn |
| Hiệu năng | Phụ thuộc chất lượng Internet | Ổn định và predictable hơn |
| Độ trễ | Có thể thay đổi | Thấp và ổn định hơn |
| Phù hợp | Small/medium setup | Enterprise workload |
| Use case | Backup, dev/test, kết nối tạm thời | Production lớn, traffic cao, yêu cầu ổn định |

---

# 8. Ví dụ thực tế

## Ví dụ 1: Công ty nhỏ dùng VPN

Một công ty nhỏ có server kế toán đặt tại văn phòng. Trên AWS có EC2 chạy API nội bộ.

Yêu cầu:

- Server kế toán cần gọi API trên EC2
- Traffic không quá lớn
- Muốn setup nhanh
- Muốn tiết kiệm chi phí

Kiến trúc phù hợp:

```text
Office Server
↓
Site-to-Site VPN
↓
AWS VPC
↓
Private EC2
```

Trường hợp này dùng **AWS VPN** là hợp lý vì rẻ, nhanh và đủ dùng.

---

## Ví dụ 2: Công ty lớn dùng Direct Connect

Một ngân hàng hoặc công ty lớn có data center riêng. Database vẫn đặt ở data center, còn ứng dụng mới chạy trên AWS.

Yêu cầu:

- Kết nối ổn định 24/7
- Truyền dữ liệu lớn
- Độ trễ thấp
- Production workload quan trọng
- Không muốn phụ thuộc Internet public

Kiến trúc phù hợp:

```text
On-Premises Data Center
↓
AWS Direct Connect
↓
AWS VPC
↓
EC2 / RDS / Analytics System
```

Trường hợp này nên dùng **Direct Connect** vì ổn định và phù hợp hệ thống enterprise.

---

# 9. Kiến trúc production tốt nhất

Trong môi trường production lớn, best practice thường là dùng cả hai:

```text
On-Premises
↓
Direct Connect Primary
↓
AWS

On-Premises
↓
VPN Backup
↓
AWS
```

Ý nghĩa:

- **Direct Connect** là đường chính
- **VPN** là đường backup
- Nếu Direct Connect gặp sự cố, hệ thống vẫn có thể kết nối qua VPN

Đây là cách thiết kế tốt hơn cho hệ thống quan trọng.

---

# 10. Khi nào chọn VPN?

Chọn **AWS VPN** khi:

- Muốn setup nhanh
- Ngân sách thấp
- Traffic không quá lớn
- Dùng cho dev/test/staging
- Dùng trong giai đoạn migration tạm thời
- Công ty nhỏ hoặc vừa
- Cần kết nối bảo mật nhưng không yêu cầu performance quá cao

Ví dụ:

Công ty chỉ cần backup dữ liệu từ server nội bộ lên AWS S3 mỗi đêm.

---

# 11. Khi nào chọn Direct Connect?

Chọn **AWS Direct Connect** khi:

- Production workload quan trọng
- Cần latency ổn định
- Truyền dữ liệu lớn thường xuyên
- Công ty có data center riêng
- Cần hybrid cloud lâu dài
- Cần network performance predictable
- Hệ thống yêu cầu SLA cao hơn

Ví dụ:

Hệ thống ERP nội bộ cần đồng bộ dữ liệu gần realtime với Amazon RDS/Aurora trên AWS.

---

# 12. Best Practice theo AWS Well-Architected

## 12.1 Reliability

Nên thiết kế chịu lỗi:

- Dùng Direct Connect cho đường chính nếu workload quan trọng
- Dùng VPN làm backup
- Có redundant connection nếu yêu cầu HA cao
- Kiểm tra failover định kỳ
- Theo dõi tunnel status và connection state

---

## 12.2 Security

Cần bảo vệ dữ liệu và network:

- VPN dùng IPSec encryption
- Direct Connect là private connection nhưng vẫn nên cân nhắc encryption nếu dữ liệu nhạy cảm
- Dùng Security Group chặt chẽ
- Dùng NACL khi cần kiểm soát subnet-level
- Không mở SSH/RDP public nếu không cần
- Dùng IAM least privilege
- Bật CloudTrail để audit

---

## 12.3 Cost Optimization

Tối ưu chi phí:

- Không cần Direct Connect nếu traffic nhỏ
- VPN phù hợp khi muốn tiết kiệm chi phí
- Direct Connect đáng tiền khi traffic lớn và cần kết nối ổn định
- Theo dõi data transfer cost
- Dùng AWS Cost Explorer và AWS Budgets để giám sát chi phí

---

## 12.4 Operational Excellence

Cần vận hành rõ ràng:

- Monitor VPN tunnel status bằng CloudWatch
- Monitor Direct Connect connection state
- Có runbook xử lý khi tunnel down
- Có cảnh báo qua SNS/Email/ChatOps
- Test failover định kỳ
- Ghi lại sơ đồ network và route table

---

# 13. Tóm tắt dễ nhớ

| Câu hỏi | Nên chọn |
|---|---|
| Muốn rẻ, nhanh, đơn giản? | VPN |
| Muốn ổn định, mạnh, enterprise? | Direct Connect |
| Workload production quan trọng? | Direct Connect + VPN backup |
| Công ty nhỏ/vừa? | VPN |
| Data center lớn, traffic cao? | Direct Connect |
| Cần kết nối tạm thời khi migration? | VPN |
| Cần hybrid cloud lâu dài? | Direct Connect |

---

# 14. Cách nhớ nhanh

**VPN = đi qua Internet nhưng có mã hóa**

**Direct Connect = đường truyền riêng đến AWS**

Nói ngắn gọn:

> VPN giống như đi đường công cộng nhưng ngồi trong xe chống đạn.  
> Direct Connect giống như có đường riêng dành cho công ty bạn.

---

# 15. Kết luận

- **AWS VPN** phù hợp khi cần kết nối nhanh, chi phí thấp, traffic vừa phải.
- **AWS Direct Connect** phù hợp khi cần kết nối riêng, ổn định, hiệu năng cao.
- Với hệ thống production quan trọng, nên dùng **Direct Connect làm primary** và **VPN làm backup**.
- Lựa chọn đúng phụ thuộc vào: **chi phí, độ ổn định, hiệu năng, bảo mật và yêu cầu kinh doanh**.
