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


### Init source mới — làm theo thứ tự này

Bước 1 — Tạo `Dockerfile`

Định nghĩa image của app bạn trông như thế nào.

```
# Ví dụ Node.js
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm", "run", "dev"]
```

Bước 2 — Tạo `docker-compose.yml`

Định nghĩa toàn bộ stack — app + db + redis + bất cứ thứ gì cần.

```
version: '3.8'

services:
  api:
    build: .                    # build từ Dockerfile ở thư mục hiện tại
    ports:
      - "3000:3000"
    volumes:
      - .:/app                  # hot reload — map code local vào container
      - /app/node_modules       # giữ node_modules của container, không bị override
    environment:
      - NODE_ENV=development
    depends_on:
      - db

  db:
    image: postgres:15-alpine   # dùng image có sẵn, không cần Dockerfile
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data  # persist data

volumes:
  postgres_data:
```

---

### Bước 3 — Tạo `.dockerignore`

Tránh copy rác vào image, **giống `.gitignore`**.
```
node_modules
.git
.env
dist
build
*.log
```

Bước 4 — Tạo `.env`

Tách config ra khỏi code, không commit file này lên git.

```
DB_HOST=db
DB_PORT=5432
DB_USER=admin
DB_PASSWORD=secret
DB_NAME=myapp
```

Rồi reference trong `docker-compose.yml`:

```
environment:
  - DB_HOST=${DB_HOST}
  - DB_PASSWORD=${DB_PASSWORD}
```

***Bước 5 — Build và chạy lần đầu***

```
# Lần đầu tiên — phải build image
docker compose up -d --build

# Kiểm tra các container đã chạy chưa
docker compose ps

# Xem log nếu có lỗi
docker compose logs -f
```

---

## Cấu trúc file sau khi xong
```
my-project/
├── src/
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .env                  # không commit
├── .env.example          # commit cái này — để người khác biết cần những biến gì
├── .gitignore
└── package.json
```

---

## Checklist nhanh
```
✅ Dockerfile       — build image app
✅ docker-compose   — kết nối các services
✅ .dockerignore    — loại file rác khỏi image
✅ .env             — config môi trường (không commit)
✅ .env.example     — template cho teammate (có commit)
✅ docker compose up -d --build  — chạy lần đầu
```

Tip: Sau khi init xong, commit ngay `Dockerfile`, `docker-compose.yml`, `.dockerignore`, `.env.example` — để teammate clone về chỉ cần cp `.env.example` `.env` rồi `docker compose up -d --build` là chạy được ngay, không cần setup gì thêm.

