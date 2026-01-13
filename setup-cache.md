# 📘 CloudFront Caching Guide

## User Upload Image (AWS Best Practice)

---

## 1. Mục tiêu & phạm vi

### 🎯 Mục tiêu

* Cache image user upload bằng **Amazon CloudFront**
* Giảm **Amazon S3 GET request** và **data transfer cost**
* Đảm bảo **user luôn thấy ảnh mới nhất khi update**
* Tránh bug phổ biến: *ảnh cũ do cache*

### 📦 Phạm vi áp dụng

* Frontend: React (`<img>`)
* Backend upload file trực tiếp (KHÔNG dùng Presigned URL)
* Origin: Amazon S3 (private bucket)
* CDN: Amazon CloudFront

---

## 2. Kiến trúc tổng thể

```
User Browser
   ↓
CloudFront (CDN + Cache)
   ↓
Amazon S3 (private bucket)
```

Nguyên tắc:

* Browser cache → CloudFront cache → S3
* Cache phụ thuộc **URL + Cache-Control**

---

## 3. Quy ước URL (CỰC KỲ QUAN TRỌNG)

### ✅ BẮT BUỘC: URL phải đổi khi content đổi

```
/uploads/u123/avatar_v1.jpg
/uploads/u123/avatar_v2.jpg
```

Hoặc:

```
/uploads/u123/20260113/avatar.jpg
```

### ❌ TUYỆT ĐỐI KHÔNG LÀM

```
/uploads/u123/avatar.jpg        # overwrite
avatar.jpg?ts=123               # query string versioning
```

> CloudFront cache theo **URL**, không theo nội dung file.

---

## 4. Backend upload lên S3 (BẮT BUỘC SET METADATA)

### 4.1 Metadata bắt buộc

Khi upload object lên S3, **PHẢI set**:

```http
Content-Type: image/jpeg
Cache-Control: public, max-age=86400
```

* `max-age=86400` = 1 ngày (khuyến nghị cho user upload image)

---

### 4.2 Ví dụ code (Python / Django)

```python
extra_args = {
    "ContentType": "image/jpeg",
    "CacheControl": "public, max-age=86400",
}

s3_client.upload_fileobj(
    file_obj,
    bucket_name,
    file_path,
    ExtraArgs=extra_args,
)
```

> ⚠️ Nếu không set `Cache-Control` → CloudFront cache không ổn định.

---

## 5. CloudFront – Cache Policy (BẮT BUỘC)

### 5.1 Cache key settings

```text
Headers: None
Cookies: None
Query strings: None
```

<img width="1852" height="206" alt="image" src="https://github.com/user-attachments/assets/9aaf7ee1-3103-4875-bf00-2f09ce6ca4bb" />

> Cache key càng nhỏ → cache hit càng cao.

---

### 5.2 TTL settings (khuyến nghị)

```text
Minimum TTL: 0
Default TTL: 86400
Maximum TTL: 31536000
```

<img width="1846" height="633" alt="image" src="https://github.com/user-attachments/assets/b0c62613-f85a-48a2-a6af-dcdf00290edf" />

---

## 6. CloudFront – Response Headers Policy (BEST PRACTICE)

### 6.1 NÊN cấu hình

**Security headers**:

```text
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Referrer-Policy: same-origin
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

**CORS (image public)**:

```text
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Headers: *
```

---

### 6.2 TUYỆT ĐỐI KHÔNG LÀM

❌ Không set `Cache-Control` trong Response Headers Policy

```text
Cache-Control: max-age=0, must-revalidate
```

> Cache-Control phải đến từ **S3**, không phải CloudFront Response Headers Policy.

---

## 7. CloudFront – Behavior checklist

* Origin: Amazon S3
* Viewer protocol policy: Redirect HTTP → HTTPS
* Cache policy: theo mục 5
* Response headers policy: theo mục 6
* Compression (Gzip/Brotli): ON
* Forward headers / cookies / query strings: NONE

---

## 8. Frontend (React)

Frontend **KHÔNG set cache**.

```jsx
<img
  src="https://cdn.example.com/uploads/u123/avatar_v2.jpg"
  loading="lazy"
  alt="avatar"
/>
```

> Cache do browser + CDN xử lý.

---

## 9. Flow update ảnh (KHÔNG invalidate)

1. User upload ảnh mới
2. Backend upload lên S3 với **path mới**
3. Backend trả về **URL mới**
4. Frontend render URL mới
5. Ảnh cũ tự hết TTL

✔ Không invalidate
✔ Không giảm TTL
✔ Không bug cache

---

## 10. Cách test đúng (TRÁNH hiểu nhầm)

### 10.1 Test chuẩn nhất – curl

```bash
curl -I https://cdn.example.com/image.jpg
curl -I https://cdn.example.com/image.jpg
```

Kết quả mong đợi:

```text
Lần 1: X-Cache: Miss from cloudfront
Lần 2: X-Cache: Hit from cloudfront
```

---

### 10.2 Lưu ý khi dùng Chrome DevTools

* `200 (from memory cache)` → Browser cache, **KHÔNG gọi CloudFront**
* `Miss from cloudfront` khi reload → Có thể là **revalidation**, không phải CDN lỗi

> Luôn dùng **curl hoặc Incognito** để kiểm tra CDN.

---

## 11. Checklist cuối (Production Ready)

* [x] URL đổi khi ảnh đổi
* [x] S3 object có Cache-Control
* [x] Content-Type đúng image
* [x] Cache Policy: query/header/cookie = none
* [x] Response Headers Policy không override cache
* [x] Không dùng invalidate cho avatar

---

## 12. Kết luận

> **Cache đúng = URL đúng + Cache-Control đúng + Cache key sạch**

Nếu 3 yếu tố này đúng → CloudFront cache ổn định, tiết kiệm cost, không bug UX.
