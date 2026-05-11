# Phase 0 — Dựng VM Ubuntu trong VMware Workstation Pro

> File này đi kèm `voip_project_solution.md`. Làm xong file này → có VM Ubuntu sẵn sàng → quay lại blueprint chính chạy Phase 1.

---

## Bước 1 — Cài VMware Workstation Pro 25H2

### 1.1. Tải về
- Link: https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro
- Chọn: **VMware Workstation Pro 25H2 for Windows**
- Cần đăng nhập tài khoản Broadcom (free, dùng email đăng ký)
- File: `VMware-workstation-full-25H2-xxxxx.exe` (~600 MB)

### 1.2. Cài đặt
1. Chạy file `.exe` với quyền Administrator
2. Next → Accept License
3. Chọn thư mục cài (mặc định OK)
4. **Bỏ tick** "Check for product updates on startup" (không cần)
5. Next → Install
6. Khi mở lần đầu, chọn **"Use VMware Workstation Pro for Personal Use"**
7. Nhập email Broadcom → Done

### 1.3. Kiểm tra ảo hóa CPU đã bật

Mở **Task Manager** (Ctrl+Shift+Esc) → tab **Performance** → **CPU**:
- Dòng "Virtualization": phải là **Enabled**
- Nếu **Disabled** → khởi động lại máy, vào BIOS/UEFI, tìm và bật:
  - Intel CPU: "Intel VT-x" hoặc "Intel Virtualization Technology"
  - AMD CPU: "AMD-V" hoặc "SVM Mode"

---

## Bước 2 — Tải Ubuntu Server ISO

### 2.1. Link
- https://ubuntu.com/download/server
- Chọn **Ubuntu Server 22.04.x LTS** (LTS = bản hỗ trợ dài hạn)
- File: `ubuntu-22.04.x-live-server-amd64.iso` (~2 GB)

### 2.2. Tại sao Server không phải Desktop?
| | Server | Desktop |
|---|---|---|
| GUI | Không | Có (GNOME) |
| RAM idle | ~500 MB | ~2 GB |
| Đủ cho Asterisk? | ✅ | ✅ nhưng phí |
| Học được gì | Hiểu CLI Linux | — |

---

## Bước 3 — Tạo VM mới trong VMware

### 3.1. Mở VMware → File → New Virtual Machine (Ctrl+N)

### 3.2. Chọn loại cấu hình
- ✅ **Typical (recommended)** → Next

### 3.3. Chọn nguồn cài
- ✅ **Installer disc image file (iso)** → Browse → chọn file `.iso` Ubuntu vừa tải → Next
- VMware tự nhận diện là Ubuntu 22.04 64-bit → ✅ "Easy Install" sẽ hiện ra

### 3.4. Easy Install Information
| Field | Giá trị |
|---|---|
| Full name | `student` (hoặc tên bạn) |
| User name | `ubuntu` (sẽ dùng để SSH sau) |
| Password | đặt mật khẩu dễ nhớ, vd `Asterisk@2026` |
| Confirm | nhập lại |
→ Next

### 3.5. Tên & vị trí VM
| Field | Giá trị |
|---|---|
| Virtual machine name | `Asterisk-PBX-Group08` |
| Location | mặc định, hoặc chọn ổ có nhiều dung lượng |
→ Next

### 3.6. Kích thước disk
- **Maximum disk size:** `20 GB`
- ✅ **Store virtual disk as a single file** (chọn cái này để di chuyển VM dễ)
→ Next

### 3.7. Trước khi Finish — bấm **Customize Hardware**
Đây là bước quan trọng nhất:

| Mục | Giá trị | Ghi chú |
|---|---|---|
| **Memory** | `2048 MB` | 2 GB là đủ |
| **Processors** | `2 cores` | Asterisk benefit từ multi-core |
| **Network Adapter** | ⭐ **Bridged** + tick "Replicate physical network connection state" | CỰC KỲ QUAN TRỌNG |
| USB Controller | giữ mặc định | |
| Sound Card | **Remove** | Không cần |
| Printer | **Remove** | Không cần |
| Display | giữ mặc định | |

→ Close → **Finish**

### 3.8. Tại sao phải chọn Bridged Adapter?

| Mode | VM có IP riêng trong LAN? | Softphone máy host gọi vào được? |
|---|---|---|
| **Bridged** ⭐ | ✅ (vd 192.168.1.100) | ✅ Dễ |
| NAT | ❌ (IP nội bộ VMware) | ⚠️ Phải port-forward phức tạp |
| Host-only | ❌ (chỉ thông với host) | ⚠️ Máy khác không gọi vào được |

---

## Bước 4 — Cài Ubuntu Server (auto)

VMware sẽ tự bật VM và chạy "Easy Install" — bạn **không cần làm gì**, chỉ ngồi đợi 10–20 phút.

### Nếu KHÔNG dùng Easy Install (cài thủ công)
Khi màn hình boot lên:
1. Chọn **Try or Install Ubuntu Server** → Enter
2. Language: **English** → Enter
3. Keyboard: **English (US)**
4. Type of install: **Ubuntu Server** (không phải minimized)
5. Network: để DHCP, ghi nhớ IP hiện ra (vd `192.168.1.123`)
6. Proxy: để trống
7. Mirror: mặc định
8. Storage:
   - ✅ Use an entire disk
   - ✅ Set up this disk as an LVM group
9. Profile:
   - Your name: `student`
   - Server's name: `asterisk-pbx`
   - Username: `ubuntu`
   - Password: `Asterisk@2026`
10. ⭐ **Install OpenSSH server**: tick ✅ (BẮT BUỘC để SSH vào sau)
11. Featured snaps: bỏ qua hết (Done)
12. Đợi cài xong → **Reboot Now**

### Lần đầu boot xong
- Login: `ubuntu` / `Asterisk@2026`

---

## Bước 5 — Cấu hình ban đầu trong VM

Login xong, làm các bước sau (gõ trong terminal VM):

### 5.1. Update hệ thống
```bash
sudo apt update && sudo apt upgrade -y
```

### 5.2. Xem IP VM (ghi nhớ để SSH từ Windows)
```bash
ip a
# Tìm dòng có "inet 192.168.x.x" trong interface ens33 hoặc ens160
```
Ví dụ: `192.168.1.123` — đây là IP bạn sẽ dùng cho softphone và SSH.

### 5.3. Đặt IP tĩnh (khuyến nghị, để IP không đổi sau khi reboot)

Sửa file netplan:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Thay nội dung thành (đổi IP/gateway cho phù hợp LAN của bạn):
```yaml
network:
  version: 2
  ethernets:
    ens33:                          # tên interface từ `ip a`
      dhcp4: no
      addresses: [192.168.1.123/24]
      routes:
        - to: default
          via: 192.168.1.1          # IP router của bạn
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Apply:
```bash
sudo netplan apply
```

### 5.4. Kiểm tra SSH hoạt động
```bash
sudo systemctl status ssh
# Phải có "active (running)"
```

### 5.5. Mở firewall cơ bản
```bash
sudo ufw allow OpenSSH
sudo ufw allow 5060/udp     # SIP
sudo ufw allow 5060/tcp
sudo ufw allow 4569/udp     # IAX2
sudo ufw allow 10000:20000/udp   # RTP
sudo ufw --force enable
sudo ufw status
```

---

## Bước 6 — SSH từ Windows vào VM

Trên Windows mở **PowerShell** hoặc **Windows Terminal**:

```powershell
ssh ubuntu@192.168.1.123
# Nhập password: Asterisk@2026
```

Nếu vào được → ✅ giờ bạn có thể đóng cửa sổ VMware (để VM chạy nền) và làm việc qua SSH cho tiện copy-paste lệnh.

### Mẹo SSH không cần nhập password
```powershell
# Trên Windows
ssh-keygen -t ed25519              # bỏ qua hết, Enter Enter
ssh-copy-id ubuntu@192.168.1.123   # nhập password lần cuối
# Lần sau ssh không cần password nữa
```

---

## Bước 7 — Snapshot VM (CỰC KỲ QUAN TRỌNG)

Trước khi cài Asterisk, **chụp snapshot** để nếu hỏng còn rollback được:

1. Trong VMware, chọn VM → menu **VM → Snapshot → Take Snapshot**
2. Name: `Ubuntu-clean-installed`
3. Description: "Trước khi cài Asterisk"
4. **Take Snapshot**

Nếu sau này có lỗi config không sửa được → **VM → Snapshot → Snapshot Manager → Restore** → quay về trạng thái sạch trong 30 giây.

---

## ✅ Hoàn thành Phase 0

Giờ bạn đã có:
- [x] VM Ubuntu 22.04 Server chạy ổn
- [x] IP tĩnh (vd `192.168.1.123`)
- [x] SSH hoạt động
- [x] Firewall mở các port VoIP
- [x] Snapshot sạch để rollback

**Bước tiếp theo:** Quay lại `voip_project_solution.md` → **Phase 1: Cài đặt Asterisk** → copy script `install_asterisk.sh` vào VM và chạy.

---

## Troubleshooting Phase 0

### VM không boot được — "VT-x is disabled"
→ Vào BIOS bật virtualization (xem Bước 1.3)

### Mạng Bridged không có IP
- Trong VMware: **Edit → Virtual Network Editor → VMnet0 → Bridged to**: chọn đúng card mạng đang dùng (Wi-Fi hoặc Ethernet)
- Reboot VM

### SSH từ Windows báo "Connection refused"
- Trong VM: `sudo systemctl restart ssh`
- Kiểm tra firewall: `sudo ufw status`
- Kiểm tra IP đúng chưa: `ip a` trong VM

### VM chạy chậm / lag
- Tăng RAM lên 4 GB
- Tăng CPU lên 4 core
- Đóng các app khác trên Windows
- Đảm bảo Virtualization đã bật trong BIOS

### Quên password Ubuntu
- Reboot VM, ở màn hình GRUB nhấn `e` → thêm `init=/bin/bash` vào cuối dòng `linux` → Ctrl+X
- Boot vào → `mount -o remount,rw /` → `passwd ubuntu` → đặt password mới → reboot

---

**HẾT Phase 0.** Tiếp tục với `voip_project_solution.md` Phase 1.
