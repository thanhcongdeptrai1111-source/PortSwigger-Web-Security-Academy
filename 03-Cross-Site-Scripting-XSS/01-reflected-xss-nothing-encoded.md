# Lab: User role controlled by request parameter

## 1. Mục tiêu (Objective)
- Khai thác lỗ hổng kiểm soát truy cập (Access Control) dựa trên tham số gói tin.
- Thay đổi vai trò người dùng (Role) từ tài khoản thường `wiener` lên quyền quản trị viên (`Admin`) để xóa người dùng `carlos`.

## 2. Cách khai thác (Exploitation Steps)
1. Truy cập trang web, chọn **My account** và đăng nhập bằng tài khoản thường: `wiener:peter`.
2. Mở Burp Suite, bật `Intercept is on`, sau đó bấm tải lại trang hoặc bấm vào mục quản lý tài khoản để bắt gói tin.
3. Phân tích gói tin `GET /my-account` hoặc Cookie đi kèm, phát hiện lập trình viên định nghĩa một tham số phân quyền rất ngây thơ:
   ```text
   Admin=false
Chỉnh sửa thủ công giá trị này trong Burp Suite từ false thành true:

Plaintext
Admin=true
Bấm Forward để gửi gói tin đi.

Kết quả: Trên giao diện trình duyệt xuất hiện thêm tính năng ẩn Admin panel. Truy cập vào đây và bấm Delete bên cạnh người dùng carlos để hoàn thành bài Lab (Solved).

💡 Mẹo nâng cao (Tự động hóa): Để không phải sửa tay từng gói tin bằng Intercept, có thể vào Settings > Match and Replace của Burp Suite. Thêm một quy tắc:

Type: Request header (hoặc body tùy nơi chứa tham số).

Match: Admin=false

Replace: Admin=true
Burp Suite sẽ chạy ngầm và tự động tráo đổi dữ liệu giúp bạn liên tục.

## 3. Phân tích kỹ thuật (Technical Analysis)
Sai lầm trong thiết kế Logic (Broken Access Control):
Lập trình viên Backend đã phạm một sai lầm chí mạng: Tin tưởng hoàn toàn vào dữ liệu do phía Client khai báo và gửi lên.

Thay vì tự quản lý và kiểm tra quyền hạn của người dùng ở môi trường an toàn bên trong Server, hệ thống lại đẩy tham số quyết định quyền sinh sát (Admin=false/true) về cho trình duyệt của người dùng giữ hộ.

Cơ chế khai thác:
Khi Hacker dùng Burp Suite sửa giá trị thành Admin=true, câu lệnh kiểm tra ở Backend đọc được chuỗi này liền lầm tưởng: "À, gói tin gửi lên bảo đây là Admin, vậy cho phép họ vào vùng quản trị". Hệ thống mở toang cửa mà không hề đối chiếu lại danh tính thật của user wiener trong Cơ sở dữ liệu.

## 4. Cách khắc phục cho Backend (Remediation)
Nguyên tắc vàng: Tuyệt đối không bao giờ cho phép Client tự quyết định hoặc thay đổi quyền hạn của mình thông qua các tham số HTTP Request.

Giải pháp: Quản lý quyền hạn tập trung ở Server thông qua cơ chế Session (Phiên làm việc):

Khi user đăng nhập thành công, Server chỉ cấp cho Client một chuỗi định danh ngẫu nhiên, vô hại gọi là Session ID (ví dụ: session=xyz123).

Toàn bộ thông tin quan trọng như username: wiener, role: user sẽ được lưu trữ an toàn trong bộ nhớ Server (như Redis) hoặc Database, gắn liền với ID xyz123 đó.

Mỗi khi người dùng thực hiện hành động (như vào trang Admin, xóa tài khoản), Backend sẽ cầm cái session=xyz123 đi tra cứu trong Database nội bộ của mình xem tài khoản này có quyền Admin thực sự không. Hacker dù có sửa gói tin ở Client thế nào cũng hoàn toàn vô tác dụng.