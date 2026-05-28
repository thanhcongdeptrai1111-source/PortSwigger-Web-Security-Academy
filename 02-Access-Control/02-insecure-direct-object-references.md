# Lab: Insecure direct object references (IDOR)

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng IDOR trong tính năng tải lịch sử trò chuyện (`Live Chat transcript`).
- Đọc trộm lịch sử chat của người dùng khác để tìm kiếm thông tin nhạy cảm (Mật khẩu tài khoản).
- Đăng nhập vào tài khoản của người dùng `carlos` để hoàn thành bài Lab.

## 2. Cách khai thác (Exploitation Steps)
1. **Thu thập gói tin gốc:**
   - Tắt chặn gói tin (`Intercept is off`).
   - Truy cập tính năng **Live Chat**, gửi một tin nhắn bất kỳ để kích hoạt phiên trò chuyện.
   - Bấm nút **View transcript** để tải file lịch sử chat về máy.
2. **Chuyển sang Repeater:**
   - Vào tab **Proxy > HTTP history** trong Burp Suite.
   - Tìm gói tin có phương thức `POST` gửi tới URL `/download-transcript`.
   - Click chuột phải vào gói tin và chọn **Send to Repeater** (hoặc nhấn `Ctrl + R`).
3. **Khai thác lỗi IDOR (Tham chiếu đối tượng trực tiếp không an toàn):**
   - Chuyển sang tab **Repeater**, nhìn sang bảng **Inspector** ở phía bên phải.
   - Mở rộng mục **Request body parameters**, phát hiện tham số `filename` đang mang giá trị là tên file của mình (ví dụ: `2.txt`).
   - Kích đúp vào giá trị đó, sửa đổi (thay đổi ID) lùi lại thành `1.txt`.
   - Bấm nút **Send** (màu cam).
4. **Thu thập kết quả:**
   - Nhìn sang ô **Response** ở bên phải, chọn tab **Pretty** và cuộn xuống dưới cùng để đọc nội dung file chat của người dùng khác.
   - Phát hiện trong đoạn chat `1.txt`, hệ thống hỗ trợ tự động (hoặc hỗ trợ viên) đã cung cấp mật khẩu của người dùng `carlos`.
   - Sao chép mật khẩu này.
5. **Chiếm quyền điều khiển tài khoản:**
   - Quay lại trình duyệt, truy cập mục **My account**.
   - Đăng nhập bằng thông tin: `carlos` / `mật_khẩu_vừa_tìm_được`.
   - Bài Lab báo thành công (`Solved`).

## 3. Phân tích kỹ thuật (Technical Analysis)

### Sai lầm trong kiến trúc Backend:
- Lập trình viên lưu trữ các file bản ghi hội thoại của người dùng trên Server bằng cách đặt tên theo số thứ tự tăng dần (ví dụ: `1.txt`, `2.txt`, `3.txt`...).
- **Lỗ hổng chí mạng (IDOR):** Khi nhận yêu cầu tải file từ Client gửi lên thông qua tham số `filename=1.txt`, Backend chỉ kiểm tra xem file `1.txt` này có tồn tại trong thư mục lưu trữ hay không. Nếu có, Server lập tức gửi file về cho Client mà **hoàn toàn không kiểm tra xem người dùng đang thực hiện Request có phải là chủ sở hữu hợp pháp của đoạn chat đó hay không**.



### Cơ chế khai thác:
Do ID (tên file) có tính quy luật và dễ đoán, Hacker chỉ cần thay đổi giá trị của tham số từ xa (`2.txt` $\rightarrow$ `1.txt`) để buộc Server trả về tài liệu của nạn nhân mà không cần vượt qua bất kỳ cơ chế xác thực phức tạp nào.

## 4. Cách khắc phục cho Backend (Remediation)
Là một lập trình viên Backend, để triệt tiêu hoàn toàn lỗi IDOR, cần áp dụng hai giải pháp sau:

1. **Kiểm tra quyền sở hữu (Access Control Check):**
   - Tuyệt đối không bao giờ tin tưởng ID do người dùng gửi lên. Khi nhận được yêu cầu truy cập đối tượng (file `1.txt`), Backend phải thực hiện một câu lệnh kiểm tra chéo trong Database:
   ```javascript
   // Ví dụ minh họa logic an toàn trong Node.js
   const currentUserId = req.session.userId; // Lấy ID của người đang đăng nhập từ Session an toàn
   const fileId = req.body.filename; // ID file do người dùng gửi lên

   // Kiểm tra xem fileId này có thuộc quyền sở hữu của currentUserId hay không
   const isOwner = await db.checkFileOwnership(fileId, currentUserId);

   if (!isOwner) {
       return res.status(403).send("Quyền truy cập bị từ chối!");
   }
   // Nếu đúng là chủ sở hữu thì mới cho phép tải file
Sử dụng ID ngẫu nhiên, khó đoán (GUID/UUID):

Thay vì đặt tên file theo số thứ tự tăng dần dễ đoán (1.txt, 2.txt), hãy sử dụng chuỗi định danh ngẫu nhiên mã hóa có độ dài lớn (UUID v4).

Ví dụ: Thay vì 1.txt, file sẽ có tên là d3b07384-d113-4956-a5db-20a500af.txt. Việc này khiến Hacker không thể dùng phương pháp dò số tuần tự để tìm kiếm file của người khác.