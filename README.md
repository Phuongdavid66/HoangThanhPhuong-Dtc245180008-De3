# Đề Tài 3: Triển khai Website Quảng bá Sản phẩm (WordPress)
**Môn học:** Triển khai và Quản trị Hệ thống Phần mềm  
**Họ và tên sinh viên:** Hoàng Thanh Phương  

---

## 📌 Tổng quan kiến trúc hệ thống (Commit 1 & Commit 2)
Hệ thống được đóng gói và vận hành hoàn toàn bằng **Docker Compose**, bao gồm 9 dịch vụ phối hợp chặt chẽ:

### 1. Khu vực Dịch vụ Cốt lõi (Web & Database)
1. **Nginx Reverse Proxy**: Tiếp nhận request bên ngoài qua cổng 80 (HTTP) và 443 (HTTPS), tự động chuyển hướng HTTP -> HTTPS, cấu hình các HTTP Security Headers (`X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`, `Strict-Transport-Security`). Đồng thời mở endpoint nội bộ `/stub_status` để phục vụ giám sát.
2. **WordPress Web Application**: Website giới thiệu sản phẩm chạy trên môi trường PHP-Apache chính thức.
3. **MySQL Database (v8.0)**: Hệ quản trị cơ sở dữ liệu lưu trữ toàn bộ bài viết, dữ liệu WordPress.
4. **phpMyAdmin**: Giao diện trực quan quản lý cơ sở dữ liệu qua cổng 8081.

### 2. Khu vực Giám sát Hệ thống (Prometheus & Grafana)
5. **cAdvisor**: Thu thập các chỉ số tài nguyên của toàn bộ Docker Container (CPU Usage, RAM Usage, Network IO).
6. **Nginx Prometheus Exporter**: Thu thập metrics hoạt động của Nginx (Active Connections, Request Rate, Connection states).
7. **MySQL Exporter**: Thu thập metrics chuyên sâu của MySQL (Client Connections, Queries/sec, Network throughput).
8. **Prometheus Server**: Thu thập (scrape) định kỳ từ 4 nguồn (Prometheus, cAdvisor, Nginx-exporter, MySQL-exporter) qua cổng 9090.
9. **Grafana**: Hiển thị bảng điều khiển (Dashboard) trực quan hóa các số liệu thu thập qua cổng 3000, hỗ trợ auto-provisioning Datasource và Dashboard tổng quan.

---

## 🛡️ Biện pháp bảo mật & Hardening
- **Network Isolation**: Tách biệt thành 3 mạng bridge độc lập:
  - `frontend_net`: Cho Nginx, WordPress và Nginx-exporter.
  - `backend_net`: Cho WordPress, phpMyAdmin, MySQL và MySQL-exporter. Cổng MySQL 3306 hoàn toàn không public ra ngoài máy chủ.
  - `monitoring_net`: Mạng riêng dành cho Prometheus, Grafana, cAdvisor và các Exporter.
- **Mật khẩu an toàn**: Quản lý qua `.env` và file cấu hình nội bộ `.my.cnf` (đều được loại trừ khỏi Git qua `.gitignore`).
- **SSL/TLS Encryption**: Mã hóa toàn bộ dữ liệu truyền nhận giữa người dùng và Nginx bằng chứng chỉ SSL.
- **Grafana Hardening**: Sử dụng mật khẩu quản trị phức tạp, vô hiệu hóa tính năng tự đăng ký tài khoản (`GF_USERS_ALLOW_SIGN_UP=false`).

---

## 🚀 Hướng dẫn triển khai và vận hành

### 1. Chuẩn bị môi trường
- Máy tính đã cài đặt **Docker Desktop** và **Docker Compose**.
- Sao chép các file cấu hình môi trường:
```bash
cp .env.example .env
cp mysqld-exporter/.my.cnf.example mysqld-exporter/.my.cnf
```

### 2. Khởi chạy toàn bộ hệ thống
```bash
docker compose up -d
```

### 3. Kiểm tra trạng thái các container
```bash
docker compose ps
```
Đảm bảo tất cả 9 container đều ở trạng thái `running / healthy`.

---

## 🌐 Đường dẫn truy cập các dịch vụ

| Dịch vụ | Cổng Host | URL truy cập | Tài khoản mặc định | Ghi chú |
| :--- | :--- | :--- | :--- | :--- |
| **Website WordPress** | 80 & 443 | `https://localhost` | Khởi tạo ban đầu | Tự chuyển hướng từ `http://localhost` |
| **phpMyAdmin** | 8081 | `http://localhost:8081` | `root` / trong `.env` | Quản lý database trực tiếp |
| **Prometheus** | 9090 | `http://localhost:9090/targets` | Không | Kiểm tra trạng thái 4 targets (`UP`) |
| **Grafana Dashboard** | 3000 | `http://localhost:3000` | `admin` / `Admin_S3cur3d#2026!` | Có sẵn Dashboard "Tổng quan Giám sát Hệ thống" |

---

## 📋 Lịch sử tiến độ (Commit Log)
- `commit 1`: Triển khai ứng dụng WordPress, MySQL, phpMyAdmin và Nginx Reverse Proxy (HTTPS + Security Headers).
- `commit 2`: Tích hợp Prometheus + Grafana và 3 Exporters (cAdvisor, Nginx-exporter, MySQL-exporter) kèm Auto-provisioning Dashboard.
- `commit 3`: (Dự kiến) Tích hợp hệ thống ghi log tập trung Loki + Promtail và hoàn thiện Hardening.
