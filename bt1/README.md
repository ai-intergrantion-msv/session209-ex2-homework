# Bài tập 1: Tạo Droplet trên DigitalOcean với xác thực SSH

## Mục tiêu

- Đăng ký tài khoản và khởi tạo Droplet chạy **Ubuntu Server** trên **DigitalOcean**.
- Cấu hình xác thực an toàn bằng **SSH Keypair** và kết nối thành công từ máy cá nhân.

---

## Bước 1: Đăng ký tài khoản DigitalOcean

1. Truy cập [https://www.digitalocean.com](https://www.digitalocean.com).
2. Nhấn **Sign Up** → Đăng ký bằng email hoặc tài khoản Google/GitHub.
3. Xác minh email và hoàn tất thông tin thanh toán (có thể dùng thẻ Visa/Mastercard hoặc PayPal).
4. DigitalOcean thường tặng **\$200 credit miễn phí trong 60 ngày** cho tài khoản mới.

---

## Bước 2: Tạo cặp khóa SSH trên máy cá nhân

### 2.1. Mở Terminal (hoặc PowerShell/Git Bash trên Windows)

Chạy lệnh sau để tạo cặp khóa SSH kiểu **Ed25519**:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### 2.2. Quá trình tạo khóa

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (C:\Users\<username>/.ssh/id_ed25519):  ← Nhấn Enter để dùng đường dẫn mặc định
Enter passphrase (empty for no passphrase):  ← Nhấn Enter (hoặc đặt mật khẩu bảo vệ)
Enter same passphrase again:  ← Nhấn Enter
```

### 2.3. Kết quả

Sau khi hoàn tất, hai file được tạo:

| File | Mô tả |
|------|--------|
| `~/.ssh/id_ed25519` | **Private Key** (Khóa bí mật) – **KHÔNG CHIA SẺ** |
| `~/.ssh/id_ed25519.pub` | **Public Key** (Khóa công khai) – Dùng để thêm vào server |

### 2.4. Xem nội dung Public Key

```bash
cat ~/.ssh/id_ed25519.pub
```

Kết quả có dạng:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your_email@example.com
```

> **Copy toàn bộ nội dung này** để sử dụng ở bước tiếp theo.

---

## Bước 3: Thêm SSH Public Key vào DigitalOcean

1. Đăng nhập DigitalOcean → Vào **Settings** → **Security** → **SSH Keys**.
2. Nhấn **Add SSH Key**.
3. Dán toàn bộ nội dung **Public Key** (đã copy ở bước 2.4) vào ô **SSH key content**.
4. Đặt tên cho khóa tại ô **Name** (ví dụ: `my-laptop-key`).
5. Nhấn **Add SSH Key** để lưu.

![Thêm SSH Key vào DigitalOcean](images/add-ssh-key.png)

---

## Bước 4: Tạo Droplet

### 4.1. Bắt đầu tạo Droplet

1. Vào **Dashboard** → Nhấn nút **Create** → Chọn **Droplets**.

### 4.2. Chọn Region (Vùng)

- Chọn **Singapore (SGP1)** — datacenter gần Việt Nam nhất, đảm bảo độ trễ thấp.

### 4.3. Chọn hệ điều hành (Image)

- Chọn tab **OS** → **Ubuntu** → **22.04 (LTS) x64** (hoặc bản LTS mới nhất).

### 4.4. Chọn cấu hình (Size)

| Thông số | Giá trị |
|----------|---------|
| **Droplet Type** | Basic (Shared CPU) |
| **CPU options** | Regular SSD |
| **Plan** | \$6/tháng (1 vCPU, 1 GB RAM, 25 GB SSD, 1 TB Transfer) |

> **Lưu ý:** Gói **\$4/tháng** (512 MB RAM) có thể không đủ cho một số ứng dụng web. Gói **\$6/tháng** là lựa chọn tối giản phù hợp nhất.

### 4.5. Chọn phương thức xác thực (Authentication)

- ✅ Chọn **SSH Keys**.
- Tick chọn SSH Key đã thêm ở **Bước 3** (ví dụ: `my-laptop-key`).

> ⚠️ **KHÔNG** chọn "Password" — xác thực bằng SSH Key an toàn hơn nhiều.

### 4.6. Tùy chọn bổ sung

| Mục | Giá trị |
|-----|---------|
| **Hostname** | `ubuntu-devops-test` (hoặc tên tùy ý) |
| **Project** | Chọn project hiện có hoặc tạo mới |
| **Backups** | Không bắt buộc (bỏ chọn để tiết kiệm chi phí) |

### 4.7. Tạo Droplet

- Nhấn **Create Droplet** và đợi 30–60 giây để Droplet được khởi tạo.
- Sau khi tạo xong, **ghi nhận địa chỉ IP** của Droplet (ví dụ: `128.199.xxx.xxx`).

![Droplet đã tạo thành công](images/droplet-created.png)

---

## Bước 5: Kết nối SSH tới Droplet

### 5.1. Mở Terminal và chạy lệnh SSH

```bash
ssh -i ~/.ssh/id_ed25519 root@<IP_ADDRESS_DROPLET>
```

Thay `<IP_ADDRESS_DROPLET>` bằng địa chỉ IP thực tế, ví dụ:

```bash
ssh -i ~/.ssh/id_ed25519 root@128.199.123.456
```

### 5.2. Xác nhận kết nối lần đầu

Lần đầu kết nối, Terminal sẽ hỏi:

```
The authenticity of host '128.199.xxx.xxx' can't be established.
ED25519 key fingerprint is SHA256:xxxxx...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

→ Gõ **yes** rồi nhấn **Enter**.

### 5.3. Kết quả mong đợi

Kết nối thành công **không cần nhập mật khẩu**, Terminal hiển thị thông tin chào mừng:

```
Welcome to Ubuntu 22.04.x LTS (GNU/Linux 5.15.0-xx-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Oct  4 16:55:00 UTC 2026

  System load:  0.0                Processes:             100
  Usage of /:   5.5% of 24.06GB   Users logged in:       0
  Memory usage: 20%                IPv4 address for eth0: 128.199.xxx.xxx
  Swap usage:   0%

root@ubuntu-devops-test:~#
```

![Kết nối SSH thành công](images/ssh-connected.png)

---

## Bước 6: Kiểm tra thông tin hệ thống (Tùy chọn)

Sau khi đăng nhập thành công, chạy các lệnh sau để kiểm tra:

```bash
# Kiểm tra phiên bản Ubuntu
lsb_release -a

# Kiểm tra hostname
hostname

# Kiểm tra địa chỉ IP
ip addr show eth0

# Kiểm tra tài nguyên hệ thống
free -h        # RAM
df -h          # Dung lượng ổ đĩa
nproc          # Số CPU
```

---

## Tóm tắt cấu hình Droplet

| Thông số | Giá trị |
|----------|---------|
| **Hệ điều hành** | Ubuntu 22.04 LTS x64 |
| **Plan** | Basic – Shared CPU |
| **CPU/RAM** | 1 vCPU / 1 GB RAM |
| **Disk** | 25 GB SSD |
| **Region** | Singapore (SGP1) |
| **Authentication** | SSH Key (Ed25519) |
| **Chi phí** | ~\$6/tháng |

---

## Lưu ý bảo mật

- 🔒 **Không bao giờ chia sẻ Private Key** (`id_ed25519`).
- 🔒 Nên tắt đăng nhập bằng mật khẩu trên server (`PasswordAuthentication no` trong `/etc/ssh/sshd_config`).
- 🔒 Cân nhắc tạo user riêng thay vì dùng `root` trực tiếp cho môi trường production.
- 🔒 Cập nhật hệ thống thường xuyên: `apt update && apt upgrade -y`.

---

## Tài liệu tham khảo

- [DigitalOcean - How to Create a Droplet](https://docs.digitalocean.com/products/droplets/how-to/create/)
- [DigitalOcean - How to Add SSH Keys](https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/)
- [SSH Keygen Manual](https://man.openbsd.org/ssh-keygen)
