# BÁO CÁO THỰC HÀNH TRIỂN KHAI VÀ QUẢN TRỊ HỆ THỐNG PHẦN MỀM
## ĐỀ TÀI 3: WEBSITE QUẢNG BÁ SẢN PHẨM (WORDPRESS)
* **Họ và tên sinh viên:** Hoàng Thanh Phương  
* **Mã số sinh viên (MSV):** Dtc245180008  
* **Repository GitHub:** [https://github.com/Phuongdavid66/HoangThanhPhuong-Dtc245180008-De3](https://github.com/Phuongdavid66/HoangThanhPhuong-Dtc245180008-De3)  

---

## 🏛️ 1. Sơ đồ Kiến trúc Tổng thể Hệ thống

Hệ thống được đóng gói và vận hành hoàn toàn tự động bằng **Docker Compose**, bao gồm **11 dịch vụ (containers)** được phân vùng bảo mật rõ ràng:

```
                            [ Khách hàng / Người dùng ]
                                         │
                             Port 80/443 (HTTP / HTTPS)
                                         ▼
                 ┌─────────────────────────────────────────────────┐
                 │       wp_nginx (Nginx Reverse Proxy)            │
                 │   - Chuyển hướng HTTP sang HTTPS                │
                 │   - SSL/TLS Tự ký & Security Headers            │
                 │   - Endpoint /stub_status (Port 8080 nội bộ)    │
                 └───────────────────────┬─────────────────────────┘
                                         │ (frontend_net)
                                         ▼
                           ┌───────────────────────────┐
                           │   wp_app (WordPress)      │
                           └─────────────┬─────────────┘
                                         │ (backend_net)
                                         ▼
   ┌──────────────────────┐    ┌───────────────────────────┐
   │ wp_phpmyadmin (:8081)│───>│   wp_mysql (MySQL 8.0)    │ (Cô lập trong backend_net,
   └──────────────────────┘    └─────────────┬─────────────┘  không public port 3306)
                                             │
      ═══════════════════════════════════════╪═══════════════════════════════════════════
                      PHÂN HỆ GIÁM SÁT & QUẢN TRỊ LOG (monitoring_net)
      ───────────────────────────────────────────────────────────────────────────────────
         [ Các Exporter thu thập Metrics ]
         ├── wp_cadvisor          : Cào metrics CPU, RAM, IO toàn bộ container
         ├── wp_nginx_exporter    : Cào metrics kết nối Nginx từ /stub_status
         └── wp_mysqld_exporter   : Cào metrics hoạt động của MySQL qua .my.cnf
                     │
                     ▼
         ┌─────────────────────────┐         ┌─────────────────────────┐
         │ wp_prometheus (:9090)   │────────>│   wp_grafana (:3000)    │
         │ (Kho lưu trữ Metrics)   │         │ (Dashboard & Tra cứu)   │
         └─────────────────────────┘         └────────────▲────────────┘
                                                          │
         [ Hệ thống Log tập trung ]                       │ (Truy vấn LogQL)
         ┌─────────────────────────┐         ┌────────────┴────────────┐
         │ wp_promtail             │────────>│   wp_loki (:3100)       │
         │ (Quét file log Docker)  │         │ (Kho lưu trữ Log)       │
         └─────────────────────────┘         └─────────────────────────┘
```

---

## 📦 2. Danh mục 11 Dịch vụ Container

| STT | Container Name | Docker Image | Cổng Public | Chức năng chính |
| :---: | :--- | :--- | :---: | :--- |
| **1** | `wp_nginx` | `nginx:alpine` | `80`, `443` | Reverse proxy, điều hướng HTTPS, bảo mật headers |
| **2** | `wp_app` | `wordpress:latest` | *Nội bộ* | Website giới thiệu sản phẩm WordPress |
| **3** | `wp_mysql` | `mysql:8.0` | *Nội bộ* | Cơ sở dữ liệu MySQL lưu trữ dữ liệu WordPress |
| **4** | `wp_phpmyadmin` | `phpmyadmin:latest` | `8081` | Giao diện quản trị cơ sở dữ liệu MySQL qua trình duyệt |
| **5** | `wp_cadvisor` | `cadvisor:v0.47.2` | *Nội bộ* | Thu thập metrics CPU, RAM, Network I/O của container |
| **6** | `wp_nginx_exporter`| `nginx-prometheus-exporter:1.1.0` | *Nội bộ* | Chuyển đổi trạng thái Nginx thành Prometheus metrics |
| **7** | `wp_mysqld_exporter`| `mysqld-exporter:latest` | *Nội bộ* | Thu thập metrics queries, connections của MySQL |
| **8** | `wp_prometheus` | `prometheus:v2.51.0`| `9090` | Máy chủ thu thập và lưu trữ số liệu (Time-Series DB) |
| **9** | `wp_grafana` | `grafana:10.4.1` | `3000` | Trực quan hóa Dashboard số liệu và phân tích Log |
| **10**| `wp_loki` | `loki:2.9.8` | `3100` | Hệ thống lưu trữ và đánh chỉ mục Log tập trung |
| **11**| `wp_promtail` | `promtail:2.9.8` | *Nội bộ* | Tự động thu thập log từ Docker daemon và đẩy vào Loki |

---

## 🛡️ 3. Các Biện Pháp Hardening & An Toàn Hệ Thống (Yêu cầu 6)

Hệ thống đã áp dụng đầy đủ **5 biện pháp bảo mật chuyên sâu (Hardening)**:

1. **Network Isolation (Cô lập mạng nội bộ 3 tầng):**
   - `frontend_net`: Chỉ bao gồm `wp_nginx`, `wp_app` và `wp_nginx_exporter`.
   - `backend_net`: Chỉ chứa `wp_app`, `wp_phpmyadmin`, `wp_mysql` và `wp_mysqld_exporter`.
   - `monitoring_net`: Mạng nội bộ riêng biệt dành cho việc giám sát (`prometheus`, `grafana`, `loki`, `promtail`, `cadvisor`).
   - **Đặc biệt:** Cổng Database `3306` **tuyệt đối không mở ra ngoài máy host**, ngăn chặn hoàn toàn nguy cơ brute-force hoặc quét cổng từ Internet.
2. **Hạn chế leo thang đặc quyền (`no-new-privileges:true`):**
   - Toàn bộ các dịch vụ trọng yếu đều được cấu hình cờ bảo mật `security_opt: ["no-new-privileges:true"]`. Cấu hình này ngăn chặn các tiến trình bên trong container sử dụng các bit `setuid`/`setgid` để chiếm quyền root của host.
3. **Chạy dưới quyền người dùng hạn chế (Non-root Containers):**
   - `wp_loki` chạy với UID/GID dành riêng `10001:10001`.
   - `wp_prometheus` chạy với user hạn chế `nobody`.
   - `wp_grafana` chạy với user nội bộ `grafana` (UID 472).
   - `wp_mysql` chạy tiến trình dịch vụ dưới user `mysql` (UID 999).
   - `wp_app` (Apache) chạy các worker xử lý dưới user `www-data` (UID 33).
4. **Bảo mật Nginx & HTTP Security Headers:**
   - Mã hóa bắt buộc bằng **SSL/TLS** (`TLSv1.2`, `TLSv1.3`).
   - Tự động chuyển hướng toàn bộ request HTTP (port 80) sang HTTPS (port 443).
   - Ẩn phiên bản máy chủ Nginx (`server_tokens off;`) để tránh bị rà quét lỗ hổng phiên bản.
   - Bổ sung 5 Security Headers chống các kiểu tấn công phổ biến:
     + `X-Frame-Options: SAMEORIGIN` (Chống Clickjacking).
     + `X-Content-Type-Options: nosniff` (Chống MIME Confusion / Sniffing).
     + `X-XSS-Protection: 1; mode=block` (Bảo vệ trình duyệt khỏi tấn công XSS phản xạ).
     + `Referrer-Policy: strict-origin-when-cross-origin` (Chống rò rỉ URL nhạy cảm).
     + `Strict-Transport-Security: max-age=31536000; includeSubDomains` (Bắt buộc duyệt HTTPS an toàn).
5. **Quản lý thông tin xác thực an toàn (Mật khẩu mạnh & .gitignore):**
   - Đặt mật khẩu có độ phức tạp cao cho Database và tài khoản Admin của Grafana.
   - Loại trừ hoàn toàn file `.env` và `mysqld-exporter/.my.cnf` khỏi Git thông qua `.gitignore`. Đẩy file mẫu `.env.example` và `.my.cnf.example` lên repository để đảm bảo an toàn bí mật mã nguồn.

---

## 🚀 4. Hướng dẫn Khởi chạy Hệ thống

### Bước 1: Sao chép các file cấu hình mẫu
```bash
# Sao chép biến môi trường cho ứng dụng và database
cp .env.example .env

# Sao chép thông tin xác thực cho exporter MySQL
cp mysqld-exporter/.my.cnf.example mysqld-exporter/.my.cnf
```

### Bước 2: Khởi động toàn bộ cụm container
```bash
docker compose up -d
```

### Bước 3: Kiểm tra trạng thái hoạt động
```bash
docker compose ps
```
*Tất cả 11 container phải hiển thị trạng thái `Up` (và `healthy` đối với MySQL, cAdvisor).*

---

## 🌐 5. Bảng Cổng và Đường dẫn Truy cập Dịch vụ

| Tên Dịch Vụ | URL Truy cập | Tài khoản / Mật khẩu | Mục đích kiểm tra / Demo |
| :--- | :--- | :--- | :--- |
| **Website WordPress** | `https://localhost` | Khởi tạo trang ban đầu | Giao diện web chính thức (HTTPS tự ký) |
| **phpMyAdmin** | `http://localhost:8081` | User: `root`<br>Pass: trong `.env` | Quản lý bảng dữ liệu MySQL |
| **Prometheus Targets** | `http://localhost:9090/targets` | Không yêu cầu | Xem trạng thái 4 targets (`UP 1/1`) |
| **Grafana Dashboard** | `http://localhost:3000` | User: `admin`<br>Pass: `Admin_S3cur3d#2026!` | Xem Dashboard giám sát và tra cứu LogQL |
| **Loki Readiness** | `http://localhost:3100/ready` | Không yêu cầu | Trả về `ready` |

---

## 🔍 6. Hướng dẫn Truy vấn Log tập trung bằng LogQL (Yêu cầu 5)

Hệ thống đã cấu hình tự động **Datasource Loki** vào Grafana. Để thực hiện truy vấn và chụp ảnh minh chứng:

1. Đăng nhập vào Grafana tại [http://localhost:3000](http://localhost:3000).
2. Vào thanh menu bên trái, chọn biểu tượng la bàn: **Explore**.
3. Tại ô chọn Datasource phía trên góc trái, chọn **Loki**.
4. Nhập các câu lệnh **LogQL** mẫu dưới đây vào ô truy vấn và bấm **Run query**:

### 🌟 Câu lệnh LogQL 1: Truy vấn toàn bộ log của Nginx Reverse Proxy
```logql
{container="wp_nginx"}
```
> **Ý nghĩa:** Trích xuất toàn bộ access log và error log phát sinh từ Nginx reverse proxy khi tiếp nhận các yêu cầu truy cập từ người dùng.

### 🌟 Câu lệnh LogQL 2: Lọc các request HTTP redirect hoặc thành công trên Nginx
```logql
{container="wp_nginx"} |= "301"
```
*(Hoặc `{container="wp_nginx"} |= "GET"`)*
> **Ý nghĩa:** Sử dụng bộ lọc chuỗi (line filter) để tìm các bản ghi chuyển hướng giao thức từ HTTP sang HTTPS (mã trạng thái HTTP 301).

### 🌟 Câu lệnh LogQL 3: Truy vấn nhật ký khởi động và kết nối của MySQL Database
```logql
{container="wp_mysql"} |= "ready for connections"
```
> **Ý nghĩa:** Tìm kiếm thời điểm MySQL khởi động hoàn tất các engine và sẵn sàng tiếp nhận kết nối từ WordPress và phpMyAdmin.

### 🌟 Câu lệnh LogQL 4: Truy vấn nhật ký xử lý ứng dụng WordPress
```logql
{container="wp_app"}
```
> **Ý nghĩa:** Theo dõi quá trình xử lý script PHP và các truy vấn nội bộ của WordPress.

---

## 📜 7. Lịch sử Commit của Dự án (Đủ 3 Commit theo đề bài)

- **`commit 1` (`6a089a8`):** Triển khai ứng dụng WordPress, MySQL, phpMyAdmin với Nginx Reverse Proxy và cấu hình HTTPS / Security Headers.
- **`commit 2` (`6c6e932`):** Tích hợp Prometheus và Grafana với cAdvisor, Nginx Prometheus Exporter và MySQL Exporter kèm Dashboard tự động.
- **`commit 3`:** Hoàn thành hệ thống ghi log tập trung Loki + Promtail, hỗ trợ truy vấn LogQL và gia cố bảo mật toàn diện (Hardening).
