# AWS CONFIG

## Kiến trúc

```
AWS Resources
   ↓
AWS Config Recorder
   ├─ Record chỉ resource quan trọng
   ├─ Default = Daily
   └─ Override = Continuous cho resource critical
   ↓
S3 bucket riêng cho Config
   └─ Lifecycle sang IA / Glacier
   ↓
AWS Config Rules
   ├─ Change-triggered cho lỗi cần biết sớm
   └─ Periodic cho kiểm tra compliance định kỳ
   ↓
EventBridge
   ↓
SNS / Email / Slack
```

### Setup S3 Lifecycle

```
Bucket: riêng cho AWS Config
Versioning: không bật nếu không có yêu cầu đặc biệt
Lifecycle:
- 30 days  -> Standard-IA
- 90 days  -> Glacier Flexible Retrieval
- 365 days -> Glacier Deep Archive
- Expire   -> chỉ bật khi retention policy đã rõ
```
