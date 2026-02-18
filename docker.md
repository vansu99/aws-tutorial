# DOCKER

## Cách dùng `docker compose up` và `docker compose down`

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


## Hướng Dẫn Docker Init — Production Ready

> Dành cho developer muốn setup Docker đúng cách ngay từ đầu, tránh các lỗi phổ biến khi đưa vào team hoặc production.

---

## Cấu trúc thư mục

```
my-project/
├── src/
├── Dockerfile
├── docker-compose.yml          # base config chung
├── docker-compose.dev.yml      # override cho dev
├── docker-compose.prod.yml     # override cho prod
├── .dockerignore
├── .env                        # KHÔNG commit
├── .env.example                # commit — template cho teammate
└── .gitignore
```

---

### Bước 1 — Dockerfile (Multi-stage)

Dùng **multi-stage build** để tách môi trường dev/prod, tránh image nặng và không an toàn.

```dockerfile
# ---- Stage 1: Cài dependencies ----
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# ---- Stage 2: Development ----
FROM node:20-alpine AS dev
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Tạo non-root user — tránh chạy với quyền root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["npm", "run", "dev"]

# ---- Stage 3: Production ----
FROM node:20-alpine AS prod
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Tại sao multi-stage?**

- Image production không chứa source code gốc, devDependencies, hay build tools
- Mỗi stage độc lập — dễ cache, build nhanh hơn
- Tách biệt rõ ràng giữa dev và prod

---

### Bước 2 — docker-compose.yml (Base)

File base chứa **config chung** cho cả dev lẫn prod.

```yaml
services:
  api:
    build:
      context: .
      target: dev             # mặc định build stage dev, prod sẽ override
    environment:
      - NODE_ENV=${NODE_ENV}
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME}
    depends_on:
      db:
        condition: service_healthy  # chờ DB thực sự sẵn sàng, không chỉ started

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

> **Lưu ý:** Không còn dùng `version: '3.8'` — Docker Compose v2 đã deprecated field này.

---

### Bước 3 — docker-compose.dev.yml (Override Dev)

```yaml
services:
  api:
    build:
      target: dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app                   # hot reload — map code local vào container
      - /app/node_modules        # giữ node_modules của container, không bị override
    environment:
      - NODE_ENV=development

  db:
    ports:
      - "5432:5432"              # expose port ra ngoài để dùng với DB client (TablePlus, DBeaver,...)
```

---

### Bước 4 — docker-compose.prod.yml (Override Prod)

```yaml
services:
  api:
    build:
      target: prod
    ports:
      - "3000:3000"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

  db:
    restart: unless-stopped
    # KHÔNG expose port 5432 ra ngoài trên production
```

---

### Bước 5 — .dockerignore

Tránh copy rác vào image — giảm build time và tăng bảo mật.

```
node_modules
.git
.env
.env.*
dist
build
coverage
*.log
.DS_Store
README.md
docker-compose*.yml
```

---

### Bước 6 — .env và .env.example

**.env** — không commit, chứa giá trị thật:

```env
NODE_ENV=development

DB_HOST=db
DB_PORT=5432
DB_USER=admin
DB_PASSWORD=supersecret
DB_NAME=myapp
```

**.env.example** — commit lên git, chứa template để teammate biết cần những biến gì:

```env
NODE_ENV=

DB_HOST=
DB_PORT=
DB_USER=
DB_PASSWORD=
DB_NAME=
```

---

### Bước 7 — Chạy lần đầu

```bash
# 1. Copy env
cp .env.example .env
# Sau đó điền giá trị vào .env

# 2. Build và chạy môi trường dev
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build

# 3. Kiểm tra các container
docker compose ps

# 4. Xem log nếu có lỗi
docker compose logs -f

# 5. Xem log của 1 service cụ thể
docker compose logs -f api
```

---

### Workflow hàng ngày

```bash
# Sáng — bật lên
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# Thêm package mới — bắt buộc phải --build
npm install axios
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build

# Debug realtime
docker compose logs -f api

# Restart 1 service mà không ảnh hưởng service khác
docker compose restart nginx

# Tối — tắt
docker compose down
```

---

### Tip: Tạo alias để gõ nhanh hơn

Thêm vào `.zshrc` hoặc `.bashrc`:

```bash
alias dc-dev="docker compose -f docker-compose.yml -f docker-compose.dev.yml"
alias dc-prod="docker compose -f docker-compose.yml -f docker-compose.prod.yml"

# Dùng
dc-dev up -d
dc-dev up -d --build
dc-dev down
dc-dev logs -f api
```

---

### Cheat sheet — Up / Down

| Command | Containers | Network | Volumes | Dùng khi |
|---|---|---|---|---|
| `up -d` | Tạo + chạy | Tạo | Giữ | Bắt đầu làm việc |
| `up -d --build` | Rebuild + chạy | Tạo | Giữ | Có thay đổi Dockerfile / dependencies |
| `stop` | Dừng (giữ lại) | Giữ | Giữ | Tạm dừng nhanh |
| `start` | Chạy lại | Giữ | Giữ | Tiếp tục sau `stop` |
| `restart [service]` | Restart 1 service | Giữ | Giữ | Fix config 1 service |
| `down` | Dừng + xóa | Xóa | Giữ | Kết thúc ngày làm việc |
| `down -v` | Dừng + xóa | Xóa | **Xóa** | Reset hoàn toàn — ⚠️ mất data DB |

---

### Checklist trước khi commit

```
✅ Dockerfile dùng multi-stage (dev + prod)
✅ Dockerfile chạy non-root user
✅ docker-compose.yml không có field version
✅ depends_on dùng condition: service_healthy
✅ .dockerignore đã có và đúng
✅ .env không có trong .gitignore → thêm vào ngay
✅ .env.example đã commit
✅ Production không expose port DB ra ngoài
✅ docker compose up -d --build chạy thành công
```

---

### Teammate clone về — chỉ cần 3 bước

```bash
git clone <repo>
cp .env.example .env        # điền các giá trị cần thiết
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

> Không cần cài Node, Postgres, hay bất cứ thứ gì khác trên máy local.

