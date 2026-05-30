# Lab: SQL injection attack, querying the database type and version on Oracle

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng UNION-based SQL Injection trên hệ quản trị cơ sở dữ liệu **Oracle**.
- Nắm vững và áp dụng quy tắc cú pháp nghiêm ngặt của Oracle (Bắt buộc phải có mệnh đề `FROM`).
- Trích xuất thành công thông tin phiên bản (Version) từ bảng hệ thống `v$version` để hoàn thành bài Lab.

## 2. Cách khai thác (Exploitation Steps)
1. **Thu thập và chuẩn bị gói tin:**
   - Trên giao diện web, chọn một danh mục sản phẩm (Ví dụ: `Gifts`) để sinh gói tin.
   - Truy cập tab **Proxy > HTTP history**, gửi gói tin `GET /filter?category=Gifts` sang tab **Repeater** (`Ctrl + R`).
2. **Dò tìm số lượng cột (Tuân thủ luật Oracle):**
   - Khác với các Database khác, Oracle không cho phép `SELECT` mà không có nguồn bảng. Do đó, ta phải mượn bảng mặc định `DUAL` để dò số cột.
   - Thử nghiệm payload tại tham số `category`:
     ```text
     Gifts' UNION SELECT NULL FROM DUAL--
     ```
     $\rightarrow$ Kết quả: Lỗi `500 Internal Server Error`.
   - Tăng số lượng cột:
     ```text
     Gifts' UNION SELECT NULL, NULL FROM DUAL--
     ```
     $\rightarrow$ Kết quả: Trả về thành công `200 OK`. Xác định hệ thống gốc có **2 cột**.
3. **Trích xuất thông tin phiên bản Database:**
   - Để lấy phiên bản của Oracle, ta sử dụng bảng hệ thống mã nguồn là `v$version` và trường dữ liệu tương ứng là `banner`.
   - Thay thế payload thành:
     ```text
     Gifts' UNION SELECT banner, NULL FROM v$version--
     ```
   - Bôi đen toàn bộ đoạn mã độc, nhấn **`Ctrl + U`** để thực hiện URL Encode trên Burp Suite nhằm đảm bảo các ký tự đặc biệt được truyền đi chính xác. Nhấn **Send**.
4. **Hoàn thành:**
   - Tại ô **Response**, hệ thống hiển thị thông tin dạng: `Oracle Database 11g Express Edition Release...` trực tiếp trên mã HTML. Bài Lab chuyển sang trạng thái `Solved`.

## 3. Phân tích kỹ thuật (Technical Analysis)

### Sự khác biệt chí mạng về kiến trúc của Oracle:
Các hệ quản trị cơ sở dữ liệu như MySQL, PostgreSQL khá tự do, cho phép thực thi lệnh `SELECT 'chuỗi_thô'` mà không cần chỉ định bảng dữ liệu nguồn. Ngược lại, Oracle coi đây là một lỗi cú pháp nghiêm trọng. 



Để giải quyết vấn đề kiểm thử hoặc chạy các hàm độc lập, Oracle thiết kế sẵn một bảng hệ thống ẩn mang tên **`DUAL`**. Bảng này chỉ có duy nhất 1 cột tên là `DUMMY` và chứa 1 dòng giá trị `'X'`. Khi hacker cần điền đầy đủ cú pháp mà không muốn bốc dữ liệu thực tế, `FROM DUAL` là giải pháp bắt buộc.

### Tiến trình xử lý câu lệnh tại Database:
Câu lệnh hoàn chỉnh bị thao túng trên Server:
```sql
SELECT id, description FROM products WHERE category = 'Gifts' UNION SELECT banner, NULL FROM v$version--'
Vế đầu tiên lấy ra danh sách sản phẩm thuộc danh mục Gifts.

Vế sau thực hiện gộp kết quả (UNION) bằng cách truy cập vào bảng thông tin hệ thống v$version, bốc dữ liệu từ cột banner (chứa thông tin phiên bản chi tiết) nạp vào cột số 1, cột số 2 được bù đầy bằng giá trị trống NULL.

Do hai vế đồng nhất về số lượng cột (2 cột) và kiểu dữ liệu (đều hỗ trợ chuỗi văn bản), câu lệnh thực thi trơn tru và đẩy thẳng thông tin nhạy cảm của lõi Database ra màn hình người dùng.

4. Cách khắc phục cho Backend (Remediation)
Để chặn đứng hoàn toàn việc kẻ tấn công lợi dụng UNION để trích xuất thông tin cấu trúc hệ thống, Backend cần tuyệt đối áp dụng cơ chế Parameterized Queries (Truy vấn có tham số).

JavaScript
// Khắc phục an toàn cho Backend Node.js sử dụng Oracle Database Driver
const sql = `SELECT id, description FROM products WHERE category = :cat`;
const binds = [req.query.category]; // Tham số hóa cô lập dữ liệu đầu vào

connection.execute(sql, binds, function(err, result) {
    if (err) {
        // Ghi log bảo mật nội bộ và trả về thông báo lỗi chung
        return res.status(500).send("Hệ thống bảo trì.");
    }
    // Trả về kết quả an toàn
});
Khi áp dụng kỹ thuật này, chuỗi ký tự ' UNION SELECT banner, NULL FROM v$version-- sẽ bị Database Driver ép hiểu hoàn toàn thành một giá trị chuỗi thuần túy (Literal value). Hệ thống sẽ tìm kiếm trong bảng products xem có danh mục nào tên dài như đoạn mã độc kia không. Vì không có, hệ thống chỉ đơn giản trả về một mảng rỗng, bẻ gãy hoàn toàn khả năng can thiệp cấu trúc SQL.