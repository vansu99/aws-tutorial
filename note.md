***Thay "Best Practice" bằng "Right Trade-off".***

Không có giải pháp tốt nhất — chỉ có trade-off phù hợp với context. Khi ai đó đề xuất một giải pháp, đừng hỏi "cái này có phải best practice không?", hãy hỏi: Team có maintain được không? Nếu sai, recover mất bao lâu? Complexity cost là gì? (thêm Kafka vào có đáng không?)

Phân biệt rõ 3 chiều khi phân tích:
```   
Correctness: System làm đúng không? Edge case nào bị bỏ qua?
Scalability: Khi load x10, x100 — cái gì chết trước?
OperabilityTeam: có thể debug, monitor, rollback dễ không?
```

**Data Flow là xương sống**

Tại sao Data Flow quan trọng hơn Component?

Hầu hết engineer khi nhìn vào system sẽ nghĩ ngay đến "cần những service/component gì". Đây là tư duy từ trên xuống theo cấu trúc (structure-first).

Tech lead nhìn khác: "Data được sinh ra ở đâu, đi đâu, biến đổi thế nào, và ai chịu trách nhiệm ở từng điểm?"

Lý do đơn giản: ***Bug không sống trong component — nó sống ở boundary giữa các component.***

**Framework phân tích Data Flow**

Với bất kỳ feature nào, tôi đều đặt 5 câu hỏi theo thứ tự:

1. ORIGIN    — Data được tạo ra từ đâu? Ai là source of truth?
2. TRANSPORT — Data đi qua đâu? Có bị transform không?
3. STORAGE   — Data được lưu ở đâu? Dưới dạng nào?
4. CONSUME   — Ai đọc data này? Với mục đích gì?
5. LIFECYCLE — Data sống bao lâu? Khi nào bị xóa/archive?

Ví dụ thực tế: Hệ thống Order của E-commerce
Tưởng tượng bạn được giao thiết kế flow "User đặt hàng". Junior sẽ vẽ ngay:

`[Frontend] → [Order API] → [Database] → Done`

Tech lead sẽ vẽ data flow thực sự:
```
[User clicks "Buy"]
        ↓
[Frontend] — gửi: productId, quantity, addressId, paymentToken
        ↓
[Order Service] — validate stock, tính price tại thời điểm T
        ↓
        ├─→ [Payment Service] — charge tiền
        │         ↓ (success/fail)
        ↓
[Order DB] — lưu order với status = PENDING
        ↓
[Event Bus] — publish OrderCreated event
        ↓
        ├─→ [Inventory Service] — trừ stock
        ├─→ [Notification Service] — gửi email xác nhận
        └─→ [Analytics Service] — ghi nhận conversion

```

Boundary nào có thể vỡ?

Khi nhìn vào data flow trên, ngay lập tức thấy các danger zone:

Boundary 1: Order Service ↔ Payment Service

Payment thành công nhưng ghi DB lỗi → tiền bị trừ, order không tồn tại. Đây là bài toán distributed transaction kinh điển.

```
Câu hỏi phải hỏi:
- Nếu payment success nhưng DB timeout → xử lý thế nào?
- Idempotency key có được dùng không?
- Refund flow có được thiết kế từ đầu chưa?
```

Boundary 2: Order Service → Event Bus

Order đã lưu DB nhưng publish event thất bại → Inventory không được trừ, user vẫn có thể mua hàng đã hết.

```
Câu hỏi phải hỏi:
- Outbox pattern có được dùng không?
- Event có idempotent không nếu bị retry?
- Consumer nào là critical (Inventory) vs non-critical (Analytics)?
```

Boundary 3: Price tại thời điểm T

User thêm vào giỏ hàng lúc giá 100k, 10 phút sau admin đổi giá thành 150k, user checkout → tính giá nào?

```
Câu hỏi phải hỏi:
- Price được snapshot vào Cart hay Order?
- Price lock có TTL không?
- Ai là source of truth cho price: Product Service hay Order?
```

***Cách luyện tập tư duy này***

Bắt đầu với bất kỳ feature nào bạn đang làm, hãy vẽ ra theo template sau:

```
Feature: _______________

DATA ORIGIN:
- Ai tạo data này?
- Data có được validate ở source không?
- Format/schema là gì?

DATA TRANSPORT:
- Synchronous (HTTP) hay Async (Queue)?
- Nếu transport fail → retry? idempotent?
- Data có bị transform ở giữa không? Ai chịu trách nhiệm transform?

DATA STORAGE:
- Strong consistency hay eventual consistency?
- Nếu cùng data lưu nhiều chỗ → ai là source of truth?
- Schema migration có backward compatible không?

DATA CONSUME:
- Consumer đọc real-time hay batch?
- Nếu data sai → consumer có tự phát hiện không?
- N+1 query có xảy ra ở consumer không?

DATA LIFECYCLE:
- GDPR/compliance có yêu cầu xóa không?
- Soft delete hay hard delete?
- Archive strategy cho data cũ?
```

***Một nguyên tắc quan trọng***

`Đừng bao giờ để 2 service cùng write vào 1 table.`

Đây là dấu hiệu data ownership bị mờ nhạt — bạn sẽ không bao giờ biết ai là source of truth, và khi có bug, việc trace data flow gần như bất khả thi.

Mỗi data entity phải có một owner duy nhất. Service khác muốn thay đổi data đó phải đi qua owner service, không được bypass.


