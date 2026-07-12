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
