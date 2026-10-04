# Bài tập 4: Cấu hình tường lửa bảo vệ máy chủ (UFW & Cloud Firewall Integration)

## Mục tiêu

- Thiết lập tường lửa **UFW** (Uncomplicated Firewall) trong Ubuntu để lọc gói tin mạng đi vào.
- Kết hợp **DigitalOcean Cloud Firewall** ở tầng hạ tầng để bảo vệ máy chủ từ biên mạng.
- Xây dựng mô hình **tường lửa 2 lớp** (Defense in Depth).

---

## Tổng quan kiến trúc tường lửa 2 lớp

```
                    Internet
                       │
                       ▼
        ┌──────────────────────────┐
        │  Lớp 2: DO Cloud Firewall│  ← Chặn tại biên mạng (trước khi
        │  (Tầng hạ tầng)          │     gói tin chạm đến Droplet)
        │  ✅ Allow: 22/tcp, 80/tcp │
        │  ❌ Deny: tất cả còn lại  │
        └────────────┬─────────────┘
                     │
                     ▼
        ┌──────────────────────────┐
        │  Lớp 1: UFW              │  ← Chặn tại OS (lớp phòng thủ
        │  (Tầng hệ điều hành)     │     cuối cùng bên trong Droplet)
        │  ✅ Allow: 22/tcp, 80/tcp │
        │  ❌ Default deny incoming  │
        └────────────┬─────────────┘
                     │
                     ▼
              ┌─────────────┐
              │   Droplet    │
              │  (Nginx,SSH) │
              └─────────────┘
```

**Tại sao cần 2 lớp?**

| Lớp | Ưu điểm | Hạn chế |
|-----|---------|---------|
| **Cloud Firewall** | Chặn ngay tại biên mạng, không tốn tài nguyên Droplet | Không kiểm soát được lưu lượng nội bộ giữa các Droplet cùng VPC |
| **UFW (OS)** | Kiểm soát chi tiết, hoạt động ngay cả khi Cloud Firewall bị gỡ | Gói tin đã tới Droplet rồi mới bị chặn, tốn tài nguyên hơn |

→ Kết hợp cả hai = **Defense in Depth** (phòng thủ theo chiều sâu).

---

## Lớp 1: Cấu hình UFW trong hệ điều hành

### Bước 1.1: Đăng nhập vào Droplet

```bash
ssh -i ~/.ssh/id_ed25519 devops@<IP_ADDRESS_DROPLET>
```

### Bước 1.2: Kiểm tra trạng thái UFW hiện tại

```bash
sudo ufw status
```

Kết quả (ban đầu):

```
Status: inactive
```

### Bước 1.3: Thiết lập chính sách mặc định

```bash
# Chặn tất cả lưu lượng đi VÀO (deny incoming)
sudo ufw default deny incoming

# Cho phép tất cả lưu lượng đi RA (allow outgoing)
sudo ufw default allow outgoing
```

Kết quả:

```
Default incoming policy changed to 'deny'
Default outgoing policy changed to 'allow'
```

**Giải thích:**

| Chính sách | Ý nghĩa |
|------------|---------|
| `deny incoming` | Mọi kết nối từ ngoài vào đều bị **chặn** trừ khi có rule cho phép |
| `allow outgoing` | Droplet vẫn có thể gửi request ra ngoài (apt update, curl, DNS...) |

### Bước 1.4: Mở cổng SSH (22/tcp)

```bash
sudo ufw allow 22/tcp
```

Kết quả:

```
Rules updated
Rules updated (v6)
```

> ⚠️ **QUAN TRỌNG:** Phải mở cổng SSH **TRƯỚC** khi bật UFW, nếu không bạn sẽ **mất kết nối** và bị khóa ngoài server!

### Bước 1.5: Mở cổng HTTP (80/tcp)

```bash
sudo ufw allow 80/tcp
```

Kết quả:

```
Rules updated
Rules updated (v6)
```

### Bước 1.6: Kích hoạt UFW

```bash
sudo ufw enable
```

Terminal sẽ cảnh báo:

```
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

→ Gõ **y** để xác nhận.

### Bước 1.7: Kiểm tra trạng thái UFW

```bash
sudo ufw status verbose
```

Kết quả mong đợi:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
```

> ✅ **Status: active** — UFW đã hoạt động.
> ✅ Chỉ có **2 cổng** được mở: `22/tcp` (SSH) và `80/tcp` (HTTP).

---

## Lớp 2: Cấu hình DigitalOcean Cloud Firewall

### Bước 2.1: Truy cập DigitalOcean Console

1. Đăng nhập [https://cloud.digitalocean.com](https://cloud.digitalocean.com).
2. Menu bên trái → **Networking** → **Firewalls**.
3. Nhấn **Create Firewall**.

### Bước 2.2: Đặt tên Cloud Firewall

- **Name:** `droplet-basic-firewall`

### Bước 2.3: Cấu hình Inbound Rules

Xóa các rule mặc định và thêm 2 rule sau:

| Type | Protocol | Port Range | Sources |
|------|----------|------------|---------|
| **SSH** | TCP | 22 | All IPv4, All IPv6 *(hoặc chỉ IP cá nhân của bạn)* |
| **HTTP** | TCP | 80 | All IPv4, All IPv6 |

> 💡 **Mẹo bảo mật nâng cao:** Thay vì cho phép SSH từ mọi nguồn, bạn có thể giới hạn chỉ từ IP cá nhân:
> - Kiểm tra IP cá nhân: truy cập [https://ifconfig.me](https://ifconfig.me)
> - Nhập IP đó vào ô Sources của rule SSH, ví dụ: `113.190.xxx.xxx/32`

### Bước 2.4: Cấu hình Outbound Rules

Giữ nguyên rule mặc định (cho phép tất cả lưu lượng đi ra):

| Type | Protocol | Port Range | Destinations |
|------|----------|------------|-------------|
| **All TCP** | TCP | All ports | All IPv4, All IPv6 |
| **All UDP** | UDP | All ports | All IPv4, All IPv6 |
| **ICMP** | ICMP | — | All IPv4, All IPv6 |

### Bước 2.5: Gán Droplet vào Cloud Firewall

1. Tại mục **Apply to Droplets**, tìm và chọn Droplet của bạn (ví dụ: `ubuntu-devops-test`).
2. Nhấn **Create Firewall**.

![Cloud Firewall đã tạo](images/cloud-firewall-created.png)

### Bước 2.6: Xác nhận Cloud Firewall đã áp dụng

Quay lại trang **Networking** → **Firewalls**, kiểm tra:

- Firewall `droplet-basic-firewall` hiển thị trạng thái **Active**.
- Droplet đã được gán (cột **Droplets** hiển thị số **1**).

![Cloud Firewall Inbound Rules](images/cloud-firewall-rules.png)

---

## Kiểm tra kết quả

### Test 1: Kết nối SSH (cổng 22) — phải thành công ✅

Từ Terminal máy cá nhân:

```bash
ssh -i ~/.ssh/id_ed25519 devops@<IP_ADDRESS_DROPLET>
```

Kết quả: Đăng nhập thành công.

```
devops@ubuntu-devops-test:~$
```

### Test 2: Truy cập HTTP (cổng 80) — phải thành công ✅

Từ trình duyệt máy cá nhân:

```
http://<IP_ADDRESS_DROPLET>
```

Hoặc dùng `curl`:

```bash
curl -I http://<IP_ADDRESS_DROPLET>
```

Kết quả:

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Content-Type: text/html
...
```

### Test 3: Quét cổng khác (ví dụ: 443, 3306, 8080) — phải bị chặn ❌

Từ máy cá nhân, thử kết nối đến cổng không được mở:

```bash
# Thử cổng 443 (HTTPS) - chưa mở
curl -m 5 http://<IP_ADDRESS_DROPLET>:443
```

Kết quả:

```
curl: (28) Connection timed out after 5000 milliseconds
```

```bash
# Thử cổng 3306 (MySQL) - chưa mở
nc -zv -w 5 <IP_ADDRESS_DROPLET> 3306
```

Kết quả:

```
Connection to <IP_ADDRESS_DROPLET> 3306 port [tcp/mysql] timed out!
```

> ✅ Các cổng không nằm trong danh sách cho phép đều bị **chặn/timeout**.

### Test 4: Kiểm tra trạng thái UFW trên server

```bash
sudo ufw status verbose
```

Kết quả:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
```

### Test 5: (Tùy chọn) Kiểm tra bằng `nmap` từ máy cá nhân

```bash
nmap -Pn <IP_ADDRESS_DROPLET>
```

Kết quả mong đợi:

```
PORT   STATE    SERVICE
22/tcp open     ssh
80/tcp open     http
```

> Các cổng khác hiển thị `filtered` hoặc không xuất hiện.

---

## Các lệnh quản lý UFW hữu ích

```bash
# Xem trạng thái chi tiết
sudo ufw status verbose

# Xem danh sách rules có đánh số
sudo ufw status numbered

# Xóa một rule theo số thứ tự
sudo ufw delete <number>

# Xóa rule theo nội dung
sudo ufw delete allow 80/tcp

# Cho phép cổng cụ thể từ một IP
sudo ufw allow from 113.190.xxx.xxx to any port 22

# Tắt UFW (tạm thời)
sudo ufw disable

# Reset toàn bộ rules về mặc định
sudo ufw reset

# Xem log tường lửa
sudo tail -f /var/log/ufw.log
```

---

## Tóm tắt cấu hình

### UFW (Lớp OS)

| Chính sách | Giá trị |
|------------|---------|
| Default Incoming | **DENY** |
| Default Outgoing | **ALLOW** |
| Port 22/tcp (SSH) | **ALLOW** |
| Port 80/tcp (HTTP) | **ALLOW** |
| Status | **Active** |

### DigitalOcean Cloud Firewall (Lớp hạ tầng)

| Direction | Type | Protocol | Port | Sources/Destinations |
|-----------|------|----------|------|---------------------|
| **Inbound** | SSH | TCP | 22 | All IPv4, All IPv6 |
| **Inbound** | HTTP | TCP | 80 | All IPv4, All IPv6 |
| **Outbound** | All TCP | TCP | All | All IPv4, All IPv6 |
| **Outbound** | All UDP | UDP | All | All IPv4, All IPv6 |
| **Outbound** | ICMP | ICMP | — | All IPv4, All IPv6 |

---

## Lưu ý bảo mật

- 🔒 **Luôn mở cổng SSH trước khi bật UFW** — tránh bị khóa ngoài server.
- 🔒 **Giới hạn nguồn SSH** theo IP cá nhân nếu có thể (thay vì `Anywhere`).
- 🔒 **Không mở cổng không cần thiết** — chỉ mở khi có dịch vụ thực sự lắng nghe.
- 🔒 **Kiểm tra định kỳ** bằng `sudo ufw status` và review Cloud Firewall rules.
- 🔒 **Ghi log** UFW để phát hiện các nỗ lực truy cập trái phép: `sudo ufw logging on`.

---

## Tài liệu tham khảo

- [DigitalOcean - How to Set Up a Firewall with UFW on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-22-04)
- [DigitalOcean - Cloud Firewalls Overview](https://docs.digitalocean.com/products/networking/firewalls/)
- [Ubuntu - UFW Documentation](https://help.ubuntu.com/community/UFW)
