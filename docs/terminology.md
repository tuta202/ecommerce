# Terminology

## 1. Coupling

Coupling là mức độ phụ thuộc giữa các service.

Nếu service A thay đổi mà service B cũng phải thay đổi theo, thì hai service đang bị coupling cao.

Ngược lại, giảm coupling tức là thiết kế sao cho mỗi service độc lập, ít bị ảnh hưởng bởi thay đổi của service khác, giúp hệ thống dễ mở rộng và bảo trì hơn.

Ví dụ dễ hiểu: nếu hai ứng dụng dùng chung một database table, chúng bị coupling cao. Nếu mỗi ứng dụng có API riêng và chỉ giao tiếp qua API, coupling thấp hơn.

## 2. Spike traffic

Spike traffic là tình huống lượng truy cập tăng đột biến trong một khoảng thời gian ngắn, vượt xa mức bình thường.

Ví dụ: một website thương mại điện tử có thể gặp spike traffic khi mở chương trình flash sale, khiến số lượng người dùng truy cập cùng lúc tăng gấp nhiều lần.

Nói ngắn gọn: spike traffic = lưu lượng truy cập bất thường, tăng mạnh và nhanh.

## 3. Workload

Trong Kubernetes, workload là nhóm tài nguyên dùng để quản lý việc chạy ứng dụng trong cluster.

Workload không chỉ là Pod. Pod là nơi container thực sự chạy, còn workload định nghĩa cách Pod được tạo ra, duy trì, scale, cập nhật hoặc chạy theo lịch.

Các loại workload phổ biến:

| Loại workload | Dùng để làm gì? |
| --- | --- |
| Deployment | Chạy ứng dụng stateless, dễ scale, rolling update và rollback. Đây là loại phổ biến nhất cho backend service thông thường. |
| StatefulSet | Chạy ứng dụng có trạng thái, cần định danh ổn định, network identity ổn định hoặc storage riêng cho từng replica. |
| DaemonSet | Đảm bảo mỗi Node phù hợp đều chạy một Pod, thường dùng cho agent như logging, monitoring hoặc networking. |
| Job | Chạy tác vụ một lần đến khi hoàn thành, ví dụ migrate dữ liệu hoặc xử lý batch. |
| CronJob | Tạo Job theo lịch, ví dụ cleanup dữ liệu mỗi đêm hoặc reconciliation định kỳ. |

Nói ngắn gọn:

```text
workload = cách Kubernetes quản lý Pod để chạy ứng dụng
```

Ví dụ:

```text
Deployment order-service
  -> tạo và duy trì ReplicaSet
    -> ReplicaSet tạo và duy trì Pod
```

Nếu một Pod thuộc Deployment bị chết, Deployment/ReplicaSet sẽ tạo Pod mới để giữ đúng số replica mong muốn.
