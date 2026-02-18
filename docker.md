# DOCKER

### Cách dùng `docker compose up` và `docker compose down`

Hiểu đơn giản thôi: up = bật, down = tắt + dọn dẹp

`docker compose up`

Dùng khi bạn muốn khởi động toàn bộ stack (app, db, redis, nginx,...).

```
docker compose up          # chạy, nhưng chiếm terminal
docker compose up -d       # chạy background (detached) — dùng cái này là chủ yếu
docker compose up --build  # rebuild image trước khi chạy (khi có thay đổi Dockerfile)
```

Dùng khi:

+ Bắt đầu làm việc buổi sáng
+ Sau khi pull code về, muốn chạy lại môi trường
+ Vừa thay đổi Dockerfile hoặc dependencies (package.json, requirements.txt)

`docker compose down`

Dùng khi muốn dừng và xóa containers (network cũng bị xóa theo).

```
docker compose down           # dừng + xóa containers & network
docker compose down -v        # + xóa luôn volumes (database sẽ mất data!)
docker compose down --rmi all # + xóa luôn images (ít dùng)
```

Dùng khi:

+ Xong việc, muốn giải phóng tài nguyên
+ Môi trường bị lỗi lạ, muốn reset sạch
+ Trước khi switch branch có thay đổi lớn về infrastructure

***Lưu ý: Dùng stop/start nếu muốn tạm dừng nhanh. Dùng down khi muốn "làm mới" hoàn toàn. Còn down -v thì cẩn thận — database sẽ bay màu.***

### Thao tác dùng hàng ngày — theo thứ tự thực tế

1. `docker compose up -d` + `docker compose down`

Đây là cặp đôi dùng nhiều nhất, gần như mỗi ngày.

```
# Sáng vào làm
git pull origin develop
docker compose up -d

# Tối về / hết giờ
docker compose down
```

Lý do ***down*** thay vì ***stop***: giải phóng port, network sạch hơn, tránh conflict lần sau up lên.

2. `docker compose up -d --build`

Dùng khi có thay đổi dependencies hoặc Dockerfile.

```
# Vừa thêm thư viện mới
npm install axios        # hoặc pip install requests
docker compose up -d --build   # bắt buộc phải --build, không thì vẫn chạy image cũ
```

Hay bị quên cái này lắm — thêm package rồi chạy `up -d` thường thôi, ngồi debug mãi không ra tại sao code không nhận thư viện mới.

3. `docker compose logs -f [service]`

Không phải up/down nhưng dùng liên tục khi debug.

```
docker compose logs -f api       # xem log realtime của service "api"
docker compose logs -f api db    # xem nhiều service cùng lúc
docker compose logs --tail=50 api  # chỉ xem 50 dòng cuối
```

4. `docker compose restart [service]`

Khi chỉ muốn restart 1 service cụ thể, không đụng cái khác.

```
# Config nginx thay đổi, chỉ cần restart nginx thôi
docker compose restart nginx

# Không cần down cả stack chỉ vì 1 service
```

***Workflow thực tế của 1 ngày làm việc***

```
# Sáng
git pull
docker compose up -d

# Trong ngày — thêm package mới
docker compose up -d --build

# Trong ngày — đang debug, xem log
docker compose logs -f api

# Trong ngày — fix config 1 service
docker compose restart nginx

# Tối
docker compose down
```

Tip thực tế: Tạo alias trong `.zshrc / .bashrc` cho nhanh:

```
alias up="docker compose up -d"
alias down="docker compose down"
alias dlog="docker compose logs -f"
```

Gõ `up` là xong, tiết kiệm cả năm 😄

