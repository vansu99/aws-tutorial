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
