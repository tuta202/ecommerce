# RabbitMQ trong Microservice

## 1. RabbitMQ dùng để làm gì?

RabbitMQ là một message broker. Nó đứng giữa service gửi message và service nhận message.

Ví dụ:

```text
Order Service  --->  RabbitMQ  --->  Payment Service
                              --->  Inventory Service
                              --->  Notification Service
```

Thay vì service gọi trực tiếp nhau bằng REST hoặc gRPC, service có thể gửi message qua RabbitMQ.

Ví dụ thực tế:

```text
User đăng ký tài khoản
  ↓
User Service publish event: UserRegistered
  ↓
RabbitMQ
  ↓
+--> Email Service gửi email welcome
+--> CRM Service lưu thông tin khách hàng
+--> Analytics Service ghi event
```

## 2. Producer, Consumer, Broker

### Producer

Producer là service gửi message.

Ví dụ:

- `Order Service` publish `OrderCreated`

Producer thường gửi message vào exchange, không gửi trực tiếp vào queue.

### Consumer

Consumer là service nhận và xử lý message từ queue.

Ví dụ:

- `Payment Service` consume `OrderCreated`

Consumer đọc message từ queue, xử lý xong thì gửi ACK cho RabbitMQ.

### Broker

RabbitMQ chính là broker, giữ vai trò trung gian.

Broker giống như một bưu điện: producer gửi thư đến bưu điện, bưu điện lưu trữ và phân loại, rồi consumer đến lấy thư.

Luồng chuẩn trong RabbitMQ:

```text
Producer
   |
   v
Exchange
   |
   v
Binding
   |
   v
Queue
   |
   v
Consumer
```

## 3. Exchange, Queue, Binding, Routing Key

### Queue

Queue là nơi lưu message chờ consumer xử lý.

Ví dụ:

- `payment.queue`
- `notification.queue`
- `inventory.queue`

Một queue có thể có một hoặc nhiều consumer.

Ví dụ:

```text
payment.queue
   |
   +--> Payment Consumer 1
   +--> Payment Consumer 2
   +--> Payment Consumer 3
```

Các consumer này cạnh tranh nhau xử lý message. Một message thường chỉ được giao cho một consumer trong cùng một queue.

### Exchange

Exchange nhận message từ producer và quyết định đẩy message vào queue nào.

Producer không cần biết queue cụ thể.

```text
Producer -> Exchange -> Queue
```

Ví dụ:

```text
Order Service
   |
   v
order.exchange
   |
   +--> payment.queue
   +--> inventory.queue
   +--> notification.queue
```

### Binding

Binding là luật nối giữa exchange và queue.

```text
exchange + binding rule + queue
```

Ví dụ:

```text
order.exchange -- routing_key=order.created --> payment.queue
```

Nghĩa là nếu message có routing key `order.created`, nó sẽ được route vào `payment.queue`.

### Routing key

Routing key là chuỗi đi kèm message để exchange quyết định route message.

Ví dụ:

- `order.created`
- `order.cancelled`
- `payment.succeeded`
- `payment.failed`
- `user.registered`

Ví dụ producer publish:

```text
exchange: order.exchange
routing_key: order.created
body:
{
  "order_id": "O123",
  "user_id": "U456",
  "amount": 500000
}
```

RabbitMQ nhìn routing key rồi route message theo loại exchange.

## 4. Các loại Exchange

### 4.1 Direct Exchange

Direct exchange route message theo routing key khớp chính xác.

```text
routing_key == binding_key
```

Ví dụ:

```text
order.direct.exchange

Bindings:
payment.queue      <- order.created
inventory.queue    <- order.created
refund.queue       <- order.cancelled
```

Publish:

```text
routing_key = order.created
```

Kết quả:

- `payment.queue`
- `inventory.queue`

Nếu publish:

```text
routing_key = order.cancelled
```

Kết quả:

- `refund.queue`

Khi nào dùng direct?

Khi event type rõ ràng và muốn route chính xác.

### 4.2 Topic Exchange

Topic exchange route message theo pattern.

Nó dùng hai ký tự đặc biệt:

- `*`: match đúng một word
- `#`: match zero hoặc nhiều word

Các word được phân cách bằng dấu chấm.

Ví dụ routing key:

- `order.created.vn`
- `order.created.us`
- `order.cancelled.vn`
- `payment.failed.vn`

Bindings:

- `billing.queue` <- `order.*.vn`
- `analytics.queue` <- `order.#`
- `error.queue` <- `*.failed.#`

Kết quả:

- `routing_key = order.created.vn` sẽ match `billing.queue`, `analytics.queue`
- `routing_key = order.cancelled.vn` sẽ match `billing.queue`, `analytics.queue`
- `routing_key = payment.failed.vn` sẽ match `error.queue`

Khi nào dùng topic?

Khi cần route linh hoạt theo pattern.

### 4.3 Fanout Exchange

Fanout exchange không quan tâm routing key. Nó broadcast message tới tất cả queue được bind vào exchange.

```text
Producer
   |
   v
fanout.exchange
   |
   +--> email.queue
   +--> analytics.queue
   +--> audit.queue
```

Publish một message:

- `UserRegistered`

Tất cả queue đều nhận được một bản copy.

### Lưu ý quan trọng

Fanout gửi tới mỗi queue, không phải mỗi consumer.

- Nếu một queue có 3 consumer:

```text
email.queue
   |
   +--> consumer 1
   +--> consumer 2
   +--> consumer 3
```

thì một message chỉ được một consumer trong `email.queue` xử lý.

- Nhưng nếu có 3 queue:

```text
email.queue
analytics.queue
audit.queue
```

thì mỗi queue nhận một bản copy.

### 4.4 Headers Exchange

Headers exchange route message dựa vào header thay vì routing key.

Khi nào dùng headers?

Khi logic route dựa trên metadata phức tạp.

## 5. Ack, Nack, Retry

### Ack là gì?

ACK là consumer báo cho RabbitMQ:

> Tôi xử lý message thành công rồi, broker có thể xóa message khỏi queue.

Luồng đúng:

```text
Consumer nhận message
  ↓
Xử lý business logic
  ↓
Ghi DB thành công
  ↓
ACK
```

Sai lầm phổ biến:

- Ack trước khi xử lý xong

```text
Consumer nhận message
  ↓
ACK ngay
  ↓
Ghi DB lỗi
```

Message đã bị xóa, dữ liệu mất.

### Manual Ack vs Auto Ack

#### Auto Ack

RabbitMQ coi message thành công ngay khi gửi cho consumer.

```text
Broker -> Consumer
Broker xóa message
Consumer xử lý sau
```

Nếu consumer crash, message mất.

Không nên dùng cho business-critical message.

#### Manual Ack

Consumer xử lý xong mới ack.

```text
Broker -> Consumer
Consumer xử lý
Consumer ack
Broker xóa message
```

Nên dùng trong production.

### Nack là gì?

NACK là consumer báo:

> Tôi xử lý lỗi message này.

Có thể:

```text
nack(requeue=true)
```

Đẩy lại message vào queue.

Hoặc:

```text
nack(requeue=false)
```

Không requeue, message có thể bị drop hoặc đi vào dead letter queue nếu có cấu hình.

## 6. Retry

Ví dụ consumer lỗi do Payment Gateway timeout.

Có 2 loại lỗi:

### Transient error

Lỗi tạm thời:

- Network timeout
- DB lock
- External API 503
- RabbitMQ connection issue

Có thể retry.

### Permanent error

Lỗi vĩnh viễn:

- Payload sai schema
- `order_id` không tồn tại
- Amount âm
- User bị khóa
- Message version không support

Không nên retry mãi.

### Bad retry pattern

Không nên retry vô hạn ngay lập tức:

```text
consume -> fail -> requeue -> consume -> fail -> requeue...
```

Vấn đề:

- Tạo vòng lặp nóng
- CPU tăng
- Queue bị nghẽn
- Message lỗi chặn message khác

### Good retry pattern

Dùng retry có giới hạn và delay.

Ví dụ:

```text
main.queue
  |
  | fail
  v
retry.5s.queue
  |
  | after 5s
  v
main.queue
  |
  | fail
  v
retry.1m.queue
  |
  | after 1m
  v
main.queue
  |
  | fail nhiều lần
  v
dead-letter.queue
```

Hoặc dùng exponential backoff:

- 5s -> 30s -> 2m -> 10m

## 7. Dead Letter Queue

DLQ là queue chứa message không xử lý được.

Message có thể vào DLQ khi:

- Bị `nack(requeue=false)`
- Message hết TTL
- Queue quá giới hạn length
- Message bị reject

Ví dụ:

```text
payment.queue
   |
   | fail 3 lần
   v
payment.dlq
```

DLQ giúp:

- Không mất message
- Không làm nghẽn queue chính
- Điều tra payload lỗi
- Replay sau khi fix bug

### DLX là gì?

Dead Letter Exchange là exchange nhận message chết.

```text
main.queue
   |
   | dead-letter
   v
dead-letter.exchange
   |
   v
payment.dlq
```

Thường cấu hình:

- `x-dead-letter-exchange`
- `x-dead-letter-routing-key`

## 8. Prefetch

Prefetch là số lượng message unacked tối đa mà RabbitMQ gửi cho một consumer.

Ví dụ:

```text
prefetch = 10
```

Nghĩa là một consumer có thể nhận tối đa 10 message chưa ack.

### Prefetch quá cao

- Consumer nhận quá nhiều message
- Xử lý chậm
- Message nằm trong RAM consumer
- Consumer khác bị đói việc
- Nếu crash thì broker phải redeliver nhiều message

### Prefetch quá thấp

```text
prefetch = 1
```

An toàn, phân phối đều, nhưng throughput có thể thấp nếu xử lý nhanh.

### Chọn prefetch thế nào?

Tùy workload:

- Xử lý nặng, lâu: `prefetch = 1 - 5`
- Xử lý nhanh, I/O nhiều: `prefetch = 10 - 100`
- Cần strict ordering theo queue: `prefetch = 1`

## 9. Competing Consumers

Competing consumers nghĩa là nhiều consumer cùng consume một queue để scale xử lý.

```text
payment.queue
   |
   +--> payment-worker-1
   +--> payment-worker-2
   +--> payment-worker-3
```

Một message chỉ được xử lý bởi một worker.

Ví dụ queue có 300 message, 3 worker:

- `worker-1`: khoảng 100
- `worker-2`: khoảng 100
- `worker-3`: khoảng 100

Mục tiêu:

- Tăng throughput
- High availability
- Chia tải

### Khi nào cẩn thận?

Nếu message cần xử lý theo thứ tự.

Ví dụ:

- `order.created`
- `order.cancelled`

Nếu nhiều consumer xử lý song song, có thể xảy ra:

- `order.cancelled` xử lý trước `order.created`

Cách xử lý:

- Partition theo aggregate id
- Dùng một queue riêng cho key quan trọng
- Kiểm tra state trong DB
- Xử lý idempotent
- Thiết kế event version

RabbitMQ không phải công cụ lý tưởng nếu bạn cần ordering mạnh theo partition như Kafka.

## 10. Message durability và persistence

Đây là phần nhiều ứng viên nhầm.

Để giảm nguy cơ mất message khi RabbitMQ restart, cần đủ các điều kiện:

### 1. Durable queue

Queue phải durable.

```text
queue durable = true
```

Queue tồn tại sau khi broker restart. Nhưng durable queue không tự động làm message bền vững.

### 2. Persistent message

Message phải persistent.

```text
delivery_mode = 2
```

Message được ghi xuống disk.

### 3. Durable exchange

Exchange cũng nên durable.

```text
exchange durable = true
```

### 4. Publisher confirm

Producer cần bật publisher confirm để biết broker đã nhận message an toàn.

Nếu không, producer có thể tưởng đã publish thành công trong khi message chưa chắc đã được broker ghi nhận.

## 11. Idempotency

Đây là phần cực kỳ quan trọng khi làm hệ thống thực tế.

RabbitMQ thường đảm bảo kiểu:

```text
at-least-once delivery
```

Nghĩa là message có thể được gửi lại nhiều lần.

### Vì sao?

Ví dụ:

Consumer xử lý thành công
Ghi DB thành công
Nhưng crash trước khi ACK

RabbitMQ không nhận được ack, nên sẽ giao lại message.

Kết quả:

- Cùng một message được xử lý 2 lần

Nếu là payment:

- Trừ tiền 2 lần

Rất nguy hiểm.

### Idempotency là gì?

Idempotency nghĩa là xử lý cùng một message nhiều lần nhưng kết quả cuối cùng vẫn như một lần.

Ví dụ:

```text
message_id = evt_123
```

Consumer lưu message đã xử lý:

```text
processed_messages
- evt_123
```

Khi nhận lại:

```text
if evt_123 đã xử lý:
    ACK và bỏ qua
else:
    xử lý và lưu evt_123
```

### Ví dụ thực tế

Payment Consumer nhận:

```json
{
  "event_id": "evt_123",
  "order_id": "O123",
  "amount": 500000
}
```

Trong DB:

```text
processed_events
event_id | processed_at
evt_123  | 2026-07-05
```

Nếu RabbitMQ redeliver `evt_123`, consumer thấy đã xử lý rồi thì ack luôn, không charge lại.

### Idempotency key

Có thể dùng:

- `event_id`
- `message_id`
- `order_id + event_type`
- `external_transaction_id`

Tùy business.

Với payment, thường dùng:

- `payment_request_id`
- hoặc `idempotency_key`

## 12. Exactly-once illusion

Nhiều ứng viên nói:

> RabbitMQ đảm bảo exactly-once.

Câu này thường sai trong thực tế.

Trong distributed system, exactly-once end-to-end gần như là ảo tưởng nếu tính cả:

- Producer
- Broker
- Consumer
- Database
- External API

RabbitMQ có thể redeliver message khi không nhận được ack.

Producer cũng có thể publish duplicate nếu retry sau timeout.

Consumer có thể xử lý xong DB nhưng crash trước ack.

Do đó thực tế phải thiết kế theo:

```text
at-least-once delivery + idempotent consumer
```

### Ví dụ crash kinh điển

1. Consumer nhận message `PaymentRequested`
2. Consumer gọi payment gateway thành công
3. Consumer ghi DB thành công
4. Consumer crash trước khi ACK RabbitMQ
5. RabbitMQ giao lại message
6. Consumer gọi payment gateway lần nữa

Nếu không có idempotency key, user bị charge 2 lần.

### Câu trả lời phỏng vấn

Mình không assume exactly-once end-to-end với RabbitMQ. RabbitMQ thường được thiết kế theo at-least-once delivery khi dùng manual ack. Vì vậy consumer phải idempotent. Với các thao tác nguy hiểm như payment, mình dùng idempotency key, unique constraint, processed event table hoặc transactional outbox/inbox pattern để tránh side effect bị thực hiện nhiều lần.

## 13. Một flow RabbitMQ production-ready

Ví dụ xử lý payment:

```text
Order Service
   |
   | publish PaymentRequested
   v
payment.exchange
   |
   | routing_key = payment.requested
   v
payment.queue
   |
   v
Payment Consumer
   |
   +--> validate schema
   +--> check idempotency
   +--> call payment gateway with idempotency key
   +--> save payment result
   +--> publish PaymentSucceeded / PaymentFailed
   +--> ack
```

Nếu lỗi:

### Transient error

```text
Transient error
   |
   v
retry queue with delay
   |
   v
retry lại
```

### Permanent error

```text
Permanent error
   |
   v
DLQ
```
