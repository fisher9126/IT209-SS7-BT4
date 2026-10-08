# Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

> Thực hiện trên VPS Ubuntu Server (SSH, quyền root).

## 1. Mục tiêu
- Nginx Server Block làm Reverse Proxy.
- `/` phục vụ trang tĩnh trực tiếp từ `/var/www/html/`.
- `/api/` chuyển tiếp (proxy_pass) tới Spring Boot ở `http://127.0.0.1:8082/`.

## 2. Tạo trang tĩnh
```bash
mkdir -p /var/www/html
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="vi">
<head><meta charset="UTF-8"><title>Student Page</title></head>
<body>
  <h1>Trang chu tinh - Reverse Proxy Demo</h1>
  <p>Ho ten: Nguyen Van Huy</p>
  <p>Ma lop: PTIT - K24 - Devops</p>
</body>
</html>
EOF
```
> Nhớ thay Họ tên / Mã lớp cho đúng của bạn.

## 3. Tệp cấu hình `/etc/nginx/sites-available/spring-proxy.conf`
```nginx
server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8082/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Giải thích:
- `location /` + `try_files`: phục vụ file tĩnh, không có thì trả 404.
- `location /api/` + `proxy_pass http://127.0.0.1:8082/`: mọi request `/api/...` được chuyển tới backend cổng 8082. Dấu `/` cuối trong `proxy_pass` cắt bỏ tiền tố `/api/` khi chuyển tiếp (vd `/api/health` -> backend nhận `/health`).
- `proxy_set_header`: giữ thông tin client thật (Host, IP) để backend nhận diện đúng.

## 4. Kích hoạt cấu hình
```bash
ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx
```
- `ln -s`: tạo symlink sang `sites-enabled` để kích hoạt.
- `rm default`: gỡ server block mặc định (cũng listen cổng 80) để tránh xung đột.
- `nginx -t`: kiểm tra cú pháp -> phải thấy `syntax is ok` + `test is successful`.

## 5. Chạy backend giả lập ở cổng 8082 (để /api/ không bị 502)
```bash
nohup python3 -m http.server 8082 > /dev/null 2>&1 &
```
(Thực tế là app Spring Boot; ở đây dùng http.server mô phỏng cho có service lắng nghe cổng 8082.)

## 6. Kiểm tra
```bash
nginx -t
curl -I http://localhost/
curl -I http://localhost/api/
```

## 7. Kết quả thực tế (điền output thật của bạn)

### nginx -t
```
<dán - phai thay "syntax is ok" va "test is successful">
```

### curl -I http://localhost/
```
<dan - phai thay HTTP/1.1 200 OK (trang tinh)>
```

### curl -I http://localhost/api/
```
<dan - phai thay ma HTTP tu backend (vd 200), KHONG phai 502 Bad Gateway>
```

### curl http://localhost/ (noi dung trang tinh)
```
<dan - thay HTML chua Ho ten + Ma lop>
```
