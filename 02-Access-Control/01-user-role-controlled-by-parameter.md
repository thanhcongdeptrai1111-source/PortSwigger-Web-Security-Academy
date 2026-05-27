
# Lab: Reflected XSS into HTML context with nothing encoded

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng Reflected XSS (XSS phản chiếu) trong tính năng tìm kiếm của trang web.
- Ép trình duyệt của người dùng thực thi mã JavaScript độc hại để hiển thị một hộp thoại thông báo (`alert`).

## 2. Cách khai thác (Exploitation Steps)
1. Truy cập vào trang chủ của bài Lab, tìm đến ô tìm kiếm **Search blog...** ở phía trên bên phải.
2. Nhập chính xác đoạn mã độc JavaScript sau vào ô tìm kiếm:
   ```html
   <script>alert(1)</script>
Bấm nút Search hoặc nhấn Enter.

Kết quả: Một hộp thoại pop-up nhỏ chứa số 1 xuất hiện ngay trên màn hình và bài Lab báo thành công (Solved).

3. Phân tích kỹ thuật (Technical Analysis)
Cách hoạt động thông thường:
Khi người dùng tìm kiếm từ khóa an toàn như hello, Server sẽ nhận vào và phản chiếu dữ liệu đó vào cấu trúc HTML trả về cho trình duyệt:

HTML
<h1>Search results for 'hello'</h1>
Trình duyệt hiểu: Đây là văn bản thuần túy, cần hiển thị chữ hello lên màn hình.

Khi bị chèn Payload:
Lập trình viên Backend đã bê nguyên xi chuỗi dữ liệu đầu vào từ ô tìm kiếm để in thẳng ra giao diện mà không có bất kỳ bộ lọc hay mã hóa nào (nothing encoded). Kết quả là mã nguồn HTML trả về có dạng:

HTML
<h1>Search results for '<script>alert(1)</script>'</h1>
Phân tích logic:
Trình duyệt Web bản chất là một cỗ máy biên dịch đọc mã HTML từ trên xuống dưới.

Khi đọc tới cặp thẻ <script>, "bộ não" của trình duyệt lập tức chuyển trạng thái: nó không coi đây là chữ viết thông thường để hiển thị nữa, mà hiểu rằng "Bắt đầu từ đây là câu lệnh JavaScript hệ thống cần mình thực thi".

Trình duyệt tự động chạy lệnh alert(1) nằm bên trong, khiến hộp thoại pop-up bất ngờ kích nổ.

## 4. Cách khắc phục cho Backend (Remediation)
Lỗi của lập trình viên: Quá tin tưởng dữ liệu từ Client, hiển thị trực tiếp input của người dùng lên ngữ cảnh HTML mà không qua xử lý.

Giải pháp: Sử dụng kỹ thuật HTML Entity Encoding (Mã hóa thực thể HTML) trước khi in bất kỳ dữ liệu nào ra màn hình. Hệ thống sẽ tự động biến các ký tự có khả năng điều khiển cấu trúc HTML thành định dạng văn bản an toàn:

Ký tự < chuyển thành &lt;

Ký tự > chuyển thành &gt;

Khi đã áp dụng bộ lọc mã hóa, mã HTML trả về sẽ là:

HTML
<h1>Search results for '&lt;script&gt;alert(1)&lt;/script&gt;'</h1>
Lúc này, trình duyệt đọc vào sẽ hiểu đây chỉ là các ký tự chữ thô. Nó sẽ in nguyên văn dòng chữ <script>alert(1)</script> lên màn hình cho người dùng đọc bằng mắt chứ không thể "kích nổ" thành câu lệnh, lỗ hổng XSS bị triệt tiêu hoàn toàn.