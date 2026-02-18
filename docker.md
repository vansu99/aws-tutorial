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


## Docker Setup — Dev / Staging / Production

> Một bộ config duy nhất, chạy được trên cả 3 môi trường.

---

### Cấu trúc thư mục

```
my-project/
├── src/
├── Dockerfile
├── docker-compose.yml              # base — config chung
├── docker-compose.dev.yml          # override dev
├── docker-compose.staging.yml      # override staging
├── docker-compose.prod.yml         # override production
├── .dockerignore
├── .env.example                    # commit
├── .env                            # KHÔNG commit — dev local
├── .env.staging                    # KHÔNG commit — staging server
├── .env.prod                       # KHÔNG commit — production server
└── Makefile                        # shortcut command
```

---

### Dockerfile

```dockerfile
# ---- Stage 1: Cài dependencies ----
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# ---- Stage 2: Development ----
# Ưu tiên đơn giản, hot reload, không cần non-root user
FROM node:20-alpine AS dev
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

# ---- Stage 3: Builder (dùng chung cho staging + prod) ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# ---- Stage 4: Staging ----
FROM node:20-alpine AS staging
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]

# ---- Stage 5: Production ----
FROM node:20-alpine AS prod
WORKDIR /app
COPY --from=builder /app/dist ./dist

# Chỉ cài production dependencies — image nhỏ hơn
COPY package*.json ./
RUN npm ci --omit=dev

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

### docker-compose.yml — Base

Config chung, **không chứa thứ gì đặc thù** cho từng môi trường.

```yaml
services:
  api:
    build:
      context: .
    environment:
      - NODE_ENV=${NODE_ENV}
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME}
      - REDIS_URL=${REDIS_URL}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

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

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

---

### docker-compose.dev.yml

```yaml
services:
  api:
    build:
      target: dev                     # dùng stage dev trong Dockerfile
    ports:
      - "3000:3000"
      - "9229:9229"                   # Node.js debug port
    volumes:
      - .:/app                        # hot reload
      - /app/node_modules             # giữ node_modules của container
    environment:
      - NODE_ENV=development

  db:
    ports:
      - "5432:5432"                   # expose để dùng với TablePlus / DBeaver

  redis:
    ports:
      - "6379:6379"                   # expose để debug với redis-cli
```

---

### docker-compose.staging.yml

```yaml
services:
  api:
    build:
      target: staging                 # dùng stage staging trong Dockerfile
    ports:
      - "3000:3000"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    restart: unless-stopped
    # KHÔNG expose port ra ngoài

  redis:
    restart: unless-stopped
    # KHÔNG expose port ra ngoài
```

---

### docker-compose.prod.yml

```yaml
services:
  api:
    build:
      target: prod                    # dùng stage prod — image nhỏ nhất, bảo mật nhất
    ports:
      - "3000:3000"
    restart: always
    deploy:
      replicas: 2                     # chạy 2 instance
      resources:
        limits:
          cpus: "1"
          memory: 1G
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"

  db:
    restart: always
    # KHÔNG expose port ra ngoài

  redis:
    restart: always
    command: redis-server --requirepass ${REDIS_PASSWORD}  # bảo mật redis trên prod
    # KHÔNG expose port ra ngoài
```

---

### .dockerignore

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
.github
```

---

### .env.example

```env
# App
NODE_ENV=

# Database
DB_HOST=db
DB_PORT=5432
DB_USER=
DB_PASSWORD=
DB_NAME=

# Redis
REDIS_URL=redis://redis:6379
REDIS_PASSWORD=
```

---

## .env — Dev local

```env
NODE_ENV=development

DB_HOST=db
DB_PORT=5432
DB_USER=admin
DB_PASSWORD=dev_secret
DB_NAME=myapp_dev

REDIS_URL=redis://redis:6379
REDIS_PASSWORD=
```

---

## .env.staging

```env
NODE_ENV=staging

DB_HOST=db
DB_PORT=5432
DB_USER=admin
DB_PASSWORD=staging_strong_password
DB_NAME=myapp_staging

REDIS_URL=redis://redis:6379
REDIS_PASSWORD=staging_redis_pass
```

---

## .env.prod

```env
NODE_ENV=production

DB_HOST=db
DB_PORT=5432
DB_USER=admin
DB_PASSWORD=prod_very_strong_password
DB_NAME=myapp_prod

REDIS_URL=redis://redis:6379
REDIS_PASSWORD=prod_redis_strong_pass
```

---

### Makefile — Shortcut command

Tránh gõ lệnh dài, dễ nhầm môi trường.

```makefile
# ========== DEV ==========
dev-up:
	docker compose -f docker-compose.yml -f docker-compose.dev.yml --env-file .env up -d

dev-up-build:
	docker compose -f docker-compose.yml -f docker-compose.dev.yml --env-file .env up -d --build

dev-down:
	docker compose -f docker-compose.yml -f docker-compose.dev.yml down

dev-logs:
	docker compose -f docker-compose.yml -f docker-compose.dev.yml logs -f

dev-reset:
	docker compose -f docker-compose.yml -f docker-compose.dev.yml down -v

# ========== STAGING ==========
staging-up:
	docker compose -f docker-compose.yml -f docker-compose.staging.yml --env-file .env.staging up -d --build

staging-down:
	docker compose -f docker-compose.yml -f docker-compose.staging.yml down

staging-logs:
	docker compose -f docker-compose.yml -f docker-compose.staging.yml logs -f

# ========== PRODUCTION ==========
prod-up:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml --env-file .env.prod up -d --build

prod-down:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml down

prod-logs:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml logs -f
```

---

### Cách dùng

### Dev

```bash
# Clone repo lần đầu
git clone <repo>
cp .env.example .env        # điền giá trị dev vào

# Chạy
make dev-up-build           # lần đầu hoặc khi có thay đổi Dockerfile/dependencies
make dev-up                 # những lần sau

# Debug
make dev-logs

# Reset sạch database
make dev-reset
```

### Staging

```bash
# Trên staging server
cp .env.example .env.staging   # điền giá trị staging vào
make staging-up
make staging-logs
```

### Production

```bash
# Trên production server
cp .env.example .env.prod      # điền giá trị prod vào
make prod-up
make prod-logs
```

---

### So sánh 3 môi trường

| | Dev | Staging | Prod |
|---|---|---|---|
| Dockerfile stage | `dev` | `staging` | `prod` |
| Non-root user | ❌ | ✅ | ✅ |
| Hot reload | ✅ | ❌ | ❌ |
| Expose DB port | ✅ | ❌ | ❌ |
| Expose Redis port | ✅ | ❌ | ❌ |
| Resource limits | ❌ | ✅ nhỏ | ✅ lớn hơn |
| Replicas | 1 | 1 | 2+ |
| restart policy | ❌ | `unless-stopped` | `always` |
| Log rotation | ❌ | ✅ | ✅ |
| Redis password | ❌ | ✅ | ✅ |

---

### Checklist trước khi deploy

```
✅ .env.staging / .env.prod không commit lên git
✅ Tất cả .env.* đã có trong .gitignore
✅ Staging test pass trước khi deploy prod
✅ DB password đủ mạnh trên staging và prod
✅ Không expose port DB / Redis trên staging và prod
✅ Chạy đúng make command cho đúng môi trường
✅ docker compose ps — tất cả services đang healthy
```

## Docker — Xử Lý Các Usecase Thực Tế

> Những tình huống thực tế hay gặp nhất khi dùng Docker hàng ngày, kèm cách xử lý từng bước.

---

### Usecase 1 — Container khởi động nhưng app bị crash ngay

**Triệu chứng:**
```bash
docker compose ps
# api    Exit 1
```

**Xử lý:**
```bash
# Xem log để biết lý do crash
docker compose logs api

# Xem log chi tiết hơn, 100 dòng cuối
docker compose logs --tail=100 api

# Nếu container restart liên tục
docker compose logs -f api   # theo dõi realtime
```

**Nguyên nhân phổ biến:**

```bash
# 1. Thiếu biến môi trường
Error: DATABASE_URL is required
→ Kiểm tra .env, đảm bảo đã khai báo đủ biến

# 2. App start trước khi DB ready
Error: connect ECONNREFUSED 127.0.0.1:5432
→ Thêm depends_on + healthcheck (xem Usecase 2)

# 3. Port bị chiếm
Error: bind: address already in use
→ Xem Usecase 3
```

---

### Usecase 2 — App crash vì connect DB quá sớm

**Triệu chứng:**
```
api_1  | Error: connect ECONNREFUSED db:5432
api_1  | App exited with code 1
```

**Nguyên nhân:** `depends_on` chỉ đảm bảo container DB **start**, không đảm bảo Postgres **sẵn sàng nhận connection**.

**Xử lý — thêm healthcheck:**
```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy   # chờ DB healthy thật sự

  db:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 5s
      timeout: 5s
      retries: 5
```

**Kiểm tra health status:**
```bash
docker compose ps
# NAME     STATUS
# db       healthy      ← phải là healthy, không phải starting
# api      running
```

---

### Usecase 3 — Port bị conflict

**Triệu chứng:**
```
Error: bind: address already in use 0.0.0.0:5432
```

**Xử lý:**
```bash
# Tìm process đang dùng port
lsof -i :5432
# hoặc
sudo ss -tulpn | grep 5432

# Kill process đó
kill -9 <PID>

# Hoặc đổi port mapping trong docker-compose.dev.yml
ports:
  - "5433:5432"   # map sang port khác trên host
```

**Tip — hay gặp nhất:**
```bash
# Postgres local đang chạy song song với container
# → Tắt postgres local đi
sudo service postgresql stop      # Linux
brew services stop postgresql     # macOS
```

---

### Usecase 4 — Thay đổi code nhưng container không nhận

**Triệu chứng:** Sửa code nhưng app vẫn chạy code cũ.

**Xử lý:**

```bash
# Trường hợp 1: Dùng volumes hot reload nhưng không hoạt động
# → Kiểm tra volumes trong docker-compose.dev.yml
volumes:
  - .:/app              # map toàn bộ thư mục hiện tại
  - /app/node_modules   # PHẢI có dòng này

# Trường hợp 2: Thêm package mới không nhận
npm install axios
docker compose up -d --build   # bắt buộc --build khi đổi dependencies

# Trường hợp 3: Thay đổi Dockerfile không nhận
docker compose up -d --build   # rebuild image

# Trường hợp 4: Vẫn không nhận sau --build (cache cũ)
docker compose build --no-cache api
docker compose up -d
```

---

### Usecase 5 — Container chạy nhưng không vào được app

**Triệu chứng:** Container status `running` nhưng `localhost:3000` không load.

**Xử lý từng bước:**

```bash
# Bước 1: Kiểm tra container có thật sự running không
docker compose ps

# Bước 2: Kiểm tra port có được map đúng không
docker compose ps
# PORTS: 0.0.0.0:3000->3000/tcp  ← phải có dòng này

# Bước 3: Vào trong container kiểm tra app có đang lắng nghe không
docker compose exec api sh
netstat -tlnp | grep 3000
# hoặc
curl localhost:3000

# Bước 4: Kiểm tra app có bind đúng host không
# ❌ Sai — chỉ lắng nghe localhost bên trong container
app.listen(3000, 'localhost')

# ✅ Đúng — lắng nghe tất cả interface
app.listen(3000, '0.0.0.0')
```

---

### Usecase 6 — Hết dung lượng ổ cứng vì Docker

**Triệu chứng:**
```
No space left on device
```

**Xử lý:**
```bash
# Xem Docker đang chiếm bao nhiêu
docker system df

# Kết quả ví dụ:
# TYPE            SIZE      RECLAIMABLE
# Images          8.2GB     6.1GB (74%)
# Containers      1.2GB     1.1GB (91%)
# Volumes         4.5GB     0B
# Build Cache     2.3GB     2.3GB

# Dọn dẹp an toàn — chỉ xóa thứ không dùng
docker system prune

# Dọn dẹp mạnh hơn — xóa cả images không dùng
docker system prune -a

# Chỉ xóa build cache
docker builder prune

# Xóa images cũ (dangling)
docker image prune

# ⚠️ Xóa tất cả volumes không dùng — CẨN THẬN mất data
docker volume prune
```

**Tip — setup tự dọn định kỳ:**
```bash
# Thêm vào crontab — chạy mỗi tuần
0 2 * * 0 docker system prune -f
```

---

### Usecase 7 — Build image chậm

**Xử lý — tối ưu cache:**

```dockerfile
# ❌ Sai — COPY . . trước làm cache miss liên tục
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install    # layer này bị rebuild mỗi khi đổi bất kỳ file nào

# ✅ Đúng — COPY package.json trước để cache npm install
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./    # chỉ copy file này
RUN npm install          # layer này chỉ rebuild khi package.json thay đổi
COPY . .                 # copy source code sau
```

```bash
# Kiểm tra thời gian build từng bước
docker build --progress=plain .

# Build song song nhiều stage (BuildKit)
DOCKER_BUILDKIT=1 docker compose build
# hoặc thêm vào .env
COMPOSE_DOCKER_CLI_BUILD=1
DOCKER_BUILDKIT=1
```

---

### Usecase 8 — Vào trong container để debug

```bash
# Vào container đang chạy
docker compose exec api sh        # Alpine Linux dùng sh
docker compose exec api bash      # Ubuntu/Debian dùng bash

# Chạy lệnh mà không cần vào trong
docker compose exec api node --version
docker compose exec db psql -U admin -d myapp

# Vào container đã stop (chạy container tạm)
docker compose run --rm api sh

# Xem thông tin chi tiết container
docker compose inspect api
```

---

### Usecase 9 — Reset môi trường khi bị lỗi lạ

**Khi môi trường dev bị lỗi không rõ nguyên nhân**, làm theo thứ tự:

```bash
# Cấp độ 1 — Restart service
docker compose restart api

# Cấp độ 2 — Down và up lại
docker compose down
docker compose up -d

# Cấp độ 3 — Rebuild image
docker compose down
docker compose up -d --build

# Cấp độ 4 — Xóa sạch, reset hoàn toàn (mất data DB)
docker compose down -v          # xóa cả volumes
docker compose up -d --build

# Cấp độ 5 — Xóa luôn image, build từ đầu
docker compose down -v
docker compose build --no-cache
docker compose up -d
```

> **Rule:** Thử từng cấp độ, đừng nhảy thẳng lên cấp 5 — vừa mất thời gian vừa mất data.

---

### Usecase 10 — Database mất data sau khi down

**Triệu chứng:** Chạy `docker compose down` xong up lại thì DB trống.

**Nguyên nhân:** Quên khai báo volume cho DB.

```yaml
# ❌ Không có volume → data mất khi container bị xóa
db:
  image: postgres:15-alpine

# ✅ Có named volume → data persist
db:
  image: postgres:15-alpine
  volumes:
    - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:    # khai báo ở cuối file
```

**Backup DB trước khi làm gì đó nguy hiểm:**
```bash
# Backup
docker compose exec db pg_dump -U admin myapp > backup.sql

# Restore
docker compose exec -T db psql -U admin myapp < backup.sql
```

---

### Usecase 11 — Nhiều project dùng Docker cùng lúc bị conflict

**Triệu chứng:** Project B up lên thì port của Project A bị chiếm.

**Xử lý — mỗi project dùng port riêng:**

```yaml
# Project A — docker-compose.dev.yml
services:
  api:
    ports:
      - "3001:3000"
  db:
    ports:
      - "5433:5432"
```

```yaml
# Project B — docker-compose.dev.yml
services:
  api:
    ports:
      - "3002:3000"
  db:
    ports:
      - "5434:5432"
```

**Hoặc đặt project name để tránh conflict network:**
```bash
docker compose -p project-a up -d
docker compose -p project-b up -d
```

---

### Usecase 12 — Xem resource usage (CPU, RAM)

```bash
# Xem realtime tất cả container
docker stats

# Xem 1 service cụ thể
docker stats $(docker compose ps -q api)

# Snapshot một lần, không realtime
docker stats --no-stream
```

---

### Usecase 13 — CI/CD build image và push lên registry

```bash
# Build với tag cụ thể
docker build --target prod -t myapp:1.0.0 .
docker build --target prod -t myapp:latest .

# Push lên Docker Hub
docker push myorg/myapp:1.0.0
docker push myorg/myapp:latest

# Pull về và chạy trên server
docker pull myorg/myapp:1.0.0
docker compose up -d
```

**docker-compose.prod.yml khi dùng image từ registry:**
```yaml
services:
  api:
    image: myorg/myapp:${APP_VERSION:-latest}   # không build trên server
    # bỏ phần build đi
```

---

### Cheat Sheet — Lệnh xử lý sự cố nhanh

```bash
# Xem trạng thái tất cả container
docker compose ps

# Xem log realtime
docker compose logs -f [service]

# Vào trong container debug
docker compose exec [service] sh

# Restart 1 service
docker compose restart [service]

# Rebuild 1 service cụ thể
docker compose up -d --build [service]

# Xem resource usage
docker stats

# Dọn dẹp Docker
docker system prune -a

# Reset hoàn toàn (⚠️ mất data)
docker compose down -v && docker compose up -d --build
```

---

### Bảng tóm tắt — Lỗi hay gặp

| Lỗi | Nguyên nhân | Cách fix |
|---|---|---|
| `Exit 1` ngay khi start | Crash do thiếu env hoặc lỗi code | `docker compose logs api` |
| `ECONNREFUSED db:5432` | App start trước DB ready | `depends_on` + `healthcheck` |
| `address already in use` | Port bị chiếm | `lsof -i :PORT` rồi kill |
| Sửa code không nhận | Thiếu volume hoặc thiếu `--build` | Kiểm tra volumes, thêm `--build` |
| `No space left on device` | Docker chiếm đầy disk | `docker system prune -a` |
| DB mất data sau `down` | Thiếu named volume | Thêm volume vào compose file |
| Build chậm | Layer order sai, cache miss | Copy `package.json` trước `COPY . .` |
| `localhost:3000` không load | App bind sai host | Đổi thành `0.0.0.0` |

