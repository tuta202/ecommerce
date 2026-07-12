# System Design: Order, Payment, Inventory bất đồng bộ

Ta dùng một bài toán e-commerce điển hình:

```text
Client tạo đơn hàng
      |
      v
Order Service
      |
      +--> Inventory Service giữ hàng
      |
      +--> Payment Service thanh toán
      |
      +--> Shipping Service giao hàng
```

## Mục tiêu

- Không mất đơn hàng
- Không trừ tiền hai lần
- Không bán vượt tồn kho
- Có thể phục hồi khi một service lỗi
- Không dùng distributed transaction giữa nhiều database

## 1. Thiết kế Order / Payment / Inventory async

### Các service

#### Order Service

- Quản lý vòng đời đơn hàng
- Trạng thái:
  - `NEW`
  - `PENDING`
  - `CONFIRMED`
  - `CANCELLED`

#### Inventory Service

- Kiểm tra và giữ tồn kho
- `Reserve stock`
- `Release stock`

#### Payment Service

- Thanh toán
- Refund

#### Notification Service

- Gửi email
- Gửi push notification

### Database riêng

Mỗi service sở hữu database riêng:

- `Order Service` -> `Order DB`
- `Inventory Service` -> `Inventory DB`
- `Payment Service` -> `Payment DB`

Không service nào đọc hoặc update trực tiếp database của service khác.

## 2. Luồng happy path

Một thiết kế phổ biến:

1. Client gọi `POST /orders`
2. Order Service tạo order trạng thái `PENDING`
3. Order Service publish `OrderCreated`
4. Inventory Service consume `OrderCreated`
5. Inventory Service reserve hàng
6. Inventory Service publish `InventoryReserved`
7. Payment Service consume `InventoryReserved`
8. Payment Service charge tiền
9. Payment Service publish `PaymentSucceeded`
10. Order Service consume `PaymentSucceeded`
11. Order chuyển sang `CONFIRMED`

```text
Client
  |
  v
Order Service
  |
  | OrderCreated
  v
RabbitMQ
  |
  v
Inventory Service
  |
  | InventoryReserved
  v
RabbitMQ
  |
  v
Payment Service
  |
  | PaymentSucceeded
  v
RabbitMQ
  |
  v
Order Service
  |
  v
CONFIRMED
```

## 3. Luồng thất bại

### 3.1 Thất bại tồn kho

```text
OrderCreated
      |
      v
Inventory Service
      |
      | Không đủ hàng
      v
InventoryRejected
      |
      v
Order Service
      |
      v
CANCELLED
```

Payment chưa xảy ra nên không cần refund.

### 3.2 Thất bại thanh toán

```text
InventoryReserved
      |
      v
Payment Service
      |
      | Thanh toán thất bại
      v
PaymentFailed
      |
      +--> Order Service: CANCELLED
      |
      +--> Inventory Service: Release stock
```

Đây là một compensation action.

Reserve stock đã thành công nhưng payment thất bại, nên hệ thống phải trả lại số lượng hàng đã giữ.

## 4. Có nên Payment trước hay Inventory trước?

Đây là câu hỏi phỏng vấn rất hay.

### Phương án 1: Reserve inventory trước

```text
Reserve stock -> Charge payment
```

#### Ưu điểm

- Không charge khi hàng đã hết
- Tránh tình huống phải refund thường xuyên

#### Nhược điểm

- Hàng bị giữ trong lúc chờ payment
- Phải có expiration để tránh giữ tồn kho vô hạn

Thường phù hợp với e-commerce.

### Phương án 2: Payment trước

```text
Charge payment -> Reserve stock
```

#### Ưu điểm

- Có thể giảm thời gian giữ hàng
- Phù hợp nếu stock không quan trọng hoặc luôn có sẵn

#### Nhược điểm

- Có thể đã charge nhưng không còn hàng
- Phải refund
- Refund có thể không tức thời

Trong đa số hệ thống bán hàng, thường chọn:

```text
Reserve inventory trước
Payment sau
```

Nhưng quyết định cuối cùng phụ thuộc business requirement.

## 5. Saga Pattern

### Vấn đề

Trong monolith, ta có thể dùng một transaction:

```text
BEGIN

INSERT order
UPDATE inventory
INSERT payment

COMMIT
```

Nhưng trong microservice:

- `Order DB`
- `Inventory DB`
- `Payment DB`

Không thể dùng một local transaction cho cả ba database.

Nếu dùng distributed transaction như two-phase commit, hệ thống thường trở nên:

- Chậm
- Khó mở rộng
- Coupling cao
- Khó vận hành

### Saga giải quyết thế nào?

Saga chia business transaction thành nhiều local transaction.

Mỗi bước thành công sẽ kích hoạt bước tiếp theo. Nếu một bước thất bại, thực hiện compensating transaction.

### Saga flow

```text
Step 1: Create Order
Step 2: Reserve Inventory
Step 3: Charge Payment
Step 4: Confirm Order
```

### Compensation

- Reserve Inventory thất bại
  - Cancel Order
- Charge Payment thất bại
  - Release Inventory
  - Cancel Order
- Confirm Order thất bại sau khi payment thành công
  - Refund payment
  - Release Inventory
  - Cancel Order

Saga không rollback database theo kiểu transaction truyền thống.

Nó thực hiện một business operation mới để bù trừ.

Ví dụ:

- Không rollback bản ghi payment cũ
- Mà tạo refund transaction mới

Điều này quan trọng vì các bước đã commit ở các service độc lập.

## 6. Hai loại Saga

### 6.1 Choreography Saga

Các service giao tiếp bằng event và tự phản ứng.

```text
Order Service
  publish OrderCreated

Inventory Service
  consume OrderCreated
  publish InventoryReserved

Payment Service
  consume InventoryReserved
  publish PaymentSucceeded

Order Service
  consume PaymentSucceeded
  confirm order
```

Không có service trung tâm điều phối.

#### Ưu điểm của choreography

- Coupling trực tiếp thấp
- Không có central orchestrator
- Dễ bắt đầu với flow đơn giản
- Phù hợp event-driven architecture

#### Nhược điểm

- Flow khó nhìn khi nhiều bước
- Business logic phân tán
- Khó debug
- Dễ tạo event chain phức tạp
- Có nguy cơ cyclic dependency

Ví dụ:

```text
A publish -> B
B publish -> C
C publish -> D
D publish -> A
```

Khi hệ thống lớn, khó biết ai đang điều khiển workflow.

### 6.2 Orchestration Saga

Có một thành phần trung tâm:

```text
Order Saga Orchestrator
```

Nó gửi command và nhận kết quả từ các service.

```text
Orchestrator
   |
   | ReserveInventory
   v
Inventory Service
   |
   | InventoryReserved
   v
Orchestrator
   |
   | ChargePayment
   v
Payment Service
   |
   | PaymentSucceeded
   v
Orchestrator
   |
   | ConfirmOrder
   v
Order Service
```

Nếu payment fail:

```text
Orchestrator
   |
   +--> ReleaseInventory
   |
   +--> CancelOrder
```

#### Ưu điểm của orchestration

- Workflow tập trung
- Dễ nhìn state machine
- Dễ biết bước hiện tại
- Compensation rõ ràng
- Dễ quản lý timeout

#### Nhược điểm

- Orchestrator có thể trở nên phức tạp
- Có nguy cơ trở thành God Service
- Các service phụ thuộc vào workflow command
- Phải đảm bảo orchestrator có độ tin cậy cao

### Khi nào chọn loại nào?

#### Choreography phù hợp khi

- Flow ngắn
- Ít service
- Event có ý nghĩa nghiệp vụ rõ ràng
- Không có nhiều nhánh hoặc compensation

#### Orchestration phù hợp khi

- Flow dài
- Có nhiều bước
- Có timeout
- Có nhiều compensation
- Cần theo dõi trạng thái workflow rõ ràng

### Câu trả lời phỏng vấn tốt

Với flow đơn giản gồm hai hoặc ba service, choreography giúp giảm coupling và triển khai nhanh. Nhưng khi workflow dài, có nhiều nhánh lỗi và compensation, tôi ưu tiên orchestration vì dễ quản lý state, timeout và khả năng phục hồi hơn.

## 7. Command và Event khác nhau thế nào?

### Command

Command yêu cầu một service thực hiện một hành động.

Ví dụ:

- `ReserveInventory`
- `ChargePayment`
- `CancelOrder`
- `RefundPayment`

Command mang tính imperative:

> Hãy làm việc này.

Thường có một intended consumer.

### Event

Event mô tả một sự việc đã xảy ra.

Ví dụ:

- `OrderCreated`
- `InventoryReserved`
- `PaymentSucceeded`
- `OrderCancelled`

Event mang tính quá khứ:

> Việc này đã xảy ra.

Có thể có nhiều consumer.

### Sai lầm thường gặp

Đặt event như command:

- `DoPayment`
- `SendEmail`

Tên tốt hơn:

- `PaymentRequested`
- `OrderConfirmed`
- `UserRegistered`

Hoặc nếu thật sự là command:

- `ChargePaymentCommand`
- `SendOrderConfirmationCommand`

Nên tách rõ semantic giữa command và event.

## 8. Outbox Pattern

Đây là một trong những pattern quan trọng nhất khi dùng database và message broker.

### Dual-write problem

Order Service cần làm hai việc:

1. Insert order vào DB
2. Publish `OrderCreated` vào RabbitMQ

Nếu không có outbox, có hai tình huống nguy hiểm.

#### Trường hợp 1: DB thành công, publish thất bại

```text
INSERT order thành công
RabbitMQ publish thất bại
```

Kết quả:

- Order tồn tại
- Nhưng Inventory không nhận được event

Hệ thống mất đồng bộ.

#### Trường hợp 2: Publish thành công, DB thất bại

```text
Publish OrderCreated thành công
INSERT order rollback
```

Inventory có thể giữ hàng cho một order không tồn tại.

### Giải pháp Outbox

Trong cùng một local database transaction:

```text
BEGIN

INSERT INTO orders(...)
INSERT INTO outbox_events(...)

COMMIT
```

Ví dụ:

```text
orders
------------------------------------------------
id       status
O123     PENDING

outbox_events
------------------------------------------------
id       event_type       aggregate_id    status
E001     OrderCreated     O123            NEW
```

Vì cả hai insert cùng transaction:

- Cùng thành công
- Hoặc cùng rollback

### Outbox publisher

Một background worker đọc bảng outbox:

```text
SELECT *
FROM outbox_events
WHERE status = 'NEW'
```

Sau đó publish lên RabbitMQ:

```text
outbox worker
      |
      v
RabbitMQ
```

Publish thành công thì update:

```text
status = PUBLISHED
```

### Luồng đầy đủ

```text
Order API
   |
   v
Database transaction
   |
   +--> Insert Order
   |
   +--> Insert Outbox Event
   |
   v
Commit
   |
   v
Outbox Publisher
   |
   v
RabbitMQ
```

## 9. Outbox có tạo duplicate không?

Có.

Ví dụ:

1. Worker publish event thành công
2. Worker crash trước khi update `status=PUBLISHED`
3. Worker restart
4. Publish lại event

Message bị duplicate.

Vì vậy outbox không loại bỏ duplicate.

Outbox giải quyết:

- Không bị mất event do dual-write

Nhưng consumer vẫn phải idempotent.

Thiết kế chuẩn:

```text
Transactional Outbox
+ At-least-once delivery
+ Idempotent Consumer
```

## 10. Polling Publisher và CDC

Có hai cách publish outbox.

### Polling Publisher

Worker polling định kỳ:

```text
SELECT ...
FROM outbox_events
WHERE status = 'NEW'
LIMIT 100
```

#### Ưu điểm

- Dễ triển khai
- Không cần hạ tầng CDC
- Dễ kiểm soát

#### Nhược điểm

- Có độ trễ polling
- Tạo tải DB
- Phải xử lý lock và concurrency

### Change Data Capture

CDC đọc transaction log của database.

```text
Database transaction log
      |
      v
CDC Connector
      |
      v
Message Broker
```

#### Ưu điểm

- Gần real-time
- Ít polling DB
- Có thể scale tốt hơn

#### Nhược điểm

- Hạ tầng phức tạp
- Phụ thuộc công nghệ database
- Vận hành khó hơn

Trong phỏng vấn, bạn có thể nói:

> Với hệ thống vừa và nhỏ, tôi thường bắt đầu bằng polling outbox vì đơn giản. Khi throughput lớn hoặc cần near-real-time, có thể cân nhắc CDC.

## 11. Inbox Pattern

Outbox giúp producer không mất event.

Inbox giúp consumer xử lý idempotent.

Consumer nhận message:

```json
{
  "event_id": "evt-123",
  "type": "InventoryReserved",
  "order_id": "O123"
}
```

Trong transaction của consumer:

```text
BEGIN

INSERT INTO inbox_events(event_id)
UPDATE business_data

COMMIT
```

Tạo unique constraint:

```text
UNIQUE(event_id)
```

Nếu message bị gửi lại:

- `INSERT inbox_events` thất bại do duplicate key
- Consumer biết event đã xử lý và ack

### Inbox và business update phải cùng transaction

Sai:

1. Insert inbox event
2. Commit
3. Update payment
4. Crash

Khi redelivery, hệ thống thấy event đã xử lý nhưng payment chưa update.

Đúng:

```text
BEGIN

Check/insert inbox event
Update payment

COMMIT
```

## 12. Event-Driven Architecture

Event-driven architecture là kiến trúc mà các component giao tiếp bằng event.

Ví dụ:

- `OrderCreated`
- `InventoryReserved`
- `PaymentSucceeded`
- `OrderConfirmed`

Service publish event sau khi state của nó thay đổi.

Các service khác subscribe và phản ứng.

### Lợi ích

#### Loose coupling

Order Service không cần gọi trực tiếp:

- Email Service
- Analytics Service
- Recommendation Service

Nó chỉ publish:

- `OrderConfirmed`

Các service quan tâm sẽ subscribe.

#### Independent scaling

Consumer có thể scale riêng:

- Payment Consumer x5
- Notification Consumer x20
- Analytics Consumer x50

#### Resilience

Nếu Notification Service tạm thời down:

- message vẫn nằm trong queue
- Order flow không nhất thiết phải thất bại

#### Extensibility

Thêm một service mới:

- Fraud Detection Service

Chỉ cần subscribe `PaymentRequested`.

Không cần sửa Order Service.

### Nhược điểm

- Eventual consistency
- Khó debug
- Khó trace
- Duplicate message
- Out-of-order delivery
- Schema evolution
- Event chain khó kiểm soát
- Cần monitoring tốt

## 13. Event notification và event-carried state transfer

### Event notification

Event chỉ chứa ít dữ liệu:

```json
{
  "event_type": "OrderCreated",
  "order_id": "O123"
}
```

Consumer phải gọi lại Order Service để lấy chi tiết.

#### Ưu điểm

- Payload nhỏ
- Ít duplicate data

#### Nhược điểm

- Tạo synchronous dependency
- Order Service có thể down
- Tăng latency
- Dễ gây N+1 API calls

### Event-carried state transfer

Event chứa dữ liệu consumer cần:

```json
{
  "event_type": "OrderCreated",
  "order_id": "O123",
  "user_id": "U001",
  "items": [
    {
      "product_id": "P01",
      "quantity": 2
    }
  ],
  "total_amount": 500000
}
```

#### Ưu điểm

- Consumer tự xử lý
- Không cần gọi ngược
- Giảm runtime coupling

#### Nhược điểm

- Payload lớn
- Duplicate data
- Cần quản lý schema version
- Có thể lộ dữ liệu không cần thiết

Trong thực tế thường gửi đủ dữ liệu để consumer xử lý, nhưng không gửi toàn bộ aggregate một cách tùy tiện.

## 14. Event schema design

Một event production-ready thường có envelope:

```json
{
  "event_id": "evt-123",
  "event_type": "OrderCreated",
  "event_version": 1,
  "occurred_at": "2026-07-12T10:30:00Z",
  "producer": "order-service",
  "correlation_id": "req-456",
  "causation_id": "cmd-789",
  "aggregate_id": "O123",
  "payload": {
    "user_id": "U001",
    "total_amount": 500000
  }
}
```

Các field quan trọng:

- `event_id`: idempotency
- `event_type`: phân loại event
- `event_version`: schema evolution
- `occurred_at`: thời điểm nghiệp vụ xảy ra
- `correlation_id`: trace toàn bộ flow
- `causation_id`: event nào hoặc command nào tạo ra event hiện tại
- `aggregate_id`: entity liên quan

## 15. Failure Handling

Đây là phần Tech Lead thường đào sâu nhất.

Ta chia failure thành nhiều lớp.

### 15.1 Producer publish thất bại

Ví dụ:

- Order đã lưu DB
- RabbitMQ unavailable

Giải pháp:

- Transactional outbox
- Publisher retry
- Publisher confirm
- Alert nếu outbox backlog tăng
- Không retry vô hạn không kiểm soát

Metrics cần theo dõi:

- `outbox_pending_count`
- `oldest_outbox_event_age`
- `publish_failure_count`

### 15.2 Consumer tạm thời lỗi

Ví dụ:

- DB timeout
- Payment gateway 503
- Network error
- Redis unavailable

Giải pháp:

- Retry có giới hạn
- Exponential backoff
- Jitter

Ví dụ:

- 5 giây
- 30 giây
- 2 phút
- 10 phút

Jitter giúp nhiều consumer không retry cùng lúc gây thundering herd.

### 15.3 Poison message

Poison message là message luôn thất bại.

Ví dụ:

- JSON sai
- Thiếu field
- Schema không tương thích
- Business data không hợp lệ

Không nên requeue vô hạn.

```text
Validate
   |
   | lỗi permanent
   v
DLQ
```

DLQ cần có:

- Alert
- Dashboard
- Cơ chế inspect
- Cơ chế replay
- Audit log

DLQ không phải nơi “ném message vào rồi quên”.

### 15.4 Consumer crash trước ACK

1. Consumer nhận event
2. Update DB thành công
3. Crash trước ACK
4. Broker redeliver

Giải pháp:

- Manual ack
- Idempotent consumer
- Inbox table
- Unique constraint
- Ack chỉ sau commit

### 15.5 Consumer crash sau gọi external API

Ví dụ:

- Payment gateway charge thành công
- Consumer crash trước khi lưu DB

Khi retry, có nguy cơ charge lần hai.

Giải pháp:

- Dùng idempotency key với payment provider
- Reconcile với provider bằng transaction ID
- Lưu payment attempt
- Không dựa chỉ vào trạng thái local
- Có reconciliation job

### 15.6 Out-of-order message

Ví dụ:

- `OrderCancelled` đến trước `OrderCreated`
- `PaymentRefunded` đến trước `PaymentSucceeded`

Nguyên nhân:

- Nhiều queue
- Retry
- Nhiều consumer
- Network delay
- Concurrent publish

Giải pháp:

- Event version
- Aggregate version
- State machine validation
- Ignore stale event
- Serialize theo aggregate khi thật sự cần
- Thiết kế operation có điều kiện

Ví dụ:

```sql
UPDATE orders
SET status = 'CANCELLED',
    version = 3
WHERE id = 'O123'
  AND version = 2;
```

### 15.7 Duplicate event

Giải pháp:

- `event_id`
- Inbox table
- Unique constraint
- Idempotent business logic

Không chỉ check trong memory hoặc Redis nếu business critical.

Redis có thể mất dữ liệu hoặc expire key.

Database unique constraint thường là lớp bảo vệ đáng tin cậy hơn.

### 15.8 Service down lâu

Ví dụ Payment Service down 6 giờ.

RabbitMQ queue sẽ tăng.

Cần:

- Queue depth monitoring
- Consumer lag monitoring
- Auto-scaling
- Disk alert
- Queue max length
- TTL phù hợp
- Backpressure
- Capacity planning

Không nên chỉ nghĩ:

> RabbitMQ giữ message nên không sao.

Queue có thể đầy disk và ảnh hưởng toàn broker.

## 16. Timeout trong Saga

Một saga có thể bị treo.

Ví dụ:

```text
OrderCreated
InventoryReserved
PaymentRequested
```

Sau đó không có `PaymentSucceeded` hoặc `PaymentFailed`.

Nguyên nhân:

- Message mất
- Consumer down
- External provider treo
- Bug

Saga cần timeout.

Ví dụ:

- Payment phải hoàn tất trong 15 phút

Nếu quá timeout:

- ReleaseInventory
- CancelOrder

Với orchestration, timeout thường dễ quản lý hơn vì orchestrator giữ state.

## 17. Saga state machine

Một saga orchestrator có thể lưu:

- `saga_id`
- `order_id`
- `current_step`
- `status`
- `retry_count`
- `deadline`
- `last_error`

Ví dụ state:

- `STARTED`
- `INVENTORY_PENDING`
- `INVENTORY_RESERVED`
- `PAYMENT_PENDING`
- `PAYMENT_SUCCEEDED`
- `COMPLETED`
- `COMPENSATING`
- `COMPENSATED`
- `FAILED`

Không nên chỉ dựa vào chuỗi event mà không lưu state của saga, đặc biệt với workflow dài.

## 18. Compensation cũng có thể thất bại

Ví dụ:

- Payment thất bại
- Release inventory

Nhưng Inventory Service đang down.

Không thể giả định compensation luôn thành công.

Compensation cần:

- Retry
- Idempotency
- DLQ
- Alert
- Manual recovery
- Reconciliation job

Saga không loại bỏ lỗi. Nó cung cấp mô hình để quản lý lỗi.

## 19. Một kiến trúc production-ready

```text
Client
  |
  v
API Gateway
  |
  v
Order Service
  |
  | DB transaction
  +--> orders
  +--> outbox_events
  |
  v
Outbox Publisher
  |
  v
RabbitMQ
  |
  +--> Inventory Queue
  |        |
  |        v
  |    Inventory Service
  |        |
  |        +--> Inventory DB
  |        +--> Inventory Outbox
  |
  +--> Payment Queue
  |        |
  |        v
  |    Payment Service
  |        |
  |        +--> Payment DB
  |        +--> Payment Outbox
  |
  +--> Notification Queue
```

Mỗi consumer có:

- Manual ack
- Inbox/idempotency
- Retry with delay
- DLQ
- Structured logging
- Tracing
- Metrics

## 20. Observability cho flow async

Mỗi message nên có:

- `trace_id`
- `correlation_id`
- `event_id`
- `order_id`
- `saga_id`

Log mẫu:

```json
{
  "service": "payment-service",
  "event": "PaymentRequested",
  "event_id": "evt-123",
  "order_id": "O123",
  "saga_id": "saga-789",
  "retry_count": 2,
  "status": "FAILED",
  "error": "gateway timeout"
}
```

Metrics quan trọng:

- `queue_depth`
- `message_publish_rate`
- `message_consume_rate`
- `consumer_error_rate`
- `retry_count`
- `dlq_count`
- `oldest_message_age`
- `saga_duration`
- `saga_failure_rate`
- `outbox_backlog`

## 21. Câu trả lời phỏng vấn mẫu

### “Hãy thiết kế flow tạo order async”

Khi client tạo order, Order Service sẽ tạo order trạng thái `PENDING` và ghi event `OrderCreated` vào outbox trong cùng một database transaction. Outbox worker publish event lên RabbitMQ. Inventory Service consume event, kiểm tra idempotency và reserve hàng trong local transaction. Nếu thành công, service publish `InventoryReserved`; nếu thất bại, publish `InventoryRejected`.

Payment Service chỉ xử lý sau khi inventory đã được reserve. Nếu payment thành công, nó publish `PaymentSucceeded`; Order Service consume event và chuyển order sang `CONFIRMED`. Nếu payment thất bại, hệ thống thực hiện compensation bằng cách release inventory và cancel order.

Tất cả consumer dùng manual ack, retry có backoff, DLQ và idempotency. Tôi không giả định exactly-once, mà thiết kế theo at-least-once delivery kết hợp outbox/inbox pattern.

## 22. Những lỗi thiết kế dễ bị trừ điểm

- Dùng chung database
  - `Order Service update inventory_table`
  - Sai ownership và tăng coupling
- Publish message trực tiếp sau khi commit DB
  - `save DB`
  - `publish RabbitMQ`
  - Có dual-write problem
- Retry vô hạn
  - `nack(requeue=true)` liên tục, không delay
  - Dễ tạo hot loop
- Không có idempotency
  - Cho rằng một message chỉ đến một lần
- Ack trước khi commit
  - Có nguy cơ mất message
- Compensation không idempotent
  - `ReleaseInventory` chạy hai lần có thể cộng tồn kho sai
- Không có timeout
  - Saga có thể treo vĩnh viễn
- Dùng event như RPC trá hình
  - `Order Service publish GetInventory`
  - `Inventory publish InventoryResponse`
  - Không phải lúc nào async RPC cũng là lựa chọn tốt. Với query cần phản hồi tức thời, REST/gRPC có thể phù hợp hơn.

## 23. Cheat sheet

- Saga: chia distributed transaction thành nhiều local transaction
- Compensation: business action để bù lại bước đã commit
- Choreography: service phản ứng theo event, không có coordinator trung tâm
- Orchestration: có orchestrator quản lý workflow và compensation
- Outbox: ghi business data và event trong cùng DB transaction
- Inbox: lưu `event_id` đã xử lý để chống duplicate
- Event-driven: service publish sự kiện, service khác subscribe và phản ứng
- Failure handling: retry + backoff + DLQ + idempotency + timeout + reconciliation
- Guarantee thực tế: at-least-once + idempotent consumer
- Không nên giả định: exactly-once end-to-end

## Câu hỏi luyện phỏng vấn

Order đã tạo thành công, inventory đã reserve thành công, payment cũng charge thành công. Tuy nhiên `PaymentSucceeded` không đến được Order Service. Order vẫn ở trạng thái `PENDING`. Bạn sẽ thiết kế hệ thống như thế nào để phát hiện và phục hồi trường hợp này?
