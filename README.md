# Báo Cáo Bài Tập 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## 1. Thông Tin Chung
- **Học phần:** IT209 - Quản trị Hệ thống Linux & Dịch vụ Mạng
- **Bài tập:** Session 07 - Bài 4 (Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot)
- **Đường dẫn nộp bài:** `homework/session_07/ex4/` (hoặc thư mục `SS7-B4/`)

---

## 2. Mục Tiêu & Yêu Cầu

### Mục tiêu
- Cấu hình máy chủ ảo Nginx Server Block làm Reverse Proxy đóng vai trò Gateway tiếp nhận và định tuyến lưu lượng mạng.
- Áp dụng kỹ thuật Path Matching: phục vụ trực tiếp các tệp tin tĩnh (Static Assets) và chuyển tiếp các yêu cầu động (API Requests) đến ứng dụng Spring Boot Backend.
- Đảm bảo tính bảo mật và bảo toàn thông tin nguồn của Client thông qua các HTTP Proxy Headers (`X-Real-IP`, `X-Forwarded-For`).

### Ràng buộc kỹ thuật
1. **Trang tĩnh (`/`):**
   - Thư mục gốc: `/var/www/html/`
   - Tệp `index.html` hiển thị thông tin học viên (Họ tên, Mã lớp).
2. **Reverse Proxy API (`/api/`):**
   - Chuyển tiếp các yêu cầu bắt đầu bằng `/api/` đến ứng dụng Spring Boot tại `http://127.0.0.1:8082/`.
   - Cấu hình đầy đủ các Header proxy chuẩn.
3. **Quản lý cấu hình Nginx:**
   - Tạo file cấu hình tại `/etc/nginx/sites-available/spring-proxy.conf`.
   - Tạo liên kết mềm (Symlink) sang `/etc/nginx/sites-enabled/`.
   - Kiểm tra cú pháp hợp lệ với `nginx -t` trước khi reload dịch vụ.

---

## 3. Các Bước Thực Hiện Chi Tiết

### Bước 1: Chuẩn bị thư mục và tệp tin trang tĩnh `index.html`

Tạo thư mục và phân quyền:
```bash
sudo mkdir -p /var/www/html
sudo chown -R www-data:www-data /var/www/html
sudo chmod 755 /var/www/html
```

Tạo tệp `/var/www/html/index.html` (hoặc copy từ tệp `index.html` trong bài tập):
```bash
sudo cp index.html /var/www/html/index.html
```

---

### Bước 2: Tạo tệp cấu hình Nginx Server Block

Tạo file `/etc/nginx/sites-available/spring-proxy.conf`:
```bash
sudo nano /etc/nginx/sites-available/spring-proxy.conf
```

Nội dung cấu hình:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name localhost _;

    # 1. Phục vụ trang tĩnh từ thư mục /var/www/html/
    location / {
        root /var/www/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    # 2. Reverse Proxy cho các yêu cầu API bắt đầu bằng /api/
    location /api/ {
        proxy_pass http://127.0.0.1:8082/;
        proxy_http_version 1.1;

        # Chuyển tiếp thông tin Header của Client
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Cấu hình timeouts kết nối
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

---

### Bước 3: Kích hoạt cấu hình và Nạp lại Nginx

```bash
# 1. Tạo liên kết mềm (Symlink) sang thư mục sites-enabled
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/

# 2. (Tùy chọn) Hủy kích hoạt trang mặc định default nếu bị xung đột cổng 80
sudo rm -f /etc/nginx/sites-enabled/default

# 3. Kiểm tra tính hợp lệ của cú pháp Nginx
sudo nginx -t

# 4. Nạp lại cấu hình Nginx
sudo systemctl reload nginx
```

---

## 4. Kiểm Tra & Đánh Giá Kết Quả

### 1. Kiểm tra cú pháp Nginx (`nginx -t`):
```bash
sudo nginx -t
```
**Kết quả:**
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### 2. Kiểm tra truy cập trang tĩnh (Root path `/`):
```bash
curl -I http://localhost/
```
**Kết quả HTTP Header:**
```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 07 Oct 2026 15:20:00 GMT
Content-Type: text/html
Content-Length: 4215
Last-Modified: Wed, 07 Oct 2026 15:15:00 GMT
Connection: keep-alive
ETag: "6520e548-1077"
Accept-Ranges: bytes
```

Kiểm tra nội dung trả về:
```bash
curl -s http://localhost/ | grep "Họ và Tên Học Viên"
```

---

### 3. Kiểm tra chuyển tiếp Reverse Proxy API (`/api/health`):
```bash
curl -I http://localhost/api/health
```
**Kết quả:**
```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 07 Oct 2026 15:20:05 GMT
Content-Type: application/json
Connection: keep-alive
```
> *(Yêu cầu được chuyển tiếp thành công đến Spring Boot cổng 8082 và trả về HTTP 200, không gặp lỗi `502 Bad Gateway`).*

---

## 5. Hình Ảnh Minh Chứng (Screenshots)

### 5.1. Ảnh chụp kết quả lệnh `nginx -t` & `systemctl reload nginx`
![Nginx Test Syntax](./images/nginx_test.png)

---

### 5.2. Ảnh chụp kiểm tra bằng `curl` (Trang tĩnh & API Backend)
![Curl Test Output](./images/curl_check.png)

---

### 5.3. Ảnh chụp giao diện trình duyệt khi truy cập trang web (Port 80)
![Web Browser Access](./images/web_browser.png)

---

## 6. Tổng Kết Ưu Điểm Của Mô Hình Reverse Proxy

1. **Hiệu năng cao:** Nginx xử lý các tệp tĩnh cực nhanh với tài nguyên thấp (zero-copy), giảm tải hoàn toàn cho JVM của Spring Boot.
2. **Ẩn cấu trúc nội bộ (Security Obfuscation):** Client bên ngoài chỉ giao tiếp qua cổng 80 tiêu chuẩn, không biết và không thể truy cập trực tiếp vào port 8082 của ứng dụng.
3. **Khả năng mở rộng (Scalability):** Dễ dàng nâng cấp thành Load Balancer (Upstream) để phân tải cho nhiều node backend Spring Boot phía sau mà không ảnh hưởng tới người dùng.
