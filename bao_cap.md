# BÁO CÁO ĐÁNH GIÁ CHIẾN LƯỢC: XÁC THỰC TRONG HỆ THỐNG BLOCKCHAIN EXPLORER

---

## Phần 1: Phân tích logic & So sánh chiến lược

Đối với một hệ thống mang tính chất toàn cầu, dữ liệu cập nhật theo thời gian thực như Blockchain Explorer, việc lựa chọn cơ chế xác thực ảnh hưởng trực tiếp đến hạ tầng vật lý và trải nghiệm người dùng. Dưới đây là bảng phân tích đối chiếu chuyên sâu giữa hai giải pháp:

### 1. Phương pháp Session-based Authentication (Stateful)
Cơ chế này lưu trữ trạng thái đăng nhập của người dùng trực tiếp trong bộ nhớ của Server (Memory/RAM) hoặc Database tập trung (Redis/Memcached), đồng thời trả về một `Session ID` lưu ở Cookie phía Client.

* **Về Khả năng mở rộng (Scalability):** Gặp rào cản rất lớn. Khi có hàng triệu người dùng truy cập đồng thời để tracking ví, hệ thống cần một lượng RAM khổng lồ chỉ để lưu Session. Nếu triển khai nhiều cụm Server (Load Balancing), chúng ta buộc phải dùng Sticky Session (ép user vào một server cố định) hoặc dựng một cụm Redis riêng để chia sẻ Session, làm phát sinh chi phí hạ tầng và tạo ra điểm nghẽn nghiêm trọng (Single Point of Failure).
* **Về Hoạt động qua API và Dịch vụ phân tán (Microservices):** Giao thức HTTP Session phụ thuộc nặng nề vào Cookie. Các ứng dụng di động (Mobile App) hoặc các bên thứ ba kết nối qua API không quản lý Cookie tốt như trình duyệt Web, dẫn đến việc tích hợp đa nền tảng bị hạn chế. Ngoài ra, việc các Microservices phải liên tục gọi ngược về Session Server để kiểm tra ID sẽ gây thắt nút cổ chai băng thông nội bộ.

### 2. Phương pháp JSON Web Token - JWT (Stateless)
Cơ chế này không lưu trạng thái trên Server. Mọi thông tin định danh và quyền hạn của người dùng đều được gói gọn, ký số an toàn bên trong Payload của JWT và giao hoàn toàn cho Client quản lý.

* **Về Khả năng mở rộng (Scalability):** Đạt điểm tối ưu tuyệt đối. Server không tốn bất kỳ một byte RAM nào để lưu trạng thái phiên. Khi có hàng triệu request đổ về, mỗi Server trong cụm chỉ cần dùng thuật toán để giải mã và xác minh chữ ký của Token. Hệ thống có thể scale-up (mở rộng theo chiều ngang) thêm hàng trăm Server mới cực kỳ dễ dàng mà không cần lo lắng về việc đồng bộ dữ liệu phiên.
* **Về Hoạt động qua API và Dịch vụ phân tán (Microservices):** JWT hoạt động qua HTTP Header (`Authorization: Bearer <Token>`), là chuẩn chung toàn cầu nên tương thích 100% với cả Web, Mobile App lẫn các thiết bị IoT. Trong kiến trúc Microservices, mỗi dịch vụ nhỏ (như dịch vụ Quản lý ví, dịch vụ Cảnh báo) đều có thể tự xác thực JWT độc lập dựa trên Public/Secret Key mà không cần gọi truy vấn chéo lẫn nhau, giúp tối ưu tốc độ phản hồi API.

---

## Phần 2: Nội dung báo cáo chiến lược & Đề xuất kiến trúc

### 1. Đánh giá ưu và nhược điểm trong ngữ cảnh Blockchain Explorer

#### A. Đối với Session-based Authentication
* **Ưu điểm 1 (Kiểm soát tuyệt đối):** Dễ dàng quản lý và hủy phiên làm việc (Revoke/Logout) của người dùng ngay lập tức từ phía Server nếu phát hiện dấu hiệu gian lận hoặc tấn công phá hoại hệ thống.
* **Ưu điểm 2 (Bảo mật Client):** Dữ liệu lưu trong Cookie với cờ `HttpOnly` và `Secure` giúp chống lại các cuộc tấn công đánh cắp thông tin phổ biến như XSS (Cross-Site Scripting) rất hiệu quả.
* **Nhược điểm 1 (Nghẽn băng thông & Bộ nhớ):** Chi phí phần cứng cực kỳ tốn kém khi lưu trữ trạng thái của hàng triệu người dùng hoạt động đồng thời (Concurrent Users).
* **Nhược điểm 2 (Kém linh hoạt phân tán):** Gây khó khăn lớn trong việc chia sẻ trạng thái phiên giữa các Microservices độc lập và không thân thiện với các kết nối API từ ứng dụng Mobile.

#### B. Đối với JSON Web Token (JWT)
* **Ưu điểm 1 (Mở rộng vô hạn - Stateless):** Server không lưu trạng thái, giúp hệ thống Blockchain Explorer xử lý lượng truy cập khổng lồ (High Traffic) ở quy mô toàn cầu mà hiệu suất không bị suy giảm.
* **Ưu điểm 2 (Tương thích hoàn hảo):** Dễ dàng truyền nhận qua HTTP Header, giúp đồng bộ tính năng theo dõi ví và tạo cảnh báo mượt mà trên cả Web, Android và iOS.
* **Nhược điểm 1 (Khó thu hồi Token):** Do đặc tính Stateless, một khi JWT đã được cấp phát và còn hạn, Server rất khó vô hiệu hóa nó ngay lập tức (trừ khi áp dụng các giải pháp phức tạp như Token Blacklisting bằng Redis).
* **Nhược điểm 2 (Kích thước lớn):** JWT chứa nhiều thông tin (Claims) nên dung lượng lớn hơn nhiều so với Session ID ngắn gọn, làm tăng nhẹ băng thông của mỗi request gửi lên.

### 2. Đề xuất giải pháp và Lý do lựa chọn

**Giải pháp đề xuất:** Chọn giải pháp xác thực dựa trên **JSON Web Token (JWT)**. Tuy nhiên, để hệ thống đạt độ chín muồi về cả hiệu năng lẫn bảo mật, kiến trúc sẽ áp dụng cơ chế **Dual-Token (AccessToken ngắn hạn + RefreshToken dài hạn)** kết hợp lưu trữ Blacklist trên Redis cache.

**Lý do lựa chọn:**
1. **Đáp ứng bài toán quy mô (Scalability):** Hệ thống Blockchain Explorer có đặc thù là lượng người đọc và tracking dữ liệu cực kỳ lớn. Kiến trúc Stateless của JWT là chìa khóa duy nhất giúp hệ thống phân tán tải đều ra các vùng địa lý (qua CDN và các cụm Microservices) mà không lo nghẽn mạng do đồng bộ session.
2. **Tích hợp linh hoạt:** Giúp đội ngũ phát triển dễ dàng tách biệt các dịch vụ (Microservices phụ trách Tracking độc lập với Microservices phụ trách Alert/Notification). Các lập trình viên Mobile và Web chỉ cần dùng chung một chuẩn Header Authorization, giảm thiểu tối đa sự phức tạp khi viết mã nguồn ở các tầng giao diện.
3. **Giải quyết nhược điểm bảo mật:** Bằng cách đặt thời gian sống của `AccessToken` rất ngắn (ví dụ: 15 phút) và quản lý việc cấp lại qua `RefreshToken` (lưu an toàn trong DB/Redis), chúng ta vừa tận dụng được sức mạnh mở rộng của JWT, vừa kiểm soát được rủi ro lộ lọt Token của người dùng.