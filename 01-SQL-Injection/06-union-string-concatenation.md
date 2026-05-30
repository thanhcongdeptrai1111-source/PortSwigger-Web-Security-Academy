# Lab: SQL injection UNION attack, retrieving multiple values in a single column

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng UNION-based SQL Injection trong kịch bản hệ thống bị giới hạn số lượng cột hiển thị dạng chuỗi.
- Áp dụng kỹ thuật nối chuỗi (String Concatenation) để gom nhiều trường dữ liệu nhạy cảm (`username` và `password`) ép hiển thị vào chung một cột duy nhất.
- Thu thập thông tin tài khoản và đăng nhập thành công với tư cách `administrator`.

## 2. Cách khai thác (Exploitation Steps)
1. **Thám thính cấu trúc bảng gốc:**
   - Sử dụng kỹ thuật `ORDER BY` nhằm xác định số cột, tìm ra câu lệnh gốc trả về **2 cột**.
   - Tiến hành kiểm tra kiểu dữ liệu bằng cách thử nghiệm chèn ký tự chữ `'a'` vào từng cột:
     - Thử cột 1: `... UNION SELECT 'a', NULL--` $\rightarrow$ Trả về `500 Internal Server Error` (Cột 1 không hỗ trợ kiểu chữ).
     - Thử cột 2: `... UNION SELECT NULL, 'a'--` $\rightarrow$ Trả về `200 OK` (Cột 2 chính xác là kiểu chuỗi văn bản).
   - **Chiến lược:** Vì chỉ có cột 2 nhận kiểu dữ liệu chuỗi, bắt buộc phải giữ cột 1 là `NULL` và dồn toàn bộ dữ liệu cần bốc vào cột 2.
2. **Thực hiện nối chuỗi trích xuất dữ liệu:**
   - Do hệ thống sử dụng cơ sở dữ liệu PostgreSQL, cú pháp nối chuỗi sẽ sử dụng toán tử hai dấu gạch đứng `||`.
   - Chèn payload cấu trúc nối chuỗi, phân tách tài khoản và mật khẩu bằng ký tự đặc biệt `~`, đồng thời chỉ định rõ nguồn từ bảng `users`:
     ```text
     Toys & Games' UNION SELECT NULL, username || '~' || password FROM users--
     ```
   - Thực hiện URL Encode (`Ctrl + U`) đoạn payload trên tab **Repeater** và bấm **Send**.
3. **Thu hoạch kết quả:**
   - Tại ô **Response**, hệ thống trả về mã lỗi `200 OK`. Cuộn xuống phần nội dung hiển thị sẽ thấy danh sách dữ liệu được định dạng rõ ràng:
     `administrator~[chuỗi_mật_khẩu_ngẫu_nhiên]`
   - Sử dụng mật khẩu thu được quay lại giao diện đăng nhập thành công tài khoản quản trị để kết thúc bài Lab (`Solved`).

## 3. Phân tích kỹ thuật (Technical Analysis)

### Thử thách thực tế về giới hạn cột:
Trong các cuộc tấn công UNION-based ngoài thực tế, rất hiếm khi Hacker gặp được kịch bản lý tưởng khi bảng gốc có số lượng cột trùng khớp hoàn toàn với số trường muốn lấy, hoặc tất cả các cột đều là kiểu chuỗi. Nếu cố tình ép Database nhét chuỗi vào cột chứa Số (`INT`), hệ thống sẽ gãy logic và văng lỗi sập trang ngay lập tức.



### Bản chất tiến trình xử lý nối chuỗi tại Database (PostgreSQL):
Câu lệnh hoàn chỉnh sau khi bị can thiệp có dạng:
```sql
SELECT id, description FROM products WHERE category = 'Toys & Games' UNION SELECT NULL, username || '~' || password FROM users--'
Toán tử || thực hiện phép liên kết các chuỗi ký tự thành một chuỗi văn bản duy nhất. Dữ liệu của hai cột tách biệt trong bảng users được đóng gói gọn gàng thành một giá trị dạng admin~pass123.

Giá trị tổng hợp này khớp hoàn toàn với kiểu dữ liệu VARCHAR/TEXT của cột số 2 (description) trong câu lệnh gốc, giúp vượt qua bộ kiểm tra kiểu dữ liệu nghiêm ngặt của Database và hiển thị trơn tru ra ngoài màn hình.

4. Cách khắc phục cho Backend (Remediation)
Giải pháp triệt để nhất để loại bỏ hoàn toàn các nguy cơ tấn công SQL Injection (bao gồm cả dạng UNION lẫn nối chuỗi dữ liệu) là triển khai cơ chế Parameterized Queries (Truy vấn có tham số).

JavaScript
// Khắc phục mã nguồn Backend an toàn tuyệt đối bằng Node.js (PostgreSQL)
const query = {
    text: 'SELECT id, description FROM products WHERE category = $1',
    values: [req.query.category], // Toàn bộ payload đầu vào bị cô lập thành dữ liệu thô
};

pool.query(query, (err, res) => {
    if (err) {
        // Ghi log lỗi hệ thống nội bộ, trả về thông báo chung chung cho client
        return response.status(500).send("An error occurred");
    }
    // Render kết quả an toàn
});
Khi xử lý bằng cơ chế này, chuỗi dữ liệu đầu vào dù chứa các toán tử phức tạp như UNION, ||, hay FROM users-- đều bị trình điều khiển cơ sở dữ liệu xử lý như một chuỗi văn bản thuần túy (Literal literal). Nó sẽ tìm kiếm một danh mục có tên chính xác là chuỗi mã độc đó, ngăn chặn hoàn toàn việc can thiệp cấu trúc logic của câu lệnh SQL.