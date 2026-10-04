# Bài tập 3: Cài đặt Nginx và thiết lập Server Block phục vụ trang HTML tĩnh

## Mục tiêu

- Cài đặt **Nginx** trên Ubuntu Server qua trình quản lý gói `apt`.
- Thiết lập **Server Block** tùy chỉnh lắng nghe cổng **80** để phục vụ trang HTML tĩnh cho dự án PTIT.

---

## Bước 1: Cài đặt Nginx

### 1.1. Đăng nhập vào Droplet

```bash
ssh -i ~/.ssh/id_ed25519 devops@<IP_ADDRESS_DROPLET>
```

### 1.2. Cập nhật danh sách gói

```bash
sudo apt update
```

### 1.3. Cài đặt Nginx

```bash
sudo apt install nginx -y
```

### 1.4. Kiểm tra phiên bản và trạng thái

```bash
nginx -v
```

Kết quả:

```
nginx version: nginx/1.18.0 (Ubuntu)
```

```bash
sudo systemctl status nginx
```

Kết quả:

```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2026-10-04 17:30:00 UTC
```

> ✅ Nginx đã **active (running)** và **enabled** (tự khởi động cùng hệ thống).

---

## Bước 2: Tạo thư mục mã nguồn và file HTML

### 2.1. Tạo thư mục chứa mã nguồn

```bash
sudo mkdir -p /var/www/ptit-web/html
```

### 2.2. Tạo file `index.html`

```bash
sudo nano /var/www/ptit-web/html/index.html
```

Dán nội dung sau (hoặc sử dụng file [index.html](index.html) đã chuẩn bị sẵn):

```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PTIT DevOps</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: #fff;
        }
        .container {
            text-align: center;
            padding: 40px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 16px;
            backdrop-filter: blur(10px);
            box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
        }
        h1 {
            font-size: 2.2rem;
            margin-bottom: 16px;
        }
        p {
            font-size: 1.1rem;
            opacity: 0.9;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Welcome to PTIT DevOps Course - Session 02</h1>
        <p>Nginx Server Block is running successfully! 🚀</p>
    </div>
</body>
</html>
```

### 2.3. Phân quyền thư mục

```bash
sudo chown -R www-data:www-data /var/www/ptit-web/
sudo chmod -R 755 /var/www/ptit-web/
```

**Giải thích:**

| Lệnh | Ý nghĩa |
|-------|---------|
| `chown -R www-data:www-data` | Đổi owner thành user `www-data` (user mặc định của Nginx) |
| `chmod -R 755` | Owner: đọc/ghi/thực thi; Group & Others: đọc/thực thi |

---

## Bước 3: Tạo file cấu hình Server Block

### 3.1. Tạo file cấu hình

```bash
sudo nano /etc/nginx/sites-available/ptit-web.conf
```

Dán nội dung sau (hoặc sử dụng file [ptit-web.conf](ptit-web.conf) đã chuẩn bị sẵn):

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/ptit-web/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    # Logging
    access_log /var/log/nginx/ptit-web.access.log;
    error_log  /var/log/nginx/ptit-web.error.log;
}
```

**Giải thích cấu hình:**

| Directive | Ý nghĩa |
|-----------|---------|
| `listen 80` | Lắng nghe kết nối HTTP trên cổng 80 (IPv4) |
| `listen [::]:80` | Lắng nghe kết nối HTTP trên cổng 80 (IPv6) |
| `server_name _` | Chấp nhận mọi tên miền / IP (catch-all) |
| `root /var/www/ptit-web/html` | Thư mục gốc chứa file tĩnh |
| `index index.html` | File mặc định khi truy cập `/` |
| `try_files $uri $uri/ =404` | Tìm file theo URI, nếu không có trả lỗi 404 |
| `access_log` / `error_log` | Đường dẫn file log riêng cho site |

---

## Bước 4: Kích hoạt Server Block

### 4.1. Tạo symlink sang `sites-enabled`

```bash
sudo ln -s /etc/nginx/sites-available/ptit-web.conf /etc/nginx/sites-enabled/
```

### 4.2. Xác nhận symlink đã được tạo

```bash
ls -la /etc/nginx/sites-enabled/
```

Kết quả:

```
lrwxrwxrwx 1 root root 43 Oct  4 17:45 ptit-web.conf -> /etc/nginx/sites-available/ptit-web.conf
lrwxrwxrwx 1 root root 34 Oct  4 17:00 default -> /etc/nginx/sites-available/default
```

---

## Bước 5: Xóa / Hủy kích hoạt cấu hình mặc định

### 5.1. Xóa symlink `default` để tránh xung đột cổng 80

```bash
sudo rm /etc/nginx/sites-enabled/default
```

### 5.2. Xác nhận chỉ còn cấu hình `ptit-web`

```bash
ls -la /etc/nginx/sites-enabled/
```

Kết quả:

```
lrwxrwxrwx 1 root root 43 Oct  4 17:45 ptit-web.conf -> /etc/nginx/sites-available/ptit-web.conf
```

> ✅ Chỉ còn duy nhất Server Block `ptit-web.conf` được kích hoạt.

> **Lưu ý:** File gốc `/etc/nginx/sites-available/default` vẫn còn nguyên, chỉ xóa symlink nên có thể khôi phục bất cứ lúc nào.

---

## Bước 6: Kiểm tra cú pháp và reload Nginx

### 6.1. Kiểm tra cú pháp cấu hình

```bash
sudo nginx -t
```

Kết quả mong đợi:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

> ✅ **syntax is ok** — cấu hình không có lỗi.

### 6.2. Reload Nginx để áp dụng cấu hình mới

```bash
sudo systemctl reload nginx
```

### 6.3. Xác nhận Nginx vẫn đang chạy

```bash
sudo systemctl status nginx
```

Kết quả:

```
● nginx.service - A high performance web server and a reverse proxy server
     Active: active (running)
```

---

## Bước 7: Kiểm tra từ trình duyệt

### 7.1. Truy cập từ máy cá nhân

Mở trình duyệt và truy cập:

```
http://<IP_ADDRESS_DROPLET>
```

Ví dụ: `http://128.199.123.456`

### 7.2. Kết quả mong đợi

Trình duyệt hiển thị trang HTML với dòng chữ:

> **Welcome to PTIT DevOps Course - Session 02**
>
> Nginx Server Block is running successfully! 🚀

### 7.3. Kiểm tra bằng `curl` (tùy chọn)

Từ Terminal máy cá nhân:

```bash
curl http://<IP_ADDRESS_DROPLET>
```

Hoặc từ trong Droplet:

```bash
curl http://localhost
```

Kết quả trả về nội dung HTML của trang `index.html`.

---

## Xử lý sự cố thường gặp

### ❌ Lỗi: Trình duyệt không thể truy cập

**Nguyên nhân:** Firewall chặn cổng 80.

**Giải pháp:**

```bash
# Kiểm tra trạng thái firewall
sudo ufw status

# Cho phép lưu lượng HTTP (cổng 80)
sudo ufw allow 'Nginx HTTP'

# Hoặc cho phép cả HTTP và HTTPS
sudo ufw allow 'Nginx Full'

# Kiểm tra lại
sudo ufw status
```

### ❌ Lỗi: `nginx -t` báo lỗi cú pháp

**Giải pháp:** Kiểm tra lại file cấu hình, đảm bảo:
- Mỗi directive kết thúc bằng dấu `;`
- Các block `{ }` đóng mở đúng cặp
- Đường dẫn `root` tồn tại trên server

### ❌ Lỗi: Hiển thị trang mặc định Nginx thay vì trang PTIT

**Giải pháp:**

```bash
# Đảm bảo đã xóa symlink default
sudo rm /etc/nginx/sites-enabled/default

# Kiểm tra và reload
sudo nginx -t
sudo systemctl reload nginx
```

---

## Cấu trúc thư mục trên Server

```
/etc/nginx/
├── nginx.conf                          # File cấu hình chính
├── sites-available/
│   ├── default                         # Cấu hình mặc định (giữ nguyên)
│   └── ptit-web.conf                   # ✅ Cấu hình site PTIT
└── sites-enabled/
    └── ptit-web.conf -> ../sites-available/ptit-web.conf  # ✅ Symlink

/var/www/ptit-web/
└── html/
    └── index.html                      # ✅ Trang HTML tĩnh

/var/log/nginx/
├── ptit-web.access.log                 # Log truy cập
└── ptit-web.error.log                  # Log lỗi
```

---

## Tóm tắt các lệnh

```bash
# 1. Cài đặt Nginx
sudo apt update
sudo apt install nginx -y

# 2. Tạo thư mục và file HTML
sudo mkdir -p /var/www/ptit-web/html
sudo nano /var/www/ptit-web/html/index.html
sudo chown -R www-data:www-data /var/www/ptit-web/
sudo chmod -R 755 /var/www/ptit-web/

# 3. Tạo Server Block
sudo nano /etc/nginx/sites-available/ptit-web.conf

# 4. Kích hoạt Server Block (symlink)
sudo ln -s /etc/nginx/sites-available/ptit-web.conf /etc/nginx/sites-enabled/

# 5. Xóa cấu hình default
sudo rm /etc/nginx/sites-enabled/default

# 6. Kiểm tra và reload
sudo nginx -t
sudo systemctl reload nginx

# 7. Mở firewall cho HTTP (nếu cần)
sudo ufw allow 'Nginx HTTP'
```

---

## Tài liệu tham khảo

- [Nginx - How to Install on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04)
- [Nginx - Server Block Documentation](https://nginx.org/en/docs/http/server_names.html)
- [Nginx - Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)
