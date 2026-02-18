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

## Framework Phân Tích Feature Trước Khi Triển Khai

Vấn đề của bạn rất phổ biến — phân tích được nhưng chỉ ở "happy path" và các case rõ ràng. Cái thiếu thường không phải kiến thức mà là thiếu một quy trình buộc bạn phải đào sâu hơn.

***Vấn đề cốt lõi: Tại sao hay miss edge case?***

Não người tự nhiên sẽ đi theo happy path vì đó là cách chúng ta được train để giải quyết vấn đề: "Làm sao để X hoạt động?" thay vì "Khi nào X sẽ không hoạt động?"

Để fix điều này, bạn cần chủ động thay đổi góc nhìn theo từng bước.

Mỗi data entity phải có một owner duy nhất. Service khác muốn thay đổi data đó phải đi qua owner service, không được bypass.

***Hệ thống 4 Lớp Phân Tích***

Lớp 1 — **WHAT**: Hiểu đúng yêu cầu trước khi nghĩ đến solution

Đây là bước bị skip nhiều nhất. Engineer thường đọc requirement xong nhảy ngay vào code.

Checklist bắt buộc:

```
□ Feature này giải quyết vấn đề gì của user?
□ "Done" trông như thế nào? Ai định nghĩa "done"?
□ Những gì KHÔNG nằm trong scope? (quan trọng không kém)
□ Có dependency nào với feature khác không?
□ Feature này sẽ thay đổi behavior hiện tại của ai?
```

***Mẹo: Viết lại requirement bằng lời của mình và confirm với PM/stakeholder trước khi làm. Bước này tốn 15 phút nhưng tiết kiệm được nhiều ngày làm lại.***

Lớp 2 — ***WHO***: Phân tích Actor và Context

Mỗi actor khác nhau tạo ra một bộ edge case hoàn toàn khác nhau.

```
Actor map:
├── User thông thường (happy path)
├── User lần đầu (chưa có data, onboarding)
├── User lâu năm (data cũ, legacy format)
├── User bị giới hạn quyền (permission edge case)
├── Admin/Internal user (bypass rules nào? audit log?)
├── Attacker (input manipulation, privilege escalation)
└── System/Bot (automated request, rate limit)
```

Ví dụ thực tế — Feature "User đổi email":

<img width="663" height="232" alt="image" src="https://github.com/user-attachments/assets/7a97a267-d5a3-4edd-9a3b-0ec89aa5db67" />


Lớp 3 — ***WHEN/WHERE***: Phân tích State và Context

Đây là nơi sinh ra nhiều edge case nhất. Hãy luôn hỏi: ***"Feature này chạy trong điều kiện nào?"***

***3.1 — State của data:***

```
Với mỗi entity liên quan, hỏi:
- Entity này có thể ở trạng thái nào? (active, deleted, suspended, pending...)
- Feature hoạt động thế nào với TỪNG trạng thái đó?
- Có state transition nào xảy ra concurrent không?
```

Ví dụ — Feature "Cancel Order":

```
Order states:
├── PENDING       → Cancel được, hoàn tiền không? (chưa charge)
├── PAYMENT_PROCESSING → Cancel được không? Race condition với payment
├── PAID          → Cancel được, refund flow thế nào?
├── SHIPPING      → Cancel được không? Cần contact shipper?
├── DELIVERED     → Không cancel được, chuyển sang Return flow
└── ALREADY_CANCELLED → Idempotent hay throw error?
```

***3.2 — Concurrent/Race condition:***

```
□ Điều gì xảy ra nếu user click 2 lần liên tiếp?
□ Điều gì xảy ra nếu 2 user cùng thao tác trên cùng resource?
□ Điều gì xảy ra nếu request timeout nhưng server đã xử lý xong?
```

***3.3 — External dependency:***

```
□ Third-party service (payment, email, SMS) bị chậm/down thì sao?
□ Database bị chậm → request queue up → memory blow thế nào?
□ Feature có chạy đúng khi bị retry không? (idempotency)
```

Lớp 4 — ***WHAT IF: Tư duy "Kẻ phá hoại"***

Đây là kỹ thuật quan trọng nhất. Sau khi phân tích xong, bạn đổi vai — không còn là người build nữa mà là người tìm cách break system.

Có 5 góc tấn công:

***① Input Attack — Data không như kỳ vọng***

```
□ Empty/null input ở mọi field
□ String cực dài (DoS via storage/processing)
□ Special characters: emoji, unicode, SQL injection, XSS payload
□ Negative number / số 0 ở những chỗ không expect
□ Date edge case: ngày 29/2, timezone khác nhau, timestamp quá khứ/tương lai
```

***② Volume Attack — Đúng nhưng quá nhiều***

```
□ User upload 1000 items thay vì 10 → pagination? timeout?
□ Bulk operation không có giới hạn → DB query timeout
□ Nested data quá sâu → stack overflow khi parse
```

***③ Order Attack — Đúng nhưng sai thứ tự***

```
□ User hoàn thành step 3 mà chưa làm step 1 (bypass URL)
□ Webhook đến không đúng thứ tự (event B đến trước event A)
□ User back/forward browser ở giữa multi-step flow
```

***④ Permission Attack — Đúng nhưng không có quyền***

```
□ Thay đổi ID trong request để access resource của người khác
□ Escalation: user thường gọi API của admin
□ Token hết hạn nhưng vẫn đang trong flow
```

***⑤ Timing Attack — Đúng nhưng sai thời điểm***

```
□ Feature bị gọi sau khi account đã bị delete
□ Coupon/promotion hết hạn đúng lúc đang checkout
□ Data thay đổi giữa lúc validate và lúc execute
```

**Quy trình thực tế — Template áp dụng ngay**

Mỗi khi nhận một feature, điền vào template này trước khi viết một dòng code:

```
## Feature: [Tên feature]

### 1. WHAT
- Mục tiêu thực sự:
- Định nghĩa "Done":
- Out of scope:
- Assumptions:

### 2. WHO (Actor analysis)
- [ ] Anonymous user
- [ ] New user (no data)
- [ ] Regular user
- [ ] Power user (large data)
- [ ] Restricted/suspended user
- [ ] Admin
- [ ] System/Bot

### 3. DATA FLOW
Origin → Transport → Storage → Consume → Lifecycle

Boundaries cần chú ý:
1.
2.

### 4. STATE ANALYSIS
Entities liên quan và các state:
- [Entity]: state_1, state_2, state_3
- Behavior tại mỗi state:

### 5. EDGE CASES (dùng 5 góc tấn công)
Input:
Volume:
Order:
Permission:
Timing:

### 6. FAILURE SCENARIOS
- Nếu [X] fail → hệ thống làm gì?
- Data có consistent không sau failure?
- User thấy gì?

### 7. OPEN QUESTIONS
(Những điều chưa rõ, cần confirm)
```

***Cách luyện tập để thành thói quen***

Vấn đề của kỹ năng này là biết nhưng không làm vì tốn thời gian. Đây là cách để nó trở thành reflex:

Tuần 1-2: Dùng template đầy đủ cho mọi feature dù nhỏ. Tốn thời gian nhưng cần thiết để xây thói quen.

Tuần 3-4: Thử phân tích feature của teammate trước khi đọc code của họ. Sau đó compare — bạn miss gì, họ miss gì.

Ongoing: Sau mỗi bug production, trace lại xem edge case đó thuộc nhóm nào trong 5 góc tấn công. Theo thời gian bạn sẽ nhận ra pattern của bản thân hay miss loại nào nhất.

## CHẤT LƯỢNG CODE

***Nguyên tắc cốt lõi***

`QC không phải lưới an toàn — họ là người verify. Lưới an toàn thực sự là quy trình của chính bạn.`

***Giai đoạn 1 — Trước khi bàn giao: Tự test có hệ thống***

***Bước 1: Chạy lại toàn bộ happy path***

Nghe đơn giản nhưng nhiều người skip. Quy tắc là tự ngồi dùng feature như một user thật, không phải như developer đang test.

```
□ Mở môi trường staging (không phải local)
□ Dùng data thật hoặc gần thật
□ Không dùng shortcut hay biết trước flow
□ Ghi lại video/screenshot nếu có thể
```

***Bước 2: Chạy qua checklist edge case từ lúc phân tích***

Nhớ lại template phân tích trước khi code? Bây giờ lấy nó ra và verify từng điểm:

```
□ Các actor khác nhau đã test chưa? (admin, new user, restricted user)
□ Các state của entity đã test chưa?
□ 5 góc tấn công đã check chưa?
  - Input: null, empty, quá dài, special chars
  - Volume: nhiều hơn bình thường
  - Order: sai thứ tự step
  - Permission: access resource của người khác
  - Timing: race condition cơ bản
```

***Bước 3: Regression check***

Feature mới hay vô tình break feature cũ. Check nhanh các flow liên quan:

```
□ Feature nào dùng chung component/API/DB table với feature mới?
□ Các flow đó còn hoạt động không?
□ Nếu có unit test/integration test → chạy và pass hết chưa?
```

***Giai đoạn 2 — Chuẩn bị tài liệu bàn giao***

Đây là thứ hầu hết developer bỏ qua nhưng lại tạo ra khác biệt lớn nhất. QC không thể test tốt nếu họ không hiểu đúng feature.

Test Scope Document

Viết một tài liệu ngắn gọn gồm:

```
## Feature: [Tên]
**Branch/Build:** staging-xxx
**Ngày bàn giao:** dd/mm/yyyy

### Mô tả ngắn
[1-2 câu feature làm gì]

### Scope (những gì cần test)
- [Flow 1]
- [Flow 2]

### Out of scope (những gì KHÔNG test lần này)
- [Intentionally excluded]

### Test accounts / Data cần chuẩn bị
- Account A: role admin, có order đang PENDING
- Account B: user mới, chưa verify email

### Known limitations / Accepted risks
- [Edge case nào bạn biết nhưng chấp nhận không handle lần này]
- [Behavior nào trông có vẻ lạ nhưng là đúng by design]

### Môi trường
- URL staging:
- Config đặc biệt cần bật:
- Third-party service đang dùng mock hay real?
```

> **Phần "Known limitations"** cực kỳ quan trọng. Nó giúp QC không tốn thời gian report bug mà thực ra là accepted behavior, và giúp bạn thể hiện rằng bạn đã nghĩ đến nó — chỉ là chưa handle sprint này.

---

## Giai đoạn 3 — Bug triage trước khi QC bắt đầu

Đừng để QC tự khám phá môi trường. Hãy **walkthrough cùng QC** 15-30 phút:
```
1. Demo happy path một lần
2. Giải thích các flow phức tạp / business logic đặc thù
3. Chỉ ra những vùng bạn đã test kỹ vs vùng còn uncertain
4. Align về test priority: cái nào critical, cái nào nice-to-have
```

Buổi này giúp QC test thông minh hơn thay vì test mò.

---

## Giai đoạn 4 — Trong quá trình QC test

### Phân loại bug đúng ngay từ đầu

Khi QC report bug, đừng để vào một đống. Phân loại ngay:

| Loại | Ý nghĩa | Hành động |
|---|---|---|
| **Blocker** | Không dùng được feature | Fix ngay trong ngày |
| **Critical** | Flow chính bị ảnh hưởng | Fix trước khi release |
| **Major** | Edge case quan trọng | Fix trong sprint này |
| **Minor** | UI/UX nhỏ, workaround có | Backlog, fix sau |
| **By design** | Behavior đúng, QC hiểu nhầm | Clarify và document |

### Khi nhận bug report — 3 câu hỏi trước khi fix
```
1. Reproduce được không? (nếu không → cần thêm info)
2. Root cause là gì? (fix đúng chỗ, không fix symptom)
3. Fix này có thể break thứ gì khác không? (regression risk)
```

---

## Giai đoạn 5 — Sau khi fix bug

Đây là bước hay bị bỏ qua nhất và gây ra **bug tái phát**:
```
□ Viết thêm test case cho bug vừa fix (prevent regression)
□ Hỏi: "Bug này xảy ra do miss ở bước phân tích nào?"
  → Input attack? Permission? State analysis?
□ Update lại template phân tích nếu có pattern mới
□ Nếu cùng loại bug xuất hiện >2 lần → fix ở process, không chỉ fix code
```

---

## Checklist tổng — Bàn giao feature
```
PRE-HANDOFF (Developer tự làm)
□ Test happy path trên staging
□ Test edge cases từ template phân tích
□ Regression check các flow liên quan
□ Code review (self-review hoặc peer review)
□ Không còn console.log / debug code
□ Environment variables đúng cho staging

HANDOFF DOCUMENT
□ Mô tả feature và scope
□ Test accounts và data đã chuẩn bị
□ Known limitations / accepted risks
□ Môi trường và config

HANDOFF MEETING
□ Demo happy path
□ Walkthrough business logic phức tạp
□ Align test priority

DURING QC
□ Bug được phân loại đúng ngay khi report
□ Blocker/Critical được fix trong ngày
□ Không fix bug mà không hiểu root cause

POST-QC
□ Regression test sau khi fix
□ Retrospective: miss edge case ở bước nào?
```

Một thay đổi nhỏ tạo ra khác biệt lớn nhất

Nếu chỉ làm được một thứ từ danh sách trên, hãy làm cái này:

`Viết "Known limitations" trước khi bàn giao.`

Nó buộc bạn phải suy nghĩ lại toàn bộ feature một lần nữa, tự phát hiện thêm vấn đề — và nó thể hiện với cả team rằng bạn hiểu sâu về feature của mình.

