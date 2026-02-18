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
