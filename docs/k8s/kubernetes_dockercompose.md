# Kubernetes qua góc nhìn Docker Compose

Nếu bạn đã quen Docker Compose, bạn đang có lợi thế lớn khi học Kubernetes.
Hai công cụ này đều giúp chạy ứng dụng containerized, nhưng chúng giải quyết vấn đề ở hai cấp độ khác nhau.

- **Docker Compose**: chạy nhiều container trên một máy, thường dùng cho local development hoặc môi trường nhỏ.
- **Kubernetes**: điều phối container trên một cụm nhiều máy, thường dùng cho staging/production.

Nói ngắn gọn: Kubernetes không thay thế Docker. Kubernetes thường thay thế vai trò của Docker Compose khi hệ thống cần chạy ổn định, scale được và tự phục hồi trong môi trường production.

## Thay đổi tư duy

Nếu Docker Compose trả lời câu hỏi:

> Làm sao chạy nhiều container trên một máy?

thì Kubernetes trả lời câu hỏi:

> Làm sao quản lý nhiều container trên nhiều máy một cách ổn định?

Ví dụ với Docker Compose:

```yaml
services:
  order:
    image: my-company/order-service
  payment:
    image: my-company/payment-service
  inventory:
    image: my-company/inventory-service
```

Bạn chạy:

```bash
docker compose up
```

Vậy là đủ cho local development.

Nhưng trong production, hệ thống có thể gồm:

- nhiều server
- nhiều instance của cùng một service
- Order Service
- Inventory Service
- Payment Service
- RabbitMQ
- Redis
- Postgres

Khi đó bạn phải xử lý thêm các tình huống như:

- một server chết
- container chết
- deploy version mới
- scale service lên nhiều instance
- rollback khi deploy lỗi
- health check
- load balancing
- quản lý cấu hình và secret

Docker Compose có thể restart container và scale thủ công ở mức đơn giản, nhưng nó không sinh ra để điều phối production trên nhiều máy. Đây là lúc Kubernetes trở nên hữu ích.

## Kubernetes giải quyết vấn đề gì?

Giả sử bạn có `Order Service` chạy 3 instance:

```text
order-1
order-2
order-3
```

Nếu `order-2` bị crash:

- Với Docker Compose: container có thể được restart nếu bạn cấu hình `restart`, nhưng vẫn chỉ trong phạm vi một máy.
- Với Kubernetes: controller phát hiện Pod chết, tạo Pod thay thế, rồi Service tiếp tục route traffic đến các Pod còn sống.

Luồng đơn giản:

```text
Pod chết
  -> Kubernetes phát hiện
  -> Kubernetes tạo Pod mới
  -> Traffic tiếp tục đi qua Service
```

Khả năng này thường được gọi là **self-healing**.

## Scale ứng dụng

Docker Compose hỗ trợ scale thủ công:

```bash
docker compose up --scale order=10
```

Đây chỉ là manual scaling. Docker Compose không tự động tăng/giảm số lượng container theo tải. Nếu muốn auto scaling, bạn cần dùng các công cụ như Docker Swarm hoặc Kubernetes.

> Nhưng nếu CPU tăng đột biến thì sao?

Docker Compose không tự scale theo CPU hoặc memory. Kubernetes có thể làm việc này bằng **Horizontal Pod Autoscaler** nếu cluster đã cài metrics phù hợp, thường là `metrics-server`.

Ví dụ:

```text
CPU > 80%
  -> HPA tăng số Pod
  -> Service route traffic đến nhiều Pod hơn
  -> tải giảm
```

## Kubernetes không phải Docker

Đây là nhầm lẫn rất phổ biến.

**Docker** là một bộ công cụ liên quan đến container. Trong đó có Docker Engine, Docker CLI, Dockerfile, Docker image, registry workflow, v.v.

**Kubernetes** không trực tiếp build image và cũng không phải container runtime. Kubernetes quản lý workload container thông qua một container runtime tương thích CRI, ví dụ:

- containerd
- CRI-O

Docker Engine từng được dùng phổ biến trong các cluster cũ, nhưng Kubernetes hiện nay thường chạy trực tiếp với `containerd`.

Có thể hình dung:

- Docker giúp bạn đóng gói và chạy container.
- Kubernetes giúp bạn điều phối container trên cả một cluster.

## Docker Compose vs Kubernetes

| Docker Compose | Kubernetes | Ghi chú |
| --- | --- | --- |
| `services` | `Deployment`, `StatefulSet`, `DaemonSet` | `Deployment` là loại phổ biến nhất cho stateless service. |
| `ports` | `Service`, `Ingress` | `Service` expose trong cluster; `Ingress` expose HTTP/HTTPS ra ngoài. |
| `environment` | `ConfigMap`, `Secret` | Config thường tách khỏi image. |
| `volumes` | `PersistentVolumeClaim` | Dùng khi dữ liệu cần tồn tại sau khi Pod bị thay thế. |
| `restart: always` | `Deployment` + `ReplicaSet` | Kubernetes duy trì số replica mong muốn. |
| `depends_on` | readiness probe, retry logic | Kubernetes không đảm bảo thứ tự app “ready” theo kiểu Compose. |
| `docker compose up` | `kubectl apply -f ...` | Kubernetes vận hành theo manifest khai báo trạng thái mong muốn. |

Lưu ý: Kubernetes hỗ trợ **rolling update** và **rollback** trực tiếp trong `Deployment`. Các chiến lược như **canary** hoặc **blue-green** thường được triển khai bằng cách phối hợp `Deployment`, `Service`, `Ingress`, service mesh hoặc công cụ như Argo Rollouts, Flagger.

## Kubernetes có khó không?

Có. Kubernetes giống một hệ điều hành cho cluster, nên nó có nhiều thành phần:

- API Server
- Scheduler
- Controller Manager
- etcd
- Kubelet
- Kube Proxy
- Container Runtime

Nhưng tin vui là Backend Engineer không cần học tất cả ngay từ đầu.

## Backend Engineer nên học gì?

### Tầng 1: Chạy application

Đây là phần đa số developer dùng thường xuyên:

- Pod
- Deployment
- Service
- ConfigMap
- Secret
- Namespace

### Tầng 2: Expose và lưu dữ liệu

- Ingress
- Volume
- PersistentVolume
- PersistentVolumeClaim

### Tầng 3: Vận hành production

- Liveness Probe
- Readiness Probe
- Startup Probe
- Rolling Update
- Rollback
- Horizontal Pod Autoscaler

### Tầng 4: Workload đặc biệt

- StatefulSet
- DaemonSet
- Job
- CronJob

## Học Kubernetes như học Docker Compose

Đừng bắt đầu bằng cách học định nghĩa kiểu Wikipedia. Hãy map các khái niệm bạn đã biết trong Docker Compose sang Kubernetes.

| Trong Docker Compose | Trong Kubernetes |
| --- | --- |
| `services` | `Deployment` |
| `ports` | `Service` hoặc `Ingress` |
| `environment` | `ConfigMap` hoặc `Secret` |
| `volumes` | `PersistentVolumeClaim` |
| `depends_on` | readiness probe + retry ở application |
| `restart: always` | `Deployment` duy trì replica |

Ví dụ một service trong Compose:

```yaml
services:
  order:
    image: my-company/order-service:1.0.0
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/order
```

Trong Kubernetes, cùng ý tưởng đó thường được tách thành nhiều resource:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
        - name: order
          image: my-company/order-service:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-secret
                  key: database-url
---
apiVersion: v1
kind: Service
metadata:
  name: order
spec:
  selector:
    app: order
  ports:
    - port: 8080
      targetPort: 8080
```

Điểm quan trọng: Kubernetes không chỉ “chạy container”. Bạn khai báo trạng thái mong muốn, ví dụ “tôi muốn luôn có 3 Pod của `order`”, còn Kubernetes liên tục cố gắng đưa cluster về trạng thái đó.

## Kết luận

Docker Compose rất tốt cho local development và hệ thống nhỏ. Kubernetes phù hợp khi bạn cần vận hành nhiều service trên nhiều máy, có self-healing, rolling update, rollback, service discovery, load balancing và autoscaling.

Cách học dễ nhất là đi từ những gì bạn đã biết:

```text
Compose service -> Kubernetes Deployment
Compose port    -> Kubernetes Service / Ingress
Compose env     -> Kubernetes ConfigMap / Secret
Compose volume  -> Kubernetes PVC
```

Khi nắm được mapping này, Kubernetes sẽ bớt giống một khối kiến thức khổng lồ và bắt đầu trở thành phiên bản production-grade của những gì bạn đã làm với Docker Compose.
