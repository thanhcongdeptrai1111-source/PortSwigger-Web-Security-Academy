# Lab: SQL injection UNION attack, finding a column containing text

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng SQL Injection dạng UNION-based.
- Dò tìm và xác định chính xác cột nào trong tập kết quả trả về của câu lệnh truy vấn gốc hỗ trợ kiểu dữ liệu chuỗi văn bản (String/Text).
- Chèn thành công một chuỗi ký tự ngẫu nhiên do hệ thống yêu cầu vào đúng cột để hoàn thành bài Lab.

## 2. Cách khai thác (Exploitation Steps)
1. **Thu thập gói tin gốc:**
   - Tắt chặn gói tin (`Intercept is off`).
   - Trên giao diện web, bấm chọn một danh mục bất kỳ để sinh gói tin (Ví dụ: `Pets`).
   - Vào tab **Proxy > HTTP history**, tìm gói tin `GET /filter?category=Pets`.
   - Click chuột phải chọn **Send to Repeater** (`Ctrl + R`).
2. **Xác định số lượng cột:**
   - Tại tab **Repeater**, dùng kỹ thuật `ORDER BY` để tìm số cột:
     ```text
     Pets' ORDER BY 1--
     Pets' ORDER BY 2--
     ...
     ```
   - Giả sử hệ thống báo lỗi ở `ORDER BY 4--`, suy ra câu lệnh gốc có **3 cột**. Xác nhận lại bằng payload `Pets' UNION SELECT NULL, NULL, NULL--` thấy trả về `200 OK`.
3. **Dò tìm cột chứa kiểu dữ liệu String:**
   - Lấy chuỗi ký tự ngẫu nhiên do bài Lab yêu cầu (Ví dụ đề bài cho chuỗi: `'abcdef'`).
   - Thử nghiệm thay thế chuỗi này vào từng vị trí `NULL` một:
     - *Thử cột 1:* `Pets' UNION SELECT 'abcdef', NULL, NULL--` $\rightarrow$ Kết quả: Lỗi `500 Internal Server Error` (Cột 1 không phải kiểu chuỗi).
     - *Thử cột 2:* `Pets' UNION SELECT NULL, 'abcdef', NULL--` $\rightarrow$ Kết quả: Trả về thành công `200 OK` và chuỗi xuất hiện trên giao diện.
4. **Hoàn thành:**
   - Khi đặt chuỗi vào đúng cột hỗ trợ văn bản, Server sẽ thực thi thành công câu lệnh và in chuỗi đó ra màn hình. Bài Lab chuyển trạng thái sang `Solved`.

## 3. Phân tích kỹ thuật (Technical Analysis)

### Quy tắc nghiêm ngặt về kiểu dữ liệu trong toán tử UNION:
Không chỉ yêu cầu khớp về **số lượng cột**, toán tử `UNION` trong các hệ quản trị cơ sở dữ liệu (như Oracle, SQL Server, PostgreSQL, MySQL) còn bắt buộc **kiểu dữ liệu của các cột tương ứng giữa hai câu lệnh phải tương thích với nhau**.



### Bản chất hành vi của Database:
- Nếu cột số 1 trong Database được định nghĩa là kiểu số nguyên (`INT` - ví dụ dùng lưu ID sản phẩm), việc Hacker cố tình chèn một chuỗi văn bản `'abcdef'` vào cột này qua lệnh UNION sẽ khiến Database không thể ép kiểu (Type conversion) được, dẫn đến xung đột dữ liệu và văng lỗi Hệ thống hệ quản trị cơ sở dữ liệu (`Conversion failed when converting the varchar value...`).
- Khi Hacker chèn trúng cột có kiểu dữ liệu là `VARCHAR` hoặc `TEXT` (ví dụ cột tên sản phẩm hoặc mô tả), Database sẽ chấp nhận hoàn toàn và gộp chuỗi độc hại đó vào luồng hiển thị HTML trả về cho người dùng.

## 4. Cách khắc phục cho Backend (Remediation)
Tương tự như các bài SQL Injection trước, giải pháp triệt để nhất là tuyệt đối không nối chuỗi dữ liệu từ Client vào câu lệnh SQL. 

```javascript
// Khắc phục an toàn bằng Parameterized Queries (Truy vấn tham số hóa) trong Node.js
const query = 'SELECT id, product_name, price FROM products WHERE category = ?';
db.execute(query, [req.query.category], function(err, results) {
    if (err) throw err;
    // Xử lý dữ liệu an toàn trả về cho client
});
Cơ chế tham số hóa (? hoặc $1) sẽ xử lý toàn bộ chuỗi nhập vào như một giá trị văn bản thô (Literal value). Kể cả khi chuỗi nhập vào là ' UNION SELECT NULL, 'abcdef', NULL--, Database cũng chỉ hiểu đó là tên của một danh mục và đi tìm kiếm danh mục có tên dài ngoằng như vậy, bẻ gãy hoàn toàn khả năng can thiệp vào cấu trúc câu lệnh SQL.