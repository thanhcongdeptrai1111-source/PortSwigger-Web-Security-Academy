Markdown
# Lab: SQL injection UNION attack, determining the number of columns returned by the query

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng SQL Injection dạng UNION-based.
- Xác định chính xác số lượng cột (columns) mà câu lệnh truy vấn gốc của Server trả về để làm bước đệm cho việc trích xuất dữ liệu nhạy cảm.

## 2. Cách khai thác (Exploitation Steps)
1. **Thu thập gói tin:**
   - Tắt chặn gói tin (`Intercept is off`).
   - Trên giao diện bài Lab, bấm chọn một danh mục sản phẩm bất kỳ (ví dụ: `Gifts`).
   - Vào tab **Proxy > HTTP history** trong Burp Suite, tìm gói tin `GET /filter?category=Gifts`.
   - Nhấp chuột phải chọn **Send to Repeater** (`Ctrl + R`).
2. **Thăm dò bằng kỹ thuật ORDER BY (Cách 1):**
   - Tại tab **Repeater**, chèn payload kiểm tra vào tham số `category`:
     ```text
     Gifts' ORDER BY 1--
     ```
   - Bấm **Send**, hệ thống trả về mã lỗi `200 OK` (Cột 1 tồn tại).
   - Tiếp tục tăng dần số thứ tự: `ORDER BY 2--`, `ORDER BY 3--`... cho đến khi tăng tới `ORDER BY 4--` thì Server lập tức trả về lỗi **`500 Internal Server Error`**.
   - **Kết luận:** Vì cột số 4 không tồn tại dẫn đến lỗi, suy ra số lượng cột tối đa của câu lệnh gốc là **3 cột**.
3. **Xác nhận lại bằng kỹ thuật UNION SELECT NULL (Cách 2):**
   - Đổi payload sang toán tử UNION với đúng 3 giá trị `NULL` tương ứng với 3 cột vừa tìm được:
     ```text
     Gifts' UNION SELECT NULL, NULL, NULL--
     ```
   - Bấm **Send**, hệ thống trả về `200 OK` mượt mà và bài Lab lập tức báo thành công (`Solved`).

## 3. Phân tích kỹ thuật (Technical Analysis)

### Nguyên lý của toán tử UNION:
Toán tử `UNION` trong SQL cho phép gộp kết quả của hai hoặc nhiều câu lệnh `SELECT` lại thành một tập kết quả duy nhất. Tuy nhiên, Database áp đặt một quy tắc cực kỳ nghiêm ngặt: **Câu lệnh SELECT thứ hai bắt buộc phải có cùng số lượng cột và cùng kiểu dữ liệu tương thích với câu lệnh SELECT gốc.**



### Phân tích Payload:
1. **Cơ chế `ORDER BY X`:** Lệnh này bắt Database sắp xếp kết quả theo số thứ tự của cột. Khi Hacker dò tới `ORDER BY 4--` mà bảng gốc chỉ có 3 cột, Database không tìm thấy cột số 4 để sắp xếp nên lập tức tung ngoại lệ lỗi (`Internal Server Error`). Nhờ dấu hiệu lỗi này, Hacker biết được số lượng cột chính xác.
2. **Cơ chế `UNION SELECT NULL, NULL, NULL--`:** Giá trị `NULL` có thể đại diện cho mọi kiểu dữ liệu (chuỗi, số, ngày tháng...). Việc chèn 3 chữ `NULL` giúp câu lệnh bổ sung khớp hoàn hảo với 3 cột của câu lệnh gốc, đánh lừa Database thực thi thành công mà không bị xung đột kiểu dữ liệu.

## 4. Cách khắc phục cho Backend (Remediation)
- **Sai lầm:** Nối trực tiếp chuỗi danh mục từ Client vào câu lệnh SQL:
  ```sql
  -- Code không an toàn của lập trình viên
  SELECT name, description, price FROM products WHERE category = 'Gifts'
Giải pháp: Sử dụng Parameterized Queries (Truy vấn có tham số).

JavaScript
// Khắc phục an toàn trong Node.js (MySQL/PostgreSQL)
const query = 'SELECT name, description, price FROM products WHERE category = ?';
db.execute(query, [req.query.category], function(err, results) {
    // Xử lý kết quả an toàn
});
Khi tham số hóa, toàn bộ chuỗi dữ liệu đầu vào dù chứa các từ khóa nguy hiểm như UNION hay ORDER BY đều bị ép hiểu là một chuỗi văn bản thô (Literal string). Database sẽ đi tìm danh mục có tên chính xác là Gifts' ORDER BY 1-- thay vì thực thi nó như một câu lệnh, bẻ gãy hoàn toàn cuộc tấn công.