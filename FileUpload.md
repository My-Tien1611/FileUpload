# Tìm hiểu về File Upload
- Người thực hiện: Lê Thị Mỹ Tiên
- Cập nhật lần cuối
# Mục lục
# Nội dung
## 1. Tìm kiếm các chức năng Upload:
### 1.1 Khái niệm - tác động:
- Đây là lỗ hổng xảy ra khi máy chủ web cho phép người dùng tải tệp lên nhưng không kiểm soát đầy đủ các thuộc tính của tệp (tên, loại, nội dung, kích thước).
- Tùy thuộc vào mức độ kiểm soát mà hệ thống thực hiện, hậu quả có thể khác nhau:
  + **Thực thi mã từ xa (nguy hiểm nhất)**: Nếu hệ thống không kiểm tra loại file và cho phép upload file có thể chạy trên server (`.php`, `.jsp`, `.asp`…), hacker có thể triển khai web shell → chiếm quyền toàn bộ máy chủ.
  + **Ghi đè hoặc thay thế file quan trọng**: Nếu không kiểm tra tên file, attacker có thể upload một file trùng tên với file hệ thống hoặc ứng dụng → ghi đè file gốc → phá hoại hoặc chiếm quyền.
  + **Chiếm quyền truy cập thư mục bất ngờ**: Nếu hệ thống kết hợp với lỗ hổng duyệt thư mục (directory traversal), kẻ tấn công có thể đưa file vào thư mục quan trọng mà quản trị không lường trước.
  + **Tấn công từ chối dịch vụ (DoS)**: Nếu không giới hạn dung lượng file, hacker có thể upload file cực lớn → làm đầy ổ đĩa, khiến hệ thống ngưng hoạt động.
### 1.2 Lỗ hổng File Upload cách phát sinh:
Lỗ hổng File Upload phát sinh chủ yếu do việc kiểm soát không toàn diện hoặc không chặt chẽ: dựa vào blacklist, xác minh thuộc tính dễ bị giả mạo, hoặc triển khai không đồng nhất → tạo ra “lỗ hổng” cho attacker lợi dụng để tải file nguy hiểm.
### 1.3 Máy chủ web xử lý các yêu cầu về tệp tĩnh
- **Cơ chế cơ bản**:
Khi có một yêu cầu HTTP, máy chủ sẽ phân tích đường dẫn (URL) để xác định phần mở rộng tệp → Sau đó, so sánh phần mở rộng này với danh sách ánh xạ MIME (MIME type mapping) đã được cấu hình sẵn → tùy theo loại tệp, máy chủ sẽ quyết định cách xử lý tiếp theo.
- Các tình huống:
  + **Tệp tĩnh không thực thi (ảnh, CSS, HTML thuần)**: Máy chủ đọc nội dung file và trả về cho client trong phản hồi HTTP.
  + **Tệp có thể thực thi (ví dụ PHP, JSP, ASP) và máy chủ hỗ trợ**: Máy chủ gán biến từ tham số HTTP → chạy file script → gửi kết quả đầu ra (HTML, JSON, v.v.) về cho client. 👉 Đây chính là cơ chế khiến việc upload file script có thể dẫn đến Remote Code Execution.
  + **Tệp có thể thực thi nhưng máy chủ không hỗ trợ**: Thông thường: máy chủ trả về lỗi. Nếu cấu hình sai: máy chủ có thể trả lại nội dung mã nguồn dưới dạng text → gây rò rỉ thông tin nhạy cảm (source code disclosure).

----

## 2. Kỹ thuật Upload Backdoor:
### 2.1 Khai thác việc tải tệp không giới hạn để triển khai shell web

### 2.2 Khai thác lỗi xác thực khi tải tệp lên
