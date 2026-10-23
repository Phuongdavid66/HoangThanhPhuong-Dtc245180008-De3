# Đề Tài 3: Triển khai Website Quảng bá Sản phẩm (WordPress)
**Môn học:** Triển khai và Quản trị Hệ thống Phần mềm  
**Họ và tên sinh viên:** Hoàng Thanh Phương  

---

## 📌 Tổng quan kiến trúc hệ thống (Commit 1)
Hệ thống được đóng gói và vận hành hoàn toàn bằng **Docker Compose**, bao gồm các dịch vụ cốt lõi:
1. **Nginx Reverse Proxy**: Điểm tiếp nhận request duy nhất từ bên ngoài qua cổng 80 (HTTP) và 443 (HTTPS), tự động chuyển hướng HTTP sang HTTPS, cấu hình các HTTP Security Headers (`X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`, `Strict-Transport-Security`).
2. **WordPress Web Application**: Website giới thiệu sản phẩm chạy trên môi trường PHP-Apache chính thức.
3. **MySQL Database (v8.0)**: Hệ quản trị cơ sở dữ liệu lưu trữ toàn bộ bài viết, cấu hình của WordPress.
4. **phpMyAdmin**: Giao diện trực quan quản lý cơ sở dữ liệu qua cổng 8081.

### 🛡️ Biện pháp bảo mật & Hardening (Commit 1)
- **Network Isolation**: Tách biệt thành 2 mạng bridge riêng biệt:
  - `frontend_net`: Chỉ cho phép Nginx và WordPress giao tiếp.
  - `backend_net`: Chỉ cho phép WordPress, phpMyAdmin và MySQL giao tiếp. Cơ sở dữ liệu MySQL không public cổng 3306 ra ngoài máy chủ.
- **Mật khẩu an toàn**: Lưu trữ qua biến môi trường `.env`, sử dụng mật khẩu độ phức tạp cao.
- **SSL/TLS Encryption**: Mã hóa toàn bộ dữ liệu truyền nhận bằng chứng chỉ SSL tự ký.

---

## 🚀 Hướng dẫn triển khai và vận hành

### 1. Chuẩn bị môi trường
- Máy tính đã cài đặt **Docker Desktop** và **Docker Compose**.
- Sao chép file cấu hình môi trường:
```bash
cp .env.example .env
```
*(Nếu cần, có thể chỉnh sửa mật khẩu trong file `.env`)*

### 2. Khởi chạy hệ thống
Chạy lệnh sau tại thư mục gốc của dự án:
```bash
docker compose up -d
```

### 3. Kiểm tra trạng thái các container
```bash
docker compose ps
```
Đảm bảo cả 4 container (`wp_nginx`, `wp_app`, `wp_phpmyadmin`, `wp_mysql`) đều ở trạng thái `running / healthy`.

---

## 🌐 Đường dẫn truy cập các dịch vụ

| Dịch vụ | Giao thức & Cổng | URL truy cập | Ghi chú |
| :--- | :--- | :--- | :--- |
| **Website WordPress** | HTTPS (443) | `https://localhost` | Có chuyển hướng từ `http://localhost` |
| **phpMyAdmin** | HTTP (8081) | `http://localhost:8081` | Đăng nhập bằng tài khoản trong `.env` |
| **MySQL DB** | Nội bộ (3306) | `db:3306` | Được cô lập trong mạng `backend_net` |

---

## 📋 Lịch sử tiến độ (Commit Log)
- `commit 1`: Triển khai ứng dụng WordPress, MySQL, phpMyAdmin và Nginx Reverse Proxy (HTTPS + Security Headers).
- `commit 2`: (Dự kiến) Tích hợp hệ thống giám sát Prometheus và Grafana.
- `commit 3`: (Dự kiến) Tích hợp hệ thống ghi log tập trung Loki + Promtail và hoàn thiện Hardening.
