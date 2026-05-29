Markdown
# Lab: SQL injection UNION attack, retrieving data from other tables

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng UNION-based SQL Injection để trích xuất dữ liệu nhạy cảm từ một bảng hoàn toàn độc lập trong Database.
- Thu thập thành công danh sách tài khoản (`username`, `password`) từ bảng `users`.
- Chiếm quyền điều khiển tài khoản quản trị viên `administrator` để hoàn thành bài Lab.

## 2. Cách khai thác (Exploitation Steps)
1. **Phân tích cấu trúc câu lệnh gốc:**
   - Tắt chặn gói tin (`Intercept is off`).
   - Chọn một danh mục sản phẩm trên giao diện, lấy gói tin `GET /filter?category=...` chuyển sang tab **Repeater** (`Ctrl + R`).
   - Thử nghiệm với kỹ thuật `ORDER BY` nhằm tìm kiếm số lượng cột:
     ```text
     Gifts' ORDER BY 2-- (Trả về 200 OK)
     Gifts' ORDER BY 3-- (Trả về 500 Internal Server Error)
     ```
     $\rightarrow$ Khẳng định: Câu lệnh gốc trả về **2 cột**.
2. **Kiểm tra khả năng tương thích kiểu dữ liệu (Data Type):**
   - Thử nghiệm chèn giá trị chuỗi vào cả 2 cột thông qua toán tử UNION:
     ```text
     Gifts' UNION SELECT 'a', 'b'--
     ```
   - Kết quả trả về mã lỗi `200 OK`, chứng tỏ cả hai cột này đều có kiểu dữ liệu là chuỗi văn bản (`String`/`Varchar`), hoàn toàn phù hợp để chứa thông tin tài khoản.
3. **Thực hiện trích xuất dữ liệu (Data Extraction):**
   - Đề bài đã cung cấp thông tin cấu trúc: Tên bảng là `users`, tên 2 cột lần lượt là `username` và `password`.
   - Tiến hành thay thế giá trị payload tại tham số `category` để ép hệ thống gộp dữ liệu từ bảng `users`:
     ```text
     Gifts' UNION SELECT username, password FROM users--
     ```
   - Nhấn **Send**.
4. **Thu hoạch kết quả & Đăng nhập:**
   - Tại ô **Response**, cuộn chuột xuống phần nội dung HTML hiển thị. 
   - Tìm kiếm cấu trúc danh sách, thu thập được cặp giá trị tài khoản quản trị:
     * **Username:** `administrator`
     * **Password:** `[Chuỗi_mật_khẩu_ngẫu_nhiên]`
   - Quay lại trình duyệt, truy cập mục **My account**, đăng nhập bằng thông tin vừa thu thập được và hoàn thành bài Lab (`Solved`).

## 3. Phân tích kỹ thuật (Technical Analysis)

### Cơ chế gộp kết quả từ bảng độc lập:
Khi lập trình viên nối chuỗi thô từ Client vào câu lệnh truy vấn, họ không lường trước việc kẻ tấn công có thể thay đổi hoàn toàn logic thực thi của Database thông qua toán tử `UNION`.



### Bản chất tiến trình xử lý tại Database:
Câu lệnh thực tế sau khi bị chèn ép payload sẽ có dạng:
```sql
SELECT name, description FROM products WHERE category = 'Gifts' UNION SELECT username, password FROM users--'
Hệ thống thực thi vế đầu: Lấy ra toàn bộ name và description của các sản phẩm thuộc danh mục Gifts.

Gặp toán tử UNION: Ép hệ thống chạy tiếp vế sau: SELECT username, password FROM users. Do 2 cột này đồng nhất về số lượng (2 cột) và kiểu dữ liệu (đều là text) với vế đầu nên Database xử lý mượt mà.

Ký tự comment -- bẻ gãy toàn bộ các ràng buộc hoặc dấu nháy đơn còn sót lại phía sau.

Kết quả của bảng dữ liệu tài khoản người dùng (users) được trộn thẳng vào luồng kết quả trả về của sản phẩm, hiển thị trực tiếp ra màn hình HTML của kẻ tấn công.

4. Cách khắc phục cho Backend (Remediation)
Để ngăn chặn việc rò rỉ dữ liệu qua kỹ thuật UNION Injection, giải pháp cốt lõi duy nhất là áp dụng Parameterized Queries (Truy vấn tham số hóa).

JavaScript
// Mã nguồn Backend Node.js được sửa đổi an toàn bằng Parameterized Queries
const query = 'SELECT name, description FROM products WHERE category = ?';

db.execute(query, [req.query.category], function(err, results) {
    if (err) {
        // Xử lý lỗi an toàn, không lộ thông tin hệ thống
        return res.status(500).send("System Error");
    }
    // Trả về kết quả an toàn cho Client
    res.render('filter', { products: results });
});
Cơ chế bảo vệ:
Khi truyền tham số thông qua mảng tách biệt [req.query.category], trình điều khiển Database (Database Driver) sẽ tự động biên dịch câu lệnh gốc trước, sau đó mới đưa giá trị chuỗi nhập vào sau.

Kể cả khi chuỗi chứa mã độc UNION SELECT username, password FROM users--, Database vẫn xử lý nó như một hằng chuỗi thông thường và đi tìm kiếm danh mục có tên giống hệt như thế, loại bỏ hoàn toàn khả năng can thiệp hay cấu trúc lại câu lệnh SQL.