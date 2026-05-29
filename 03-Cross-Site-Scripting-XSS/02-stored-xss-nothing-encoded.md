# Lab: Stored XSS into HTML context with nothing encoded

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng Stored XSS (XSS lưu trữ) trong tính năng bình luận của bài viết.
- Lưu thành công mã độc JavaScript vào Cơ sở dữ liệu (Database) của Server để ép trình duyệt của bất kỳ ai truy cập vào bài viết đều bị thực thi mã độc (`alert`).

## 2. Cách khai thác (Exploitation Steps)
*Bài này dữ liệu được chèn và lưu trực tiếp từ giao diện nên không cần sử dụng Burp Suite để can thiệp gói tin.*

1. Tại trang chủ của bài Lab, bấm chọn vào một bài viết bất kỳ (**View post**).
2. Cuộn xuống khu vực bình luận (**Leave a comment**).
3. Tại ô **Comment**, chèn đoạn payload JavaScript sau:
   ```html
   <script>alert(1)</script>
Các ô còn lại điền thông tin bất kỳ (Ví dụ: Name: hacker, Email: test@test.com).

Bấm nút Post comment để gửi dữ liệu lên Server.

Bấm vào liên kết Back to blog để quay lại trang bài viết.

Kết quả: Ngay khi trang được tải lại, một hộp thoại pop-up alert(1) lập tức xuất hiện trên màn hình và bài Lab báo thành công (Solved).

3. Phân tích kỹ thuật (Technical Analysis)
Sự khác biệt chí mạng của Stored XSS:
Đối với Reflected XSS, mã độc chỉ nằm tạm thời trên URL hoặc Request của chính nạn nhân. Nhưng với Stored XSS, mã độc đã chính thức "định cư" bên trong Database của hệ thống.

Cơ chế hoạt động trong bài Lab:
Giai đoạn lưu trữ (Storage): Khi bấm Post comment, Backend nhận chuỗi <script>alert(1)</script> và thực thi câu lệnh INSERT INTO comments ... để lưu thẳng đoạn văn bản này vào Cơ sở dữ liệu mà không hề qua bộ lọc hay mã hóa nào.

Giai đoạn kích nổ (Execution): Khi bất kỳ người dùng nào (bao gồm cả nạn nhân hoặc Admin) bấm vào xem bài viết này, Backend sẽ chạy lệnh SELECT để lấy tất cả bình luận ra và in trực tiếp vào cấu trúc HTML trả về cho Client:

HTML
<p><script>alert(1)</script></p>
Trình duyệt của nạn nhân khi render tới đoạn này sẽ hiểu đây là một thẻ script thực thi của hệ thống, tự động chạy lệnh alert(1) ngay lập tức. Lỗ hổng này cực kỳ nguy hiểm vì kẻ tấn công chỉ cần ra tay một lần nhưng có thể tiêm nhiễm mã độc đến hàng ngàn người xem sau đó.

4. Cách khắc phục cho Backend (Remediation)
Để chống lại Stored XSS, lập trình viên Backend phải tuân thủ nghiêm ngặt hai lớp phòng thủ:

HTML Entity Encoding (Bắt buộc khi hiển thị): Trước khi in dữ liệu từ Database ra giao diện HTML, phải chuyển đổi các ký tự nguy hiểm thành thực thể an toàn:

< biến thành &lt;

> biến thành &gt;
Mã nguồn trả về lúc này sẽ là &lt;script&gt;alert(1)&lt;/script&gt;, trình duyệt chỉ hiển thị dòng chữ thô chứ không thực thi lệnh.

Sử dụng thư viện Sanitize (Khi nhận đầu vào): Nếu tính năng yêu cầu người dùng được phép nhập định dạng giàu (như Text Editor, thẻ HTML để in đậm, in nghiêng), hãy sử dụng các thư viện kiểm duyệt nổi tiếng như DOMPurify hoặc sanitize-html trên Backend để chủ động bóc tách và loại bỏ tận gốc các thẻ nguy hiểm như <script>, onload, onerror trước khi lưu vào Database.

JavaScript
// Ví dụ minh họa sử dụng thư viện sanitize-html trong Node.js
const sanitizeHtml = require('sanitize-html');
const cleanComment = sanitizeHtml(req.body.comment, {
    allowedTags: [ 'b', 'i', 'em', 'strong', 'a' ], // Chỉ cho phép các thẻ an toàn này
    allowedAttributes: { 'a': [ 'href' ] }
});
// Lưu cleanComment vào Database thay vì dữ liệu thô ban đầu