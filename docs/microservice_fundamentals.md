# Vì sao dùng microservice?

## Monolith

```text
+--------------------------------------+
|               Application            |
|                                      |
| Auth                                 |
| User                                 |
| Product                              |
| Order                                |
| Payment                              |
| Notification                         |
|                                      |
+--------------------------------------+
                 |
             One Database
```

Tất cả module nằm trong cùng một project.

### Ưu điểm

- Đơn giản
- Dễ debug
- Deploy nhanh
- Transaction dễ

### Khi hệ thống lớn lên

Nếu vẫn là monolith, thường sẽ gặp:

- Code rất lớn
- Build lâu
- Deploy toàn hệ thống
- Một bug ở Notification có thể làm sập cả Order
- Team đông dễ conflict liên tục

## Microservice

Ví dụ:

```text
User Service
Order Service
Inventory Service
Payment Service
Notification Service
```

Mỗi service độc lập:

- Source code riêng
- Database riêng
- Deploy riêng
- Scale riêng

## Khi nào nên dùng?

Không phải dự án nào cũng nên dùng microservice.

Thường phù hợp khi:

- Hệ thống lớn
- Nhiều team
- Cần scale từng phần
- Release liên tục
- Business domain rõ ràng

## Tại sao startup mới không nên dùng microservice?

Vì:

- Chi phí hạ tầng cao
- Cần DevOps
- Monitoring
- Service discovery
- Logging
- Tracing
- Deployment phức tạp

Nếu chỉ có vài module thì monolith thường nhanh và hiệu quả hơn.

## Ưu điểm / Nhược điểm

### Ưu điểm

1. Deploy độc lập
2. Scale độc lập

Ví dụ:

Vào Black Friday:

- Order: 1000 req/s
- Payment: 100 req/s

Ta có thể scale:

- Order x10
- Payment giữ nguyên

Nếu là monolith thì phải scale toàn bộ, rất tốn tiền.

3. Team độc lập

- Team A: Order
- Team B: Payment

Không phải merge chung một project.

4. Có thể dùng công nghệ khác nhau

### Nhược điểm

#### Network failure

Monolith:

```text
function()
```

Microservice:

- HTTP
- RabbitMQ
- gRPC

Network có thể:

- Timeout
- Retry
- Mất kết nối

#### Distributed transaction

Ví dụ:

- Order
- Payment
- Inventory

Không còn transaction kiểu:

```text
BEGIN
COMMIT
```

như trong một database.

Phải dùng:

- Saga
- Compensation

#### Debug khó

```text
Request
  ↓
Gateway
  ↓
Order
  ↓
Inventory
  ↓
Payment
  ↓
Notification
```

Một request đi qua nhiều service, nên nếu không có tracing thì rất khó debug.

#### Data consistency

Thường là:

- Eventual consistency

Thay vì:

- Strong consistency

#### Hạ tầng phức tạp

Cần:

- Gateway
- Registry
- Monitoring
- Config
- CI/CD
- Message queue
- Distributed logging

## Service boundary

### Câu hỏi

Làm sao chia service?

### Sai lầm phổ biến

- User table => User service
- Order table => Order service

### Cách đúng

Chia theo business capability hoặc bounded context.

Ví dụ e-commerce:

- Identity
- Catalog
- Order
- Inventory
- Payment
- Shipping
- Notification

Mỗi service quản lý một nhóm nghiệp vụ riêng.

Ví dụ Order service chịu trách nhiệm:

- Tạo order
- Hủy order
- Cập nhật trạng thái

Không xử lý:

- Thanh toán
- Email
- Tồn kho

## Database per service

### Vì sao?

Nếu dùng chung database:

- `SELECT`
- `UPDATE`
- `JOIN`

Thì các service sẽ phụ thuộc chặt vào schema của nhau.

Nếu Inventory đổi schema, Order có thể bị lỗi.

### Giao tiếp đúng

Không đọc trực tiếp DB của service khác.

Thay vào đó:

```text
Order
  ↓ REST
Inventory
```

hoặc:

```text
RabbitMQ Event
```

### Ví dụ

Sai:

```sql
SELECT quantity
FROM inventory_table
```

Đúng:

```http
GET /inventory
```

hoặc:

```text
InventoryReserved event
```

## Communication: Sync vs Async

### Synchronous

```text
Order
  ↓ REST
Payment
  ↓ Response
```

Order phải chờ Payment trả lời.

#### Ưu điểm

- Đơn giản
- Phản hồi ngay
- Dễ debug

#### Nhược điểm

- Phụ thuộc lẫn nhau
- Timeout
- Cascade failure

### Asynchronous

```text
Order
  ↓ RabbitMQ
Payment
  ↓ Notification
```

Order không cần chờ.

#### Ưu điểm

- Giảm coupling
- Chịu tải tốt
- Tăng khả năng mở rộng

#### Nhược điểm

- Khó debug hơn
- Eventual consistency
- Cần retry và idempotency

### Khi nào dùng?

#### Sync

- Login
- Lấy profile
- Validate token
- Tra cứu dữ liệu cần phản hồi ngay

#### Async

- Gửi email
- Notification
- Logging
- Audit
- Xử lý ảnh
- Sinh embedding cho LLM
- Recommendation
- Báo cáo

## API Gateway

```text
Client
  ↓
Gateway
  ↓
Order
  ↓
Payment
  ↓
User
```

> Gateway là điểm vào duy nhất của client.

### Nhiệm vụ

- Authentication
- Authorization
- Routing
- Rate limiting
- SSL termination
- Request aggregation
- Logging

Client không cần biết địa chỉ của từng service.

## Service discovery

Trong Kubernetes, địa chỉ IP của service có thể thay đổi sau mỗi lần scale hoặc restart.

Nếu gọi trực tiếp bằng IP, các service khác sẽ lỗi.

Service discovery giúp ánh xạ tên service, ví dụ `payment-service`, tới địa chỉ hiện tại. Trong Kubernetes, DNS nội bộ đảm nhiệm vai trò này; trong hệ sinh thái khác có thể dùng Consul hoặc Netflix Eureka.

## Config management

Config là toàn bộ thông tin có thể thay đổi giữa các môi trường nhưng không làm thay đổi logic của ứng dụng.

Không hard-code cấu hình.

Nếu hardcode thì bạn phải:

```text
Sửa code
  ↓
Commit
  ↓
Build
  ↓
Deploy lại
```

Trong khi chỉ đổi password.

### Đúng là:

```text
Application
  ↓ đọc biến môi trường
  ↓ kết nối DB
```

Khi chuyển sang production chỉ cần đổi:

```text
DB_HOST=prod-db.company.com
```

Không cần sửa code.

### Nên quản lý tập trung

Trong microservice, cấu hình nên được quản lý tập trung hoặc thông qua biến môi trường để dễ thay đổi theo môi trường `dev`, `staging`, `production` mà không cần sửa mã nguồn.

Ví dụ:

- `DB_HOST`
- `DB_USER`
- `JWT_SECRET`
- `REDIS_URL`
- `RABBITMQ_URL`

## Observability

Đây là khả năng quan sát và chẩn đoán hệ thống khi vận hành.

Gồm 4 thành phần chính:

- Logging: ghi lại log có cấu trúc từ từng service
- Metrics: theo dõi CPU, RAM, QPS, latency, error rate
- Tracing: theo dõi một request đi qua nhiều service bằng `trace_id`
- Alerting: cảnh báo khi có lỗi hoặc vượt ngưỡng

Một stack phổ biến:

- Logging: ELK Stack hoặc Loki
- Metrics: Prometheus + Grafana
- Tracing: Jaeger hoặc OpenTelemetry

## Những câu hỏi Tech Lead hay hỏi

- Tại sao không nên chia microservice theo từng bảng?
- Vì sao mỗi service nên có database riêng?
- Khi nào dùng REST, khi nào dùng RabbitMQ?
- Tại sao microservice thường chấp nhận eventual consistency thay vì strong consistency?
- Vì sao API Gateway không nên chứa business logic?
- Điều gì xảy ra nếu một service gọi đồng bộ qua 5 service khác và service cuối cùng bị timeout?
- Nếu bạn là Tech Lead, với một startup chỉ có 3 lập trình viên và một hệ thống CRUD đơn giản, bạn sẽ chọn Monolith hay Microservice? Vì sao?
