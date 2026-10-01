# BÀI 4: CẤU HÌNH TƯỜNG LỬA BẢO VỆ MÁY CHỦ (UFW & CLOUD FIREWALL INTEGRATION)

## 1. Mục tiêu
- Thiết lập tường lửa phần mềm **UFW (Uncomplicated Firewall)** tích hợp sẵn trên Ubuntu Server để kiểm soát lưu lượng mạng ở tầng hệ điều hành.
- Cấu hình **DigitalOcean Cloud Firewall** ở tầng hạ tầng để sàng lọc và ngăn chặn các gói tin độc hại từ biên mạng trước khi tiếp cận Droplet.
- Triển khai mô hình phòng thủ theo chiều sâu (**Defense in Depth**) 2 lớp, chỉ cho phép các cổng dịch vụ thiết yếu: **SSH (22/tcp)** và **HTTP (80/tcp)**.

---

## 2. Môi trường thực hiện
- **Hệ điều hành:** Ubuntu Server (chạy trên DigitalOcean Droplet).
- **Lớp tường lửa 1 (OS Level):** UFW (Uncomplicated Firewall).
- **Lớp tường lửa 2 (Cloud Infrastructure Level):** DigitalOcean Cloud Firewall.
- **Các cổng dịch vụ cho phép:**
  - Cổng 22/tcp: Dịch vụ truy cập quản trị SSH.
  - Cổng 80/tcp: Dịch vụ web tĩnh Nginx (HTTP).

---

## 3. Cấu hình Lớp 1: Tường lửa UFW (Hệ điều hành)

### 3.1. Thiết lập chính sách mặc định (Default Policies)
Thiết lập nguyên tắc bảo mật: Chặn toàn bộ kết nối đi vào và cho phép toàn bộ kết nối đi ra:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 3.2. Mở các cổng dịch vụ cần thiết
Cho phép kết nối qua cổng SSH (22) và HTTP (80) trước khi kích hoạt tường lửa:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
```

### 3.3. Kích hoạt UFW
Khởi chạy và bật tính năng tự khởi động cùng hệ thống:

```bash
sudo ufw enable
```

> **Lưu ý quan trọng:** Luôn phải mở cổng `22/tcp` trước khi chạy `ufw enable`, nếu không bạn sẽ bị mất quyền truy cập SSH vào máy chủ.

### 3.4. Kiểm tra trạng thái UFW
Xác minh trạng thái chi tiết của tường lửa:

```bash
sudo ufw status verbose
```

---

## 4. Cấu hình Lớp 2: DigitalOcean Cloud Firewall (Hạ tầng mạng)

### 4.1. Các bước thực hiện trên DigitalOcean Console
1. Đăng nhập vào trang quản trị **DigitalOcean Control Panel**.
2. Điều hướng tới mục **Manage** -> **Networking** -> chọn tab **Firewalls**.
3. Nhấp chọn **Create Firewall** và đặt tên cho firewall (ví dụ: `fw-droplet-web`).
4. Thiết lập **Inbound Rules** và **Outbound Rules** theo bảng cấu hình bên dưới.
5. Tại mục **Apply to Droplets**, nhập tên Droplet cần bảo vệ và nhấn **Create Firewall**.

### 4.2. Bảng quy tắc Firewall (Firewall Rules)

**Inbound Rules (Lưu lượng đi vào):**
| Loại dịch vụ (Type) | Giao thức (Protocol) | Dải cổng (Port Range) | Nguồn cho phép (Sources) |
| :--- | :---: | :---: | :--- |
| **SSH** | TCP | 22 | All IPv4 (`0.0.0.0/0`), All IPv6 (`::/0`) |
| **HTTP** | TCP | 80 | All IPv4 (`0.0.0.0/0`), All IPv6 (`::/0`) |

**Outbound Rules (Lưu lượng đi ra):**
| Loại dịch vụ (Type) | Giao thức (Protocol) | Dải cổng (Port Range) | Đích đến (Destinations) |
| :--- | :---: | :---: | :--- |
| **All TCP** | TCP | All | All IPv4, All IPv6 |
| **All UDP** | UDP | All | All IPv4, All IPv6 |
| **All ICMP** | ICMP | All | All IPv4, All IPv6 |

---

## 5. Kiểm tra và xác thực kết nối

### 5.1. Kiểm tra trên máy chủ Droplet
Chạy lệnh kiểm tra danh sách rules đang hoạt động của UFW:

```bash
sudo ufw status verbose
```

**Kết quả mong đợi:**
- Trạng thái `Status: active`.
- `Default: deny (incoming), allow (outgoing), disabled (routed)`.
- Xuất hiện cổng `22/tcp` và `80/tcp` ở trạng thái `ALLOW IN`.

### 5.2. Kiểm tra từ máy cá nhân (Client)
1. **Kiểm tra kết nối SSH (Cổng 22):**
   ```bash
   ssh -i /path/to/private_key devops@<IP_ADDRESS_DROPLET>
   ```
   *Kết quả mong đợi:* Đăng nhập thành công vào VPS.

2. **Kiểm tra dịch vụ HTTP (Cổng 80):**
   ```bash
   curl -I http://<IP_ADDRESS_DROPLET>
   ```
   *Kết quả mong đợi:* Trả về mã phản hồi `HTTP/1.1 200 OK`.

3. **Kiểm tra cổng không được phép (Ví dụ cổng 8080 hoặc 3306):**
   ```bash
   nc -zv -w 3 <IP_ADDRESS_DROPLET> 8080
   ```
   *Kết quả mong đợi:* Kết nối bị từ chối hoặc hết thời gian chờ (`timed out` / `Connection refused`), chứng minh tường lửa đã chặn thành công.

---

## 6. Kết quả thực hiện

### 6.1. Log kích hoạt và kiểm tra UFW trên máy chủ
```bash
devops@ubuntu-server:~$ sudo ufw default deny incoming
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)

devops@ubuntu-server:~$ sudo ufw default allow outgoing
Default outgoing policy changed to 'allow'
(be sure to update your rules accordingly)

devops@ubuntu-server:~$ sudo ufw allow 22/tcp
Rules updated
Rules updated (v6)

devops@ubuntu-server:~$ sudo ufw allow 80/tcp
Rules updated
Rules updated (v6)

devops@ubuntu-server:~$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup

devops@ubuntu-server:~$ sudo ufw status verbose
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

### 6.2. Ảnh chụp thiết lập DigitalOcean Cloud Firewall trên Console
<!-- Đính kèm ảnh chụp màn hình cấu hình Inbound Rules và Apply to Droplets trên DigitalOcean -->
```text
+-----------------------------------------------------------------------------------+
| DigitalOcean Cloud Firewall: fw-droplet-web                                       |
|                                                                                   |
| Inbound Rules:                                                                    |
|  - SSH    | TCP | Port 22 | Sources: All IPv4, All IPv6                           |
|  - HTTP   | TCP | Port 80 | Sources: All IPv4, All IPv6                           |
|                                                                                   |
| Applied Droplets:                                                                 |
|  - ubuntu-server-droplet (Applied)                                                |
+-----------------------------------------------------------------------------------+
[Dán đường dẫn ảnh chụp DigitalOcean Console tại đây nếu có]
```

### 6.3. Kiểm tra kết nối từ máy Client (SSH, HTTP và Port bị chặn)
```bash
# 1. Kiểm tra SSH cổng 22: Thành công
user@local-pc:~$ ssh -i ~/.ssh/id_rsa devops@159.89.123.45
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-105-generic x86_64)
devops@ubuntu-server:~$ exit
logout
Connection to 159.89.123.45 closed.

# 2. Kiểm tra truy cập HTTP cổng 80: Thành công (HTTP 200 OK)
user@local-pc:~$ curl -I http://159.89.123.45
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Thu, 01 Oct 2026 20:40:00 GMT
Content-Type: text/html
Content-Length: 228
Connection: keep-alive

# 3. Kiểm tra cổng chưa mở (Port 8080): Bị chặn hoàn toàn
user@local-pc:~$ nc -zv -w 3 159.89.123.45 8080
nc: connect to 159.89.123.45 port 8080 (tcp) timed out: Operation now in progress
```

---

## 7. Kết luận
Bài thực hành đã triển khai thành công hệ thống phòng vệ 2 lớp (Defense in Depth) cho máy chủ:
- **Tầng hệ điều hành (UFW):** Đã thiết lập chính sách chặn mặc định, chỉ mở 2 cổng `22/tcp` và `80/tcp`, bảo vệ an toàn cho các tiến trình nội bộ.
- **Tầng biên hạ tầng (Cloud Firewall):** Đã lọc gói tin trực tiếp từ biên mạng đám mây của DigitalOcean, giảm thiểu tải xử lý và ngăn ngừa tấn công thăm dò cổng vào Droplet.
- **Kiểm thử thực tế:** Các kết nối quản trị SSH và dịch vụ Web HTTP hoạt động ổn định, các cổng dịch vụ khác ngoài danh mục cho phép đều bị ngắt kết nối và chặn triệt để.
