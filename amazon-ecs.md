# Amazon ECS — Elastic Container Service

> **Mục tiêu:** Sau bài này, bạn sẽ hiểu ECS là gì, chạy container trên AWS như thế nào, khác gì Docker thủ công, khi nào dùng Fargate/EC2/EKS, và có thể tự triển khai một app Nginx đơn giản bằng ECS Fargate.

---

## 1. Khái niệm

### 1.1 Amazon ECS là gì?

**Amazon ECS — Elastic Container Service** là dịch vụ quản lý container của AWS.

Nói đơn giản:

> ECS giúp bạn chạy nhiều container trên AWS mà không cần tự viết script để start/stop/restart/scale container thủ công.

Nếu so với đời thực, hãy tưởng tượng ECS như **người quản lý đội xe container**:

| Thành phần thực tế                                                | Trong ECS                 |
| ----------------------------------------------------------------- | ------------------------- |
| Bãi xe container                                                  | ECS Cluster               |
| Bản hướng dẫn xe chở gì, cần tài xế nào, cần bao nhiêu nhiên liệu | Task Definition           |
| Một xe container đang chạy thật                                   | Task                      |
| Đội xe phải luôn có 3 xe đang chạy                                | ECS Service               |
| Container hàng hóa                                                | Docker Container          |
| Kho chứa mẫu container đã đóng gói                                | Amazon ECR                |
| Trạm điều phối traffic                                            | Application Load Balancer |

ECS không phải là “một container”. ECS là **hệ thống điều phối container**.

---

### 1.2 Container là gì?

**Container** là một gói đóng sẵn gồm:

- Source code/app đã build.
- Runtime cần thiết, ví dụ Node.js, PHP, Python, Java.
- Library/dependency.
- Config cơ bản để app chạy.
- Command khởi động app.

Ví dụ app Laravel cần PHP, Composer package, extension, nginx config. Thay vì setup lại từng thứ trên từng server, bạn đóng tất cả vào một Docker image.

Sau đó image này có thể chạy gần như giống nhau ở:

- Laptop developer.
- Server staging.
- Server production.
- ECS trên AWS.

---

### 1.3 Vì sao doanh nghiệp dùng container?

Vì container giải quyết một vấn đề kinh điển:

> “Máy em chạy được, sao lên server lại lỗi?”

Container giúp:

- **Deploy nhất quán:** dev/staging/prod chạy cùng image.
- **Rollback nhanh:** lỗi version mới thì chạy lại image cũ.
- **Scale dễ:** cần thêm capacity thì chạy thêm container.
- **Tách service rõ ràng:** web, worker, cron, API có thể chạy riêng.
- **Phù hợp microservices:** mỗi service một container.
- **Tối ưu CI/CD:** build image một lần, deploy nhiều môi trường.

---

### 1.4 ECS giúp chạy, quản lý, scale và giám sát container như thế nào?

ECS làm các việc chính:

| Nhu cầu            | ECS xử lý như thế nào                     |
| ------------------ | ----------------------------------------- |
| Chạy container     | Dựa vào Task Definition để chạy Task      |
| Giữ app luôn sống  | ECS Service tự thay task lỗi              |
| Scale              | Tăng/giảm số lượng task                   |
| Expose ra Internet | Kết hợp Application Load Balancer         |
| Pull image         | Lấy image từ ECR hoặc Docker Hub          |
| Ghi log            | Đẩy log container vào CloudWatch Logs     |
| Phân quyền         | Dùng IAM Task Role và Execution Role      |
| Health check       | Tự phát hiện task/container lỗi           |
| Deploy version mới | Rolling update hoặc blue/green deployment |

---

### 1.5 ECS khác gì so với tự chạy Docker thủ công trên EC2?

Giả sử bạn tự chạy Docker trên EC2:

```bash
docker run -d -p 80:80 nginx
```

Cách này đơn giản lúc demo. Nhưng production sẽ gặp nhiều câu hỏi:

- Container chết thì ai restart?
- Muốn chạy 10 container thì quản lý thế nào?
- Muốn deploy version mới không downtime thì làm sao?
- Muốn scale theo CPU/memory thì viết script à?
- Log nằm ở đâu?
- Container nào được quyền access S3/RDS/Secrets Manager?
- EC2 chết thì container có chạy lại ở nơi khác không?

ECS sinh ra để xử lý các việc đó.

| Tiêu chí         | Docker thủ công trên EC2   | Amazon ECS                       |
| ---------------- | -------------------------- | -------------------------------- |
| Start container  | Tự chạy `docker run`       | ECS tự chạy theo Task Definition |
| Restart khi lỗi  | Tự viết script/systemd     | ECS Service tự thay task lỗi     |
| Scale            | Thủ công                   | Auto Scaling                     |
| Load balancing   | Tự cấu hình                | Tích hợp ALB/NLB                 |
| Logging          | Tự gom log                 | CloudWatch Logs                  |
| IAM cho app      | Thường dùng EC2 Role chung | Task Role riêng cho từng service |
| Deployment       | Tự script                  | Rolling/Blue-Green               |
| Production-ready | Cần tự build nhiều thứ     | Có sẵn nhiều tính năng vận hành  |

---

### 1.6 ECS, EKS và AWS Fargate khác nhau thế nào?

| Dịch vụ     | Hiểu đơn giản                                | Khi nào dùng                                                |
| ----------- | -------------------------------------------- | ----------------------------------------------------------- |
| **ECS**     | Dịch vụ quản lý container native của AWS     | Muốn chạy container đơn giản, tích hợp AWS tốt, ít vận hành |
| **EKS**     | Kubernetes được AWS quản lý                  | Công ty đã dùng Kubernetes, cần hệ sinh thái K8s            |
| **Fargate** | Cách chạy container không cần quản lý server | Muốn chạy ECS/EKS mà không lo EC2                           |
| **ECR**     | Kho lưu Docker image                         | Lưu image private/public trên AWS                           |

Nói cực ngắn:

```text
ECS = Điều phối container
EKS = Kubernetes trên AWS
Fargate = Chạy container không cần quản lý EC2
ECR = Kho chứa Docker image
```

---

## 2. Các thành phần then chốt

### 2.1 Cluster

**Cluster** là nhóm tài nguyên dùng để chạy container.

Ví dụ:

```text
production-cluster
staging-cluster
batch-worker-cluster
```

Trong ECS, bạn phải tạo Cluster trước, sau đó mới chạy Task hoặc Service bên trong Cluster.

---

### 2.2 Task Definition

**Task Definition** là “bản thiết kế” mô tả container sẽ chạy như thế nào.

Nó chứa:

- Image name.
- CPU.
- Memory.
- Port mapping.
- Environment variables.
- IAM Role.
- Logging configuration.
- Secrets.
- Command/container entrypoint.
- Network mode.

Ví dụ dễ hiểu:

```text
Task Definition = Docker Compose file phiên bản ECS
```

---

### 2.3 Task

**Task** là một instance đang chạy thật từ Task Definition.

Ví dụ:

```text
Task Definition: nginx-web:revision-1
Task đang chạy:
- task-001
- task-002
- task-003
```

Nếu Task Definition là “bản thiết kế xe”, thì Task là “chiếc xe đang chạy thật”.

Một Task có thể có một hoặc nhiều container. Ví dụ:

```text
Task:
- app container
- sidecar log container
- monitoring agent container
```

---

### 2.4 Service

**Service** giữ cho số lượng Task luôn đúng như mong muốn.

Ví dụ bạn cấu hình:

```text
Desired tasks = 3
```

Nếu 1 task chết, ECS Service tự tạo task mới để quay lại 3 task.

Service thường dùng cho:

- Web app.
- API.
- Backend service.
- Worker chạy liên tục.

---

### 2.5 Container Image

**Container Image** là gói ứng dụng đã build sẵn.

Ví dụ:

```text
nginx:latest
php-app:v1.0.0
backend-api:2026-06-14
```

Image thường được lưu ở:

- Docker Hub.
- Amazon ECR.
- GitHub Container Registry.
- Private registry khác.

---

### 2.6 Amazon ECR

**Amazon ECR — Elastic Container Registry** là nơi lưu Docker image trên AWS.

Ví dụ workflow:

```text
Developer build image
        ↓
Push image lên ECR
        ↓
ECS pull image từ ECR
        ↓
Run container
```

ECR hỗ trợ:

- Private repository.
- Public repository.
- Image scanning.
- Lifecycle policy để xóa image cũ.
- Tích hợp IAM.
- Tích hợp ECS/Fargate rất mượt.

---

### 2.7 Launch Types: EC2 vs Fargate

ECS có nhiều cách chạy task. Phổ biến nhất là **EC2 Launch Type** và **Fargate Launch Type**.

| Tiêu chí         | ECS EC2 Launch Type                            | ECS Fargate Launch Type        |
| ---------------- | ---------------------------------------------- | ------------------------------ |
| Server           | Bạn tự tạo/quản lý EC2                         | AWS quản lý                    |
| Cần ECS Agent    | Có                                             | Không cần bạn quản lý          |
| Quản lý OS/Patch | Bạn chịu trách nhiệm                           | AWS chịu phần hạ tầng          |
| Control sâu      | Cao                                            | Ít hơn                         |
| Dễ vận hành      | Trung bình                                     | Rất dễ                         |
| Phù hợp          | Workload lớn, cần custom, GPU, tối ưu cost sâu | Web/API/worker thông thường    |
| Scale            | Scale EC2 + scale task                         | Chỉ scale task                 |
| Pricing          | Trả tiền EC2                                   | Trả tiền theo vCPU/memory task |

---

### 2.8 ECS Agent

**ECS Agent** là phần mềm chạy trên EC2 instance trong ECS EC2 Launch Type.

Nó giúp EC2 giao tiếp với ECS Control Plane.

Nhiệm vụ:

- Nhận lệnh chạy container.
- Pull image.
- Start/stop container.
- Report trạng thái task.
- Gửi log/metadata.
- Quản lý lifecycle container.

Với **Fargate**, bạn không cần quan tâm ECS Agent vì AWS quản lý hạ tầng bên dưới.

---

### 2.9 Service Discovery

Trong microservices, các service cần tìm thấy nhau.

Ví dụ:

```text
api-service cần gọi user-service
order-service cần gọi payment-service
```

Nếu IP task thay đổi liên tục, hardcode IP là thảm họa.

**Service Discovery** giúp service gọi nhau bằng tên DNS nội bộ, ví dụ:

```text
http://user-service.local
http://payment-service.local
```

Trên AWS, ECS thường dùng:

- AWS Cloud Map.
- Private DNS namespace.
- Service Connect.

---

### 2.10 Load Balancer Integration

ECS thường kết hợp với **Application Load Balancer — ALB** để expose app ra Internet.

Flow phổ biến:

```text
User
 ↓
Route 53
 ↓
Application Load Balancer
 ↓
ECS Service
 ↓
ECS Tasks
 ↓
Containers
```

ALB làm các việc:

- Nhận traffic HTTP/HTTPS.
- Health check task.
- Chỉ gửi traffic tới task healthy.
- Hỗ trợ path-based routing.
- Hỗ trợ host-based routing.
- Tích hợp HTTPS qua ACM.
- Tích hợp ECS Service.

Ví dụ routing:

```text
/api/*     → api-service
/admin/*   → admin-service
/web/*     → web-service
```

---

### 2.11 IAM Role: Task Role vs Execution Role

Đây là phần rất quan trọng.

#### Execution Role

**Execution Role** là role để ECS/Fargate chuẩn bị chạy container.

Nó thường dùng để:

- Pull image từ ECR.
- Ghi log vào CloudWatch Logs.
- Lấy secret từ Secrets Manager/SSM lúc start container.

Nói dễ hiểu:

```text
Execution Role = quyền cho ECS runtime chuẩn bị container
```

#### Task Role

**Task Role** là role mà code bên trong container sử dụng.

Ví dụ app của bạn cần:

- Read object từ S3.
- Send message vào SQS.
- Get secret từ Secrets Manager.
- Query DynamoDB.

Khi đó cấp quyền cho **Task Role**, không hardcode access key.

```text
Task Role = quyền cho ứng dụng bên trong container
```

#### So sánh nhanh

| Role           | Ai dùng?                         | Dùng để làm gì?                                    |
| -------------- | -------------------------------- | -------------------------------------------------- |
| Execution Role | ECS/Fargate runtime              | Pull image, ghi log, đọc secret khi khởi động      |
| Task Role      | Application code trong container | Gọi AWS API như S3, SQS, DynamoDB, Secrets Manager |

---

### 2.12 CloudWatch Logs

Container chạy trên ECS không nên ghi log “trôi nổi” trong máy.

Best practice là gửi stdout/stderr của container vào **CloudWatch Logs**.

Ví dụ log group:

```text
/ecs/nginx-demo
/ecs/backend-api
/ecs/worker
```

Lợi ích:

- Debug dễ.
- Search log theo thời gian.
- Tạo metric filter.
- Tạo alarm.
- Tích hợp dashboard.
- Không phụ thuộc vào việc task còn sống hay đã bị terminate.

---

## 3. Ví dụ thực hành: Triển khai Nginx bằng Amazon ECS Fargate

### 3.1 Kiến trúc mục tiêu

```text
Internet User
     ↓
Application Load Balancer - Public Subnet
     ↓
ECS Service - Fargate
     ↓
Nginx Container - Private Subnet
     ↓
CloudWatch Logs
```

Đây là kiến trúc production-friendly hơn so với việc đặt container public trực tiếp.

---

### 3.2 Các bước tóm tắt

#### Bước 1: Chuẩn bị Docker image

Có 2 cách:

**Cách nhanh:** dùng image public:

```text
nginx:latest
```

**Cách custom:** tự build image rồi push lên ECR.

---

#### Bước 2: Dockerfile đơn giản

Tạo file `Dockerfile`:

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

Tạo file `index.html`:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>ECS Fargate Demo</title>
  </head>
  <body>
    <h1>Hello from Amazon ECS Fargate!</h1>
    <p>Deployed with Nginx container.</p>
  </body>
</html>
```

---

#### Bước 3: Tạo ECR repository

Giả sử dùng region Tokyo:

```bash
aws ecr create-repository \
  --repository-name nginx-demo \
  --region ap-northeast-1
```

---

#### Bước 4: Login vào ECR

Thay `<account-id>` bằng AWS Account ID của bạn:

```bash
aws ecr get-login-password --region ap-northeast-1 \
| docker login \
  --username AWS \
  --password-stdin <account-id>.dkr.ecr.ap-northeast-1.amazonaws.com
```

---

#### Bước 5: Build image

```bash
docker build -t nginx-demo .
```

---

#### Bước 6: Tag image

```bash
docker tag nginx-demo:latest \
<account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/nginx-demo:latest
```

---

#### Bước 7: Push image lên ECR

```bash
docker push \
<account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/nginx-demo:latest
```

---

#### Bước 8: Tạo ECS Cluster

Ví dụ cluster:

```text
nginx-demo-cluster
```

Chọn:

```text
Infrastructure: AWS Fargate
```

---

#### Bước 9: Tạo Task Definition

Cấu hình cơ bản:

| Field          | Giá trị ví dụ                   |
| -------------- | ------------------------------- |
| Launch type    | Fargate                         |
| CPU            | 0.25 vCPU                       |
| Memory         | 0.5 GB                          |
| Network mode   | awsvpc                          |
| Container name | nginx                           |
| Image          | ECR image URI hoặc nginx:latest |
| Container port | 80                              |
| Log driver     | awslogs                         |

---

### 3.3 Task Definition mẫu

Ví dụ JSON rút gọn:

```json
{
  "family": "nginx-demo-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::<account-id>:role/nginxDemoTaskRole",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "<account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/nginx-demo:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "APP_ENV",
          "value": "production"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/nginx-demo",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---

#### Bước 10: Tạo CloudWatch Log Group

```bash
aws logs create-log-group \
  --log-group-name /ecs/nginx-demo \
  --region ap-northeast-1
```

---

#### Bước 11: Tạo ECS Service chạy trên Fargate

Cấu hình gợi ý:

| Field             | Giá trị                        |
| ----------------- | ------------------------------ |
| Cluster           | nginx-demo-cluster             |
| Service name      | nginx-demo-service             |
| Launch type       | Fargate                        |
| Desired tasks     | 2                              |
| Subnets           | Private subnets ở ít nhất 2 AZ |
| Security Group    | Chỉ allow traffic từ ALB       |
| Load Balancer     | Application Load Balancer      |
| Target Group      | Port 80                        |
| Health check path | `/`                            |

---

#### Bước 12: Security Group

Nên tách 2 Security Group.

##### ALB Security Group

Inbound:

| Type  | Port | Source    |
| ----- | ---: | --------- |
| HTTP  |   80 | 0.0.0.0/0 |
| HTTPS |  443 | 0.0.0.0/0 |

Outbound:

| Type        | Destination |
| ----------- | ----------- |
| All traffic | ECS Task SG |

##### ECS Task Security Group

Inbound:

| Type | Port | Source             |
| ---- | ---: | ------------------ |
| HTTP |   80 | ALB Security Group |

Outbound:

| Type             | Destination        |
| ---------------- | ------------------ | --------- |
| HTTPS            | 443                | 0.0.0.0/0 |
| hoặc All traffic | Tùy policy công ty |

Best practice: **không mở ECS Task trực tiếp từ Internet**.

---

#### Bước 13: Truy cập ứng dụng

Sau khi service ổn định, lấy DNS name của ALB:

```text
http://your-alb-name.ap-northeast-1.elb.amazonaws.com
```

Kết quả mong muốn:

```text
Hello from Amazon ECS Fargate!
```

---

#### Bước 14: Kiểm tra log trên CloudWatch

Vào:

```text
CloudWatch → Logs → Log groups → /ecs/nginx-demo
```

Bạn sẽ thấy log stream dạng:

```text
ecs/nginx/<task-id>
```

---

### 3.4 Command ECS CLI tham khảo

Đăng ký task definition:

```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --region ap-northeast-1
```

Tạo service:

```bash
aws ecs create-service \
  --cluster nginx-demo-cluster \
  --service-name nginx-demo-service \
  --task-definition nginx-demo-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-aaa,subnet-bbb],securityGroups=[sg-ecs-task],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:ap-northeast-1:<account-id>:targetgroup/nginx-demo-tg/xxxx,containerName=nginx,containerPort=80" \
  --region ap-northeast-1
```

---

## 4. AWS Best Practices cho ECS

### 4.1 Dùng Fargate khi muốn giảm công sức vận hành

Dùng Fargate nếu bạn không muốn quản lý:

- EC2 instance.
- OS patching.
- ECS Agent.
- Capacity của cluster.
- Auto Scaling EC2.

Phù hợp cho:

- Web app.
- API.
- Worker.
- Service vừa và nhỏ.
- Team chưa có nhiều kinh nghiệm vận hành container platform.

---

### 4.2 Dùng EC2 Launch Type khi cần kiểm soát sâu

Dùng ECS EC2 Launch Type nếu bạn cần:

- GPU.
- Custom AMI.
- Daemon đặc biệt trên host.
- Tối ưu chi phí ở scale lớn.
- Dùng Reserved Instance/Savings Plans/Spot Instance sâu hơn.
- Workload cần cấu hình host-level đặc biệt.

---

### 4.3 Không hardcode secret trong image hoặc environment variables

Không nên làm:

```dockerfile
ENV DB_PASSWORD=my-secret-password
```

Không nên hardcode trong source code:

```php
$password = "my-secret-password";
```

Nên dùng:

- AWS Secrets Manager.
- SSM Parameter Store.
- ECS secrets integration.

Ví dụ trong Task Definition:

```json
"secrets": [
  {
    "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:<account-id>:secret:prod/db/password"
  }
]
```

---

### 4.4 Dùng IAM Task Role theo Least Privilege

Mỗi ECS Service nên có Task Role riêng.

Ví dụ:

| Service         | Quyền nên có               |
| --------------- | -------------------------- |
| web-service     | Read secret, write log     |
| image-worker    | Read/write S3 image bucket |
| email-worker    | Send SES email             |
| payment-service | Access payment secret only |

Không nên dùng một role quá rộng kiểu:

```json
{
  "Action": "*",
  "Resource": "*"
}
```

---

### 4.5 Bật CloudWatch Logs cho container

Mọi container production nên có log.

Không có log thì debug giống như “chữa cháy trong phòng tối”.

Nên chuẩn hóa log group:

```text
/ecs/prod/web
/ecs/prod/api
/ecs/prod/worker
/ecs/stg/web
```

---

### 4.6 Sử dụng ECS Service Auto Scaling

Scale dựa trên:

- CPU utilization.
- Memory utilization.
- ALB Request Count Per Target.
- Custom CloudWatch Metric.

Ví dụ:

```text
CPU > 70% → tăng task
CPU < 30% → giảm task
```

Hoặc:

```text
RequestCountPerTarget > 1000/min → scale out
```

---

### 4.7 Đặt container trong private subnet

Best practice kiến trúc:

```text
Internet
  ↓
Public ALB
  ↓
Private ECS Tasks
```

Không nên:

```text
Internet
  ↓
ECS Task public IP
```

Vì task public trực tiếp làm tăng attack surface.

---

### 4.8 Dùng ECR image scanning

Bật scan image để phát hiện package/library có CVE.

Nên có quy trình:

```text
Build image
  ↓
Scan image
  ↓
Nếu critical vulnerability → không deploy
  ↓
Nếu pass → deploy ECS
```

---

### 4.9 Gắn tag cho Cluster, Service, Task

Tag giúp quản lý cost và vận hành:

```text
Environment=production
Project=video-app
Owner=backend-team
CostCenter=media-platform
Service=api
```

Khi bill tăng, bạn có thể biết service nào gây cost.

---

### 4.10 Dùng Health Check để ECS tự thay task lỗi

ALB health check nên có endpoint rõ ràng:

```text
/health
```

Ví dụ response:

```json
{
  "status": "ok"
}
```

Không nên health check vào endpoint nặng như:

```text
/report/monthly/export
```

---

### 4.11 Multi-AZ để tăng High Availability

Nên chạy ECS Service ở ít nhất 2 AZ:

```text
ap-northeast-1a
ap-northeast-1c
```

Nếu một AZ lỗi, task ở AZ khác vẫn phục vụ được.

---

### 4.12 Deployment strategy: Rolling update hoặc Blue/Green

#### Rolling update

ECS thay task cũ bằng task mới dần dần.

Ví dụ:

```text
4 task cũ
→ chạy thêm 2 task mới
→ health check OK
→ stop 2 task cũ
→ tiếp tục cho đến khi xong
```

Phù hợp hầu hết app.

#### Blue/Green deployment

Có 2 môi trường:

```text
Blue = version hiện tại
Green = version mới
```

Sau khi Green pass test, chuyển traffic sang Green.

Phù hợp hệ thống quan trọng, cần rollback nhanh.

---

## 5. Các use case thực tế

### Kịch bản 1: Chạy Web Application dạng container

#### Bối cảnh

Công ty có web app Laravel/Node.js/Java cần deploy production.

#### Kiến trúc

```text
Route 53
  ↓
Application Load Balancer
  ↓
ECS Service Auto Scaling
  ↓
Fargate Tasks
  ↓
RDS / ElastiCache / S3
```

#### Vì sao dùng ECS?

- Deploy app bằng image.
- Scale theo CPU/request.
- Log vào CloudWatch.
- Không quản lý server nếu dùng Fargate.
- Dễ rollback version.

#### Ví dụ thực tế

```text
api.example.com → ECS service backend-api
admin.example.com → ECS service admin
```

---

### Kịch bản 2: Chạy background worker xử lý queue

#### Bối cảnh

App có các job chạy nền:

- Xử lý video.
- Resize ảnh.
- Gửi email.
- Sync data.
- Generate report.
- Consume SQS message.

#### Kiến trúc

```text
SQS Queue
  ↓
ECS Worker Service
  ↓
Fargate Tasks
  ↓
S3 / RDS / SES / MediaConvert
```

#### Vì sao dùng ECS?

- Worker đóng gói thành container.
- Scale theo số lượng message trong SQS.
- Job lỗi thì task/container có thể restart.
- Không cần chạy worker cố định trên EC2.

#### Ví dụ

```text
video-processing-worker
email-sending-worker
csv-import-worker
```

---

### Kịch bản 3: Microservices architecture

#### Bối cảnh

Một hệ thống lớn được tách thành nhiều service nhỏ:

```text
user-service
order-service
payment-service
notification-service
search-service
```

#### Kiến trúc

```text
ALB
 ↓
frontend-service
 ↓
api-gateway-service
 ↓
user-service
order-service
payment-service
notification-service
```

Các service nội bộ tìm nhau qua:

```text
Service Discovery / Cloud Map / Service Connect
```

#### Vì sao dùng ECS?

- Mỗi service deploy độc lập.
- Mỗi service scale độc lập.
- Mỗi service có IAM Task Role riêng.
- Lỗi một service không kéo sập toàn hệ thống.
- Dễ rollback từng service.

---

## Tóm tắt nhanh để nhớ

```text
ECS = Dịch vụ quản lý và chạy container trên AWS
ECR = Kho lưu Docker image
Task Definition = Bản thiết kế container
Task = Container/app đang chạy thật
Service = Giữ số lượng task luôn đúng và hỗ trợ scale/deploy
Cluster = Nhóm tài nguyên chạy task/service
Fargate = Chạy container không cần quản lý EC2
EC2 Launch Type = Chạy container trên EC2 do bạn quản lý
ALB = Cổng nhận traffic từ Internet vào ECS
CloudWatch Logs = Nơi xem log container
Task Role = Quyền cho app trong container
Execution Role = Quyền để ECS pull image, ghi log, lấy secret
```

---

## Lộ trình học ECS khuyến nghị

1. Hiểu Docker: image, container, Dockerfile, port, volume.
2. Học ECR: build, tag, push image.
3. Chạy ECS Fargate với image `nginx`.
4. Gắn ALB vào ECS Service.
5. Bật CloudWatch Logs.
6. Thêm Auto Scaling.
7. Thêm Secrets Manager/SSM Parameter Store.
8. Deploy app thật: Laravel/Node.js/Java.
9. Tách web service và worker service.
10. Nâng cấp lên CI/CD với GitHub Actions hoặc CodePipeline.

ECS học đúng cách thì không khó. Cứ nhớ một câu: **Docker đóng gói app, ECR lưu image, ECS chạy và quản lý container, Fargate giúp bạn khỏi quản lý server.**
