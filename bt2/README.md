# Bài tập 2: Khởi tạo User thường và thiết lập đặc quyền quản trị (Sudoers Configuration)

## Mục tiêu

- Tạo tài khoản người dùng thường (**non-root**) trên Ubuntu Server để hạn chế rủi ro khi thao tác trực tiếp bằng quyền `root`.
- Cấp đặc quyền quản trị an toàn qua nhóm `sudo` và sao chép cấu hình SSH Key để đăng nhập.
- Tuân thủ nguyên tắc **đặc quyền tối thiểu (Least Privilege)**.

---

## Bước 1: Đăng nhập vào Droplet bằng tài khoản root

Từ Terminal máy cá nhân, SSH vào Droplet:

```bash
ssh -i ~/.ssh/id_ed25519 root@<IP_ADDRESS_DROPLET>
```

Kết quả:

```
root@ubuntu-devops-test:~#
```

---

## Bước 2: Tạo tài khoản người dùng mới `devops`

### 2.1. Tạo user

```bash
adduser devops
```

Terminal sẽ yêu cầu nhập thông tin:

```
Adding user `devops' ...
Adding new group `devops' (1001) ...
Adding new user `devops' (1001) with group `devops' ...
Creating home directory `/home/devops' ...
Copying files from `/etc/skel' ...

New password:            ← Nhập mật khẩu cho user devops
Retype new password:     ← Nhập lại mật khẩu

Changing the user information for devops
Enter the new value, or press ENTER for the default
    Full Name []: DevOps User
    Room Number []:       ← Enter để bỏ qua
    Work Phone []:        ← Enter để bỏ qua
    Home Phone []:        ← Enter để bỏ qua
    Other []:             ← Enter để bỏ qua
Is the information correct? [Y/n] Y
```

### 2.2. Xác nhận user đã được tạo

```bash
id devops
```

Kết quả:

```
uid=1001(devops) gid=1001(devops) groups=1001(devops)
```

---

## Bước 3: Thêm user `devops` vào nhóm `sudo`

### 3.1. Thêm vào nhóm sudo

```bash
usermod -aG sudo devops
```

### 3.2. Xác nhận user đã thuộc nhóm sudo

```bash
id devops
```

Kết quả:

```
uid=1001(devops) gid=1001(devops) groups=1001(devops),27(sudo)
```

> ✅ User `devops` đã nằm trong nhóm `sudo` (GID 27).

### 3.3. Giải thích cơ chế sudoers

File `/etc/sudoers` chứa dòng cấu hình cho nhóm `sudo`:

```
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL
```

Điều này có nghĩa: **Tất cả thành viên của nhóm `sudo`** được phép chạy **bất kỳ lệnh nào** với quyền quản trị, miễn là xác nhận mật khẩu của chính mình.

---

## Bước 4: Sao chép cấu hình SSH Key từ root sang user `devops`

### 4.1. Sao chép thư mục `.ssh` bằng `rsync`

```bash
rsync --archive --chown=devops:devops ~/.ssh /home/devops/
```

**Giải thích tham số:**

| Tham số | Ý nghĩa |
|---------|---------|
| `--archive` | Sao chép đệ quy, giữ nguyên quyền truy cập, symlink, timestamp |
| `--chown=devops:devops` | Tự động đổi owner và group thành `devops` cho tất cả file/thư mục được sao chép |
| `~/.ssh` | Thư mục nguồn (SSH config của root) |
| `/home/devops/` | Thư mục đích (home của user devops) |

### 4.2. Kiểm tra kết quả sao chép

```bash
ls -la /home/devops/.ssh/
```

Kết quả mong đợi:

```
total 12
drwx------ 2 devops devops 4096 Oct  4 17:00 .
drwxr-xr-x 3 devops devops 4096 Oct  4 17:00 ..
-rw------- 1 devops devops  xxx Oct  4 17:00 authorized_keys
```

---

## Bước 5: Kiểm tra và đảm bảo phân quyền chính xác

### 5.1. Bảng phân quyền chuẩn cho SSH

| Đối tượng | Quyền | Lệnh chmod |
|-----------|-------|------------|
| `/home/devops/` | `755` (rwxr-xr-x) | `chmod 755 /home/devops/` |
| `/home/devops/.ssh/` | `700` (rwx------) | `chmod 700 /home/devops/.ssh/` |
| `/home/devops/.ssh/authorized_keys` | `600` (rw-------) | `chmod 600 /home/devops/.ssh/authorized_keys` |

### 5.2. Chạy lệnh phân quyền (nếu cần chỉnh lại)

```bash
chmod 700 /home/devops/.ssh
chmod 600 /home/devops/.ssh/authorized_keys
chown -R devops:devops /home/devops/.ssh
```

### 5.3. Xác nhận phân quyền

```bash
ls -la /home/devops/.ssh/
```

Kết quả đúng:

```
drwx------ 2 devops devops 4096 Oct  4 17:00 .       ← 700
-rw------- 1 devops devops  xxx Oct  4 17:00 authorized_keys  ← 600
```

> ⚠️ **Nếu phân quyền sai**, SSH daemon sẽ **từ chối đăng nhập** bằng key và báo lỗi `Permission denied (publickey)`.

---

## Bước 6: Kiểm tra kết nối SSH bằng user `devops`

### 6.1. Mở Terminal MỚI trên máy cá nhân (không đóng session root)

```bash
ssh -i ~/.ssh/id_ed25519 devops@<IP_ADDRESS_DROPLET>
```

### 6.2. Kết quả mong đợi

Đăng nhập thành công **không cần mật khẩu**:

```
Welcome to Ubuntu 22.04.x LTS (GNU/Linux 5.15.0-xx-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

Last login: Sun Oct  4 17:05:00 2026 from xxx.xxx.xxx.xxx

devops@ubuntu-devops-test:~$
```

> ✅ Prompt hiển thị `devops@` thay vì `root@` → đăng nhập đúng user.

### 6.3. Kiểm tra quyền sudo

```bash
sudo whoami
```

Kết quả:

```
[sudo] password for devops:   ← Nhập mật khẩu đã đặt ở Bước 2
root
```

> ✅ Kết quả trả về `root` → user `devops` có quyền chạy lệnh với đặc quyền quản trị.

---

## Bước 7: (Tùy chọn) Tắt đăng nhập root qua SSH

Để tăng cường bảo mật, sau khi xác nhận user `devops` hoạt động tốt, có thể tắt đăng nhập `root` qua SSH:

```bash
# Đăng nhập bằng devops, sau đó:
sudo nano /etc/ssh/sshd_config
```

Tìm và sửa dòng:

```
PermitRootLogin yes
```

Thành:

```
PermitRootLogin no
```

Khởi động lại SSH daemon:

```bash
sudo systemctl restart sshd
```

> ⚠️ **Lưu ý:** Chỉ thực hiện bước này sau khi **chắc chắn** user `devops` đã đăng nhập và chạy `sudo` thành công.

---

## Tóm tắt các lệnh đã sử dụng

```bash
# 1. Tạo user mới
adduser devops

# 2. Thêm user vào nhóm sudo
usermod -aG sudo devops

# 3. Sao chép SSH key từ root sang devops
rsync --archive --chown=devops:devops ~/.ssh /home/devops/

# 4. Đảm bảo phân quyền đúng
chmod 700 /home/devops/.ssh
chmod 600 /home/devops/.ssh/authorized_keys
chown -R devops:devops /home/devops/.ssh

# 5. Kiểm tra đăng nhập (từ máy cá nhân)
ssh -i ~/.ssh/id_ed25519 devops@<IP_ADDRESS_DROPLET>

# 6. Kiểm tra quyền sudo
sudo whoami
```

---

## Giải thích nguyên tắc Least Privilege

```
┌─────────────────────────────────────────────────┐
│                   root (UID 0)                  │
│         Quyền tối cao – mọi thao tác            │
│    ⚠️ Rủi ro: Một sai lầm có thể phá hủy hệ thống │
├─────────────────────────────────────────────────┤
│              devops (UID 1001)                  │
│        User thường + quyền sudo khi cần         │
│    ✅ An toàn: Phải xác nhận mật khẩu trước     │
│       khi thực hiện lệnh quản trị               │
└─────────────────────────────────────────────────┘
```

**Lợi ích:**
- Giảm thiểu rủi ro thao tác nhầm với quyền cao.
- Ghi log mọi lệnh `sudo` (kiểm tra tại `/var/log/auth.log`).
- Tuân thủ best practices bảo mật hệ thống.

---

## Tài liệu tham khảo

- [DigitalOcean - Initial Server Setup with Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04)
- [Ubuntu - Sudoers Documentation](https://help.ubuntu.com/community/Sudoers)
- [SSH Key Permissions Explained](https://superuser.com/questions/215504/permissions-on-private-key-in-ssh-folder)
