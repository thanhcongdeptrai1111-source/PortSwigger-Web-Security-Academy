# Lab: SQL injection vulnerability allowing login bypass

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng SQL Injection tại tính năng Đăng nhập (Login).
- Đăng nhập thành công vào tài khoản Quản trị viên tối cao `administrator` mà không cần biết mật khẩu.

## 2. Cách khai thác (Exploitation Steps)
1. Truy cập vào mục **My account** trên trang web bài Lab.
2. Tại ô **Username**, nhập vào đoạn payload:
   ```text
   administrator'-- 
(Lưu ý: Có một dấu cách/khoảng trắng ngay sau hai dấu gạch ngang).
3. Tại ô Password, nhập một mật khẩu bất kỳ (ví dụ: 123).
4. Mở Burp Suite, bật Intercept is on và bấm nút Log in.
5. Gói tin POST /login bị giữ lại. Bạn bôi đen toàn bộ đoạn dữ liệu trong ô Username (administrator'-- ), nhấn Ctrl + U để mã hóa URL (chuỗi sẽ biến thành administrator'%2d%2d+).
6. Bấm Forward để gửi gói tin đi và tắt Intercept.
7. Kết quả: Hệ thống đăng nhập thẳng bạn vào giao diện quản trị của administrator và bài Lab báo thành công (Solved).

## 3. Phân tích kỹ thuật (Technical Analysis)
Câu lệnh kiểm tra đăng nhập gốc trên Server:
SQL
SELECT * FROM users WHERE username = 'input_user' AND password = 'input_password'

Logic thông thường: Phép toán AND bắt buộc câu lệnh chỉ thành công khi đồng thời cả 2 vế đều đúng (Tìm thấy đúng Username VÀ khớp đúng Password).

Câu lệnh sau khi bị chèn Payload:
SQL
SELECT * FROM users WHERE username = 'administrator'-- ' AND password = '123'

## Phân tích logic:
Dấu nháy đơn ('): Đóng phân vùng dữ liệu chuỗi tên đăng nhập, giúp chữ administrator khớp hoàn hảo với điều kiện username = 'administrator'.

Dấu comment (-- ): Biến toàn bộ đoạn mã kiểm tra mật khẩu phía sau (AND password = '123') thành một dòng chú thích vô hại. Database gặp dấu này sẽ lập tức phớt lờ và bỏ qua toàn bộ phần phía sau.

Database nhận lệnh và thực thi: "Tìm xem có ai tên là administrator không?". Vì tài khoản này có thật, Database trả về thông tin User hợp lệ cho Backend. Đoạn code Backend thấy có dữ liệu trả về liền tin rằng người dùng đã đăng nhập đúng và cấp quyền truy cập ngay lập tức mà chưa từng kiểm tra mật khẩu.

## 4. Cách khắc phục cho Backend (Remediation)
Lỗi của lập trình viên: Trực tiếp nối chuỗi dữ liệu đầu vào của người dùng vào câu lệnh SQL truy vấn đăng nhập.

Giải pháp: Sử dụng cơ chế Parameterized Queries (Truy vấn có tham số) kết hợp với việc băm mật khẩu (Hashing).

## JavaScript
// Ví dụ minh họa an toàn trong Node.js bằng Parameterized Query
const query = 'SELECT * FROM users WHERE username = ?';
db.execute(query, [input_user], function(err, results) {
    // Sau đó mới lấy mật khẩu đã băm (hash) trong Database ra 
    // để so sánh bằng thư viện bảo mật (như bcrypt) ở tầng code Backend
});
## Khi áp dụng tham số hóa, chuỗi administrator'--  sẽ bị ép hiểu hoàn toàn là một cái tên Username thô. Hệ thống sẽ đi tìm người dùng có tên chính xác là administrator'--  (người dùng này không tồn tại), cuộc tấn công thất bại hoàn toàn.