# Báo cáo Bài 2: Cấu hình phân quyền Nhóm và sudoers bằng visudo

## 1. Mục tiêu & Yêu cầu
- **Mục tiêu:**
  - Nắm vững cách tạo Group (`devops-admin`) và User (`deployer`) trên hệ điều hành Linux.
  - Hiểu cơ chế hoạt động của đặc quyền quản trị `sudo` và tệp cấu hình `/etc/sudoers`.
  - Áp dụng nguyên tắc đặc quyền tối thiểu (*Principle of Least Privilege*): Cấp quyền điều khiển dịch vụ hệ thống qua `systemctl` (start, stop, restart, status) mà không cấp toàn quyền root và không cần nhập mật khẩu (`NOPASSWD`).

---

## 2. Các bước triển khai

### Bước 1: Tạo nhóm `devops-admin`
```bash
sudo groupadd devops-admin
```

### Bước 2: Tạo người dùng `deployer` và gán vào nhóm `devops-admin`
```bash
sudo useradd -m -s /bin/bash -G devops-admin deployer
```
*Kiểm tra thông tin tài khoản:*
```bash
id deployer
```
**Kết quả:**
```
uid=1001(deployer) gid=1002(deployer) groups=1002(deployer),1001(devops-admin)
```

---

### Bước 3: Cấu hình phân quyền trong `/etc/sudoers`

Mở trình chỉnh sửa cấu hình sudoers an toàn bằng lệnh `visudo`:
```bash
sudo visudo
# Hoặc cấu hình theo file riêng trong thư mục include:
# sudo visudo -f /etc/sudoers.d/devops-admin
```

Thêm dòng cấu hình phân quyền cho nhóm `devops-admin` vào cuối file:
```sudoers
%devops-admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

#### Giải thích cú pháp:
- `%devops-admin`: Dấu `%` chỉ định đây là một **Group** (thay vì user đơn lẻ). Tất cả user thuộc nhóm này đều được áp dụng quy tắc.
- `ALL=`: Quy tắc có hiệu lực trên tất cả máy chủ/mạng (Host).
- `(ALL)`: Có thể thực thi lệnh dưới tư cách bất kỳ user/group nào (thường mặc định là root).
- `NOPASSWD:`: Cho phép thực thi các lệnh được liệt kê phía sau mà **không cần nhập mật khẩu**.
- `/usr/bin/systemctl start *, /usr/bin/systemctl stop *, ...`: Chỉ cho phép thực thi lệnh `systemctl` với đúng 4 thao tác quản lý dịch vụ (`start`, `stop`, `restart`, `status`). Dấu `*` đại diện cho tên bất kỳ dịch vụ nào cần điều khiển.

Kiểm tra cú pháp tệp cấu hình để đảm bảo không có lỗi:
```bash
sudo visudo -c
```
**Kết quả:**
```
/etc/sudoers: parsed OK
```

---

## 3. Kiểm tra và Xác thực kết quả

### 3.1. Chuyển sang tài khoản `deployer`
```bash
su - deployer
```

### 3.2. Kiểm tra danh sách lệnh đặc quyền sudo được phép (`sudo -l`)
```bash
sudo -l
```
**Kết quả hiển thị trên terminal:**
```
User deployer may run the following commands on HuyTuan:
    (ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```
*Nhận xét:* Hệ thống xác nhận user `deployer` (thông qua nhóm `devops-admin`) chỉ có quyền chạy `systemctl start|stop|restart|status` mà không cần mật khẩu.

---

### 3.3. Thực hiện khởi động lại dịch vụ hệ thống (`cron`)
```bash
sudo systemctl restart cron
sudo systemctl status cron
```
**Kết quả chạy thực tế:**
```text
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-10-06 12:11:59 UTC; 8s ago
 Invocation: f871a7fff9324c1883ce8b41a15d692a
       Docs: man:cron(8)
   Main PID: 494 (cron)
      Tasks: 1 (limit: 14163)
     Memory: 352K (peak: 1.9M)
        CPU: 7ms
     CGroup: /system.slice/cron.service
             └─494 /usr/sbin/cron -f -P

Oct 06 12:11:59 HuyTuan systemd[1]: Started cron.service - Regular background program processing daemon.
Oct 06 12:11:59 HuyTuan (cron)[494]: cron.service: Referenced but unset environment variable evaluates to an empty string: EXTRA_OPTS
Oct 06 12:11:59 HuyTuan cron[494]: (CRON) INFO (pidfile fd = 3)
Oct 06 12:11:59 HuyTuan cron[494]: (CRON) INFO (Skipping @reboot jobs -- not system startup)
```
*Nhận xét:* Lệnh `sudo systemctl restart cron` hoàn tất ngay lập tức và thành công mà không yêu cầu nhập mật khẩu.

---

### 3.4. Kiểm tra bảo mật (Thử nghiệm lệnh trái phép)
Thử thực hiện các lệnh yêu cầu quyền root khác không nằm trong danh mục được cấp quyền:
```bash
sudo cat /etc/shadow
```
**Kết quả:**
```text
sudo: I'm sorry deployer. I'm afraid I can't do that.
```
*Nhận xét:* Hệ thống từ chối thực thi ngay lập tức, đảm bảo tài khoản `deployer` không thể leo thang đặc quyền để can thiệp vào các tài nguyên nhạy cảm khác của hệ điều hành.

---

## 4. Kết luận
- Đã hoàn thành toàn bộ yêu cầu của bài thực hành:
  1. Tạo Group `devops-admin` và User `deployer`.
  2. Cấu hình đúng cú pháp trong sudoers với `NOPASSWD` và giới hạn lệnh `systemctl`.
  3. Kiểm tra thực tế xác nhận phân quyền thành công và đảm bảo an toàn thông tin theo chuẩn vận hành DevOps/Ops.
