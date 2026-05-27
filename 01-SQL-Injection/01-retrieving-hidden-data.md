# Lab: SQL injection vulnerability in WHERE clause allowing retrieval of unreleased data

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng SQL Injection tại bộ lọc danh mục sản phẩm (`category`).
- Buộc hệ thống phải hiển thị toàn bộ sản phẩm trong Database, bao gồm cả những sản phẩm chưa được phát hành (unreleased products).

## 2. Cách khai thác (Exploitation Steps)
1. Truy cập vào trang web, bấm chọn một danh mục bất kỳ (ví dụ: `Gifts`).
2. Mở Burp Suite, bật `Intercept is on`.
3. Bấm tải lại trang (F5) để Burp Suite bắt được gói tin `GET /filter?category=Gifts`.
4. Sửa đổi tham số `category` trong gói tin thành:
   ```text
   Gifts' OR 1=1--
## 3. Sửa code 
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
1. Dấu nháy đơn ('): Đóng chuỗi dữ liệu 'Gifts' một cách có chủ đích, giúp tách đoạn mã phía sau ra khỏi vùng văn bản thuần túy.

2. Mệnh đề OR 1=1: Phép toán logic "Hoặc". Vì 1=1 là một điều kiện luôn luôn đúng (Luôn là TRUE), nên toàn bộ mệnh đề WHERE phía trước sẽ luôn trả về TRUE bất kể danh mục là gì.

3. Dấu comment (-- ): Kích hoạt tính năng chú thích để xóa hoàn toàn vế phía sau (~~' AND released = 1~~). Điều này giúp loại bỏ điều kiện bắt buộc sản phẩm phải được phát hành.

## 4. Cách khắc phục 
1. Cách khắc phục cho Backend (Remediation)
`Lỗi của lập trình viên`: Đã dùng phương pháp cộng chuỗi (nối chuỗi) trực tiếp dữ liệu nhập từ Client vào câu lệnh SQL.

`Giải pháp`: Sử dụng Parameterized Queries (Truy vấn có tham số). Dữ liệu đầu vào sẽ được truyền tách biệt hoàn toàn với câu lệnh:

JavaScript
// Ví dụ minh họa an toàn trong Node.js
const query =`('SELECT * FROM products WHERE category = ? AND released = 1';)`

db.execute(query, [input_category]);

Khi dùng cách này, dù user có nhập ' OR 1=1--, Database vẫn hiểu toàn bộ cụm đó chỉ là một cái tên danh mục chữ thô và đi tìm danh mục mang tên đó, hoàn toàn không thể bẻ gãy logic câu lệnh.