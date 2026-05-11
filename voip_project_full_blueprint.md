# Project 2 — Cấu hình Tổng đài thoại VoIP
## Blueprint hoàn chỉnh từ A→Z cho AI Agent

> **HƯỚNG DẪN CHO AI AGENT (Cowork / Claude Code)**
>
> File này là blueprint duy nhất, đầy đủ từ bước cài VMware đến lúc đóng gói nộp bài. Agent đọc tuần tự, dừng lại ở các bước **🛑 STOP** để chờ user thao tác vật lý (cài app, nhập trong VM, quay video).
>
> **Tham số cần đổi trước khi chạy:**
> - `GROUP_NUMBER` = `08` (đổi theo nhóm thực, luôn 2 chữ số)
> - `SERVER_IP` = sẽ biết sau khi tạo VM (vd `192.168.1.123`)
> - `EXTERNAL_NUMBER` = `0951000008` (số ngoài giả lập)
> - `PUBLIC_NUMBER` = `0952014308` (số public công ty)

---

## MỤC LỤC

- **PHẦN A** — Dựng môi trường (VMware + Ubuntu)
- **PHẦN B** — Cài Asterisk 20 LTS
- **PHẦN C** — File cấu hình (.conf)
- **PHẦN D** — File âm thanh tiếng Việt
- **PHẦN E** — Triển khai + test
- **PHẦN F** — Báo cáo + nộp bài
- **PHẦN G** — Troubleshooting
- **PHẦN H** — Checklist cho agent

---

## Bảng extension (mặc định GROUP_NUMBER=08)

| Phòng | Ext | Giao thức | Password |
|---|---|---|---|
| Giám đốc | 5085 | IAX2 | Pass5085! |
| Nhân sự | 6086 | SIP | Pass6086! |
| Kỹ thuật 1 | 7081 | IAX2 | Pass7081! |
| Kỹ thuật 2 | 7082 | SIP | Pass7082! |
| Bán hàng 1 | 8080 | SIP | Pass8080! |
| Bán hàng 2 | 8086 | IAX2 | Pass8086! |
| Bán hàng 3 | 8088 | SIP | Pass8088! |
| External (giả lập) | 0951000008 | SIP | ExtPass! |
| Public (gọi vào) | 0952014308 | — | — |
| Conference | 4084 | MeetMe | user=654321, admin=123456 |
| Voicemail check | 500 | — | PIN 1234 |

---

# 🅰️ PHẦN A — DỰNG MÔI TRƯỜNG

## A1. Cài VMware Workstation Pro 25H2 🛑 USER

### Tải về
- Link: https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro
- Chọn: **VMware Workstation Pro 25H2 for Windows**
- Cần tài khoản Broadcom (free)

### Cài đặt
1. Chạy file `.exe` với quyền Administrator
2. Next → Accept License → Next
3. **Bỏ tick** "Check for product updates"
4. Install
5. Mở lần đầu → chọn **"Use VMware Workstation Pro for Personal Use"**
6. Nhập email Broadcom → Done

### Kiểm tra ảo hóa CPU
- Task Manager (Ctrl+Shift+Esc) → Performance → CPU
- Dòng **Virtualization: Enabled** ✅
- Nếu **Disabled** → restart, vào BIOS bật **Intel VT-x** / **AMD-V**

---

## A2. Tải Ubuntu 24.04 LTS Server ISO 🛑 USER

- Link: https://ubuntu.com/download/server
- File: `ubuntu-24.04.x-live-server-amd64.iso` (~3.2 GB)

**Tại sao 24.04?** Có Asterisk 20 LTS trong repo (cài 5 phút bằng apt) + hỗ trợ chan_sip & MeetMe khớp đề bài.

---

## A3. Tạo VM trong VMware 🛑 USER

1. VMware → **File → New Virtual Machine** (Ctrl+N)
2. **Typical** → Next
3. **Installer disc image file** → Browse → chọn file ISO → Next
4. Easy Install info:
   - Full name: `student`
   - Username: `ubuntu`
   - Password: `Asterisk@2026`
5. VM name: `Asterisk-PBX-Group08`, location mặc định → Next
6. Disk: `20 GB`, ✅ **Store as single file** → Next
7. **Customize Hardware** (BẮT BUỘC):

| Thiết bị | Giá trị |
|---|---|
| Memory | `2048 MB` |
| Processors | `2 cores` |
| Network Adapter | ⭐ **Bridged** + tick "Replicate physical network connection state" |
| Sound Card | **Remove** |
| Printer | **Remove** |

8. Close → **Finish**

VMware sẽ tự bật VM và chạy Easy Install (~15–20 phút). Không cần làm gì, đi pha cà phê.

---

## A4. Cấu hình ban đầu trong VM 🛑 USER

Login vào VM (`ubuntu` / `Asterisk@2026`) rồi chạy:

### A4.1. Lấy IP VM
```bash
ip a
# Tìm dòng "inet 192.168.x.x/24" → ghi nhớ IP này, vd 192.168.1.123
```

### A4.2. Update hệ thống
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget vim net-tools
```

### A4.3. Đặt IP tĩnh (để IP không đổi sau reboot)

Sửa netplan:
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Thay nội dung (đổi IP/gateway cho phù hợp LAN của bạn):
```yaml
network:
  version: 2
  ethernets:
    ens33:                          # tên interface từ `ip a`
      dhcp4: no
      addresses: [192.168.1.123/24]
      routes:
        - to: default
          via: 192.168.1.1          # IP router
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Apply:
```bash
sudo netplan apply
```

### A4.4. Mở firewall cho VoIP
```bash
sudo ufw allow OpenSSH
sudo ufw allow 5060/udp           # SIP
sudo ufw allow 5060/tcp
sudo ufw allow 4569/udp           # IAX2
sudo ufw allow 10000:20000/udp    # RTP media
sudo ufw --force enable
sudo ufw status
```

### A4.5. SSH từ Windows vào VM (tùy chọn nhưng khuyến nghị)

Trên Windows PowerShell:
```powershell
ssh ubuntu@192.168.1.123
# nhập Asterisk@2026
```

→ Từ giờ làm việc qua SSH cho dễ copy-paste lệnh.

---

## A5. Snapshot trước khi cài Asterisk 🛑 USER

VMware menu: **VM → Snapshot → Take Snapshot**
- Name: `Ubuntu-clean-installed`
- Description: "Trước khi cài Asterisk"
- **Take Snapshot**

→ Lỡ hỏng sau này → Restore về đây trong 30 giây.

✅ **Kết thúc Phần A.** Sang Phần B.

---

# 🅱️ PHẦN B — CÀI ASTERISK 20 LTS

## B1. Cài Asterisk qua APT (chỉ 5 phút trên Ubuntu 24.04)

Trên VM (qua SSH hoặc terminal VM):

```bash
# Cài Asterisk + module + sox để xử lý audio
sudo apt install -y asterisk asterisk-modules asterisk-config \
    asterisk-mp3 asterisk-voicemail \
    dahdi-dkms dahdi-linux dahdi sox libsox-fmt-mp3

# Bật DAHDI dummy cho MeetMe (vì VM không có card timing thật)
sudo modprobe dahdi_dummy
echo "dahdi_dummy" | sudo tee -a /etc/modules

# Enable + start service
sudo systemctl enable asterisk
sudo systemctl restart asterisk
```

## B2. Verify Asterisk chạy

```bash
sudo systemctl status asterisk
# Phải thấy "active (running)"

sudo asterisk -rvvv
# Vào CLI Asterisk, prompt: asterisk*CLI>
```

Trong CLI test:
```
asterisk*CLI> core show version
asterisk*CLI> module show like chan_sip
asterisk*CLI> module show like chan_iax2
asterisk*CLI> module show like app_meetme
asterisk*CLI> exit
```

Nếu các module trên đều **Running** → ✅ OK. Nếu chan_sip không load (Asterisk 20 deprecated chan_sip), enable lại ở bước B3.

## B3. Enable chan_sip (vì đề bài yêu cầu `sip.conf`)

Asterisk 20 mặc định ưu tiên chan_pjsip, nhưng đề bài muốn `sip.conf` truyền thống → ép load chan_sip:

```bash
sudo nano /etc/asterisk/modules.conf
```

Đảm bảo có dòng (thêm vào nếu chưa có):
```ini
[modules]
autoload=yes

; Bật chan_sip cổ điển
load => chan_sip.so

; Disable chan_pjsip để tránh xung đột port 5060
noload => chan_pjsip.so
noload => res_pjsip.so
noload => res_pjsip_session.so
noload => res_pjsip_pubsub.so
```

Reload:
```bash
sudo systemctl restart asterisk
sudo asterisk -rx "module show like chan_sip"
# State: Running
```

## B4. Backup config gốc (trước khi đè bằng config của ta)

```bash
sudo cp -r /etc/asterisk /etc/asterisk.bak.$(date +%Y%m%d)
```

✅ **Kết thúc Phần B.** Sang Phần C.

---

# 🅲 PHẦN C — FILE CẤU HÌNH

Agent tạo các file sau trên máy host trong thư mục `voip-project-group08/configs/`, rồi sau đó deploy lên VM bằng script ở Phần E.

## C1. `configs/sip.conf`

```ini
[general]
context=default
allowoverlap=no
udpbindaddr=0.0.0.0:5060
tcpenable=no
transport=udp
srvlookup=yes
disallow=all
allow=ulaw
allow=alaw
allow=gsm
language=vi
nat=force_rport,comedia
qualify=yes
dtmfmode=rfc2833

;========== TEMPLATE ==========
[office-sip](!)
type=friend
host=dynamic
context=internal
disallow=all
allow=ulaw
allow=alaw
allow=gsm
qualify=yes
nat=force_rport,comedia
dtmfmode=rfc2833

;========== PHÒNG NHÂN SỰ ==========
[6086](office-sip)
secret=Pass6086!
callerid="HR Staff" <6086>
mailbox=6086@default

;========== PHÒNG KỸ THUẬT ==========
[7082](office-sip)
secret=Pass7082!
callerid="Tech 02" <7082>
mailbox=7082@default

;========== PHÒNG BÁN HÀNG ==========
[8080](office-sip)
secret=Pass8080!
callerid="Sales 01" <8080>
mailbox=8080@default

[8088](office-sip)
secret=Pass8088!
callerid="Sales 03" <8088>
mailbox=8088@default

;========== TRUNK GIẢ LẬP SỐ NGOÀI ==========
; Đăng ký 1 softphone với username=external-trunk để giả lập "máy ngoài công ty"
[external-trunk]
type=friend
host=dynamic
context=from-external
secret=ExtPass!
disallow=all
allow=ulaw
allow=alaw
```

## C2. `configs/iax.conf`

```ini
[general]
bindport=4569
bindaddr=0.0.0.0
disallow=all
allow=ulaw
allow=alaw
allow=gsm
language=vi
jitterbuffer=yes
forcejitterbuffer=yes

[office-iax](!)
type=friend
host=dynamic
context=internal
disallow=all
allow=ulaw
allow=alaw
allow=gsm
qualify=yes

;========== PHÒNG GIÁM ĐỐC ==========
[5085](office-iax)
secret=Pass5085!
callerid="Director" <5085>
mailbox=5085@default

;========== PHÒNG KỸ THUẬT ==========
[7081](office-iax)
secret=Pass7081!
callerid="Tech 01" <7081>
mailbox=7081@default

;========== PHÒNG BÁN HÀNG ==========
[8086](office-iax)
secret=Pass8086!
callerid="Sales 02" <8086>
mailbox=8086@default
```

## C3. `configs/voicemail.conf`

```ini
[general]
format=wav49|gsm|wav
serveremail=asterisk@uit.local
attach=yes
maxmsg=100
maxsecs=180
minsecs=3
maxlogins=3
emaildateformat=%A, %d %B %Y at %H:%M:%S
sendvoicemail=yes

[zonemessages]
vietnam=Asia/Ho_Chi_Minh|'vm-received' Q 'digits/at' HM

[default]
5085 => 1234,Director,director@company.local,,attach=yes|tz=vietnam
6086 => 1234,HR Staff,hr@company.local
7081 => 1234,Tech 01,tech01@company.local
7082 => 1234,Tech 02,tech02@company.local
8080 => 1234,Sales 01,sales01@company.local
8086 => 1234,Sales 02,sales02@company.local
8088 => 1234,Sales 03,sales03@company.local
```

## C4. `configs/meetme.conf`

```ini
[general]
audiobuffers=32

[rooms]
; Room 4084: phong hop noi bo
; Cú pháp: conf => roomno,user_pin,admin_pin
; Theo đề: "password QUẢN LÝ và GIA NHẬP lần lượt là 123456 và 654321"
;   -> admin_pin (quản lý) = 123456
;   -> user_pin  (gia nhập) = 654321
conf => 4084,654321,123456
```

## C5. `configs/musiconhold.conf`

```ini
[default]
mode=files
directory=moh
```

## C6. `configs/extensions.conf` ⭐ (file trung tâm)

```ini
[general]
static=yes
writeprotect=no
clearglobalvars=no

[globals]
GROUP=08
DIRECTOR=5085
HR=6086
TECH1=7081
TECH2=7082
SALES1=8080
SALES2=8086
SALES3=8088
CONFROOM=4084
PUBLIC_NUM=0952014308

;==================================================
; CONTEXT: internal — cho các extension nội bộ
;==================================================
[internal]
; --- Gọi nội bộ trực tiếp ---
exten => _5XX5,1,NoOp(Goi Giam doc)
 same => n,Dial(IAX2/${EXTEN},20,tT)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

exten => _6XX6,1,NoOp(Goi Nhan su)
 same => n,Dial(SIP/${EXTEN},20,tT)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

exten => _7XX1,1,NoOp(Goi Ky thuat IAX)
 same => n,Dial(IAX2/${EXTEN},20,tT)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

exten => _7XX2,1,NoOp(Goi Ky thuat SIP)
 same => n,Dial(SIP/${EXTEN},20,tT)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

exten => _8XX0,1,NoOp(Goi Ban hang 1)
 same => n,Dial(SIP/${EXTEN},20,tT)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

exten => _8XX6,1,NoOp(Goi Ban hang 2)
 same => n,Dial(IAX2/${EXTEN},20,tT)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

exten => _8XX8,1,NoOp(Goi Ban hang 3)
 same => n,Dial(SIP/${EXTEN},20,tT)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

; --- Phòng họp nội bộ (room conference) ---
exten => _4XX4,1,NoOp(Phong hop noi bo)
 same => n,Answer()
 same => n,MeetMe(${EXTEN},pPMs)
 same => n,Hangup()

; --- Voicemail check: bấm 500 để nghe thư thoại ---
exten => 500,1,NoOp(Voicemail main)
 same => n,Answer()
 same => n,VoiceMailMain(${CALLERID(num)}@default)
 same => n,Hangup()

; --- Gọi ra ngoài (prefix 9) ---
exten => _9.,1,NoOp(Goi ra ngoai: ${EXTEN:1})
 same => n,Set(CALLERID(num)=${PUBLIC_NUM})
 same => n,Dial(SIP/external-trunk/${EXTEN:1},30)
 same => n,Hangup()

; --- Catch-all ---
exten => i,1,Playback(invalid)
 same => n,Hangup()
exten => t,1,Playback(vm-goodbye)
 same => n,Hangup()


;==================================================
; CONTEXT: from-external — gọi từ ngoài vào số public
;==================================================
[from-external]
; Bất kỳ cuộc gọi nào đến số public đều vào IVR
exten => _X.,1,NoOp(Cuoc goi tu ngoai: ${CALLERID(num)} -> ${EXTEN})
 same => n,Answer()
 same => n,Wait(1)
 same => n,Goto(ivr-main,s,1)


;==================================================
; CONTEXT: ivr-main — IVR chào mừng + 5 lựa chọn
;==================================================
[ivr-main]
exten => s,1,NoOp(IVR chinh)
 same => n,Set(TIMEOUT(digit)=5)
 same => n,Set(TIMEOUT(response)=10)
 same => n,Background(custom/welcome)
 same => n,WaitExten(10)

exten => 1,1,Goto(ivr-sales,s,1)
exten => 2,1,Goto(ivr-tech,s,1)
exten => 3,1,Goto(ivr-hr,s,1)
exten => 4,1,Goto(ivr-feedback,s,1)
exten => 5,1,Goto(ivr-main,s,1)        ; phát lại lời chào

exten => i,1,Playback(invalid)
 same => n,Goto(ivr-main,s,1)
exten => t,1,Playback(vm-goodbye)
 same => n,Hangup()


;==================================================
; CONTEXT: ivr-sales — phím 1 (Phòng bán hàng)
;==================================================
[ivr-sales]
exten => s,1,NoOp(IVR Sales - ring all)
 same => n,Playback(custom/sales-welcome)
 ; Ring đồng loạt 3 số (SIP + IAX)
 same => n,Dial(SIP/8080&IAX2/8086&SIP/8088,25,tT)
 same => n,NoOp(Dial status = ${DIALSTATUS})
 same => n,GotoIf($["${DIALSTATUS}"="ANSWER"]?end)
 same => n,Playback(custom/sales-busy)
 same => n,VoiceMail(8080@default,u)
 same => n(end),Hangup()


;==================================================
; CONTEXT: ivr-tech — phím 2 (Phòng kỹ thuật)
;==================================================
[ivr-tech]
exten => s,1,NoOp(IVR Tech - ring sequential)
 same => n,Dial(IAX2/7081,15,tT)
 same => n,GotoIf($["${DIALSTATUS}"="ANSWER"]?end)
 same => n,Dial(SIP/7082,15,tT)
 same => n,GotoIf($["${DIALSTATUS}"="ANSWER"]?end)
 same => n,Playback(custom/tech-busy)
 same => n,Hangup()
 same => n(end),Hangup()


;==================================================
; CONTEXT: ivr-hr — phím 3 (Phòng nhân sự)
;==================================================
[ivr-hr]
exten => s,1,NoOp(IVR HR)
 same => n,Dial(SIP/6086,25,tT)
 same => n,GotoIf($["${DIALSTATUS}"="ANSWER"]?end)
 same => n,VoiceMail(6086@default,u)
 same => n(end),Hangup()


;==================================================
; CONTEXT: ivr-feedback — phím 4 (Để lại VM cho GĐ)
;==================================================
[ivr-feedback]
exten => s,1,NoOp(IVR Feedback - leave voicemail to Director)
 same => n,Playback(custom/feedback-thanks)
 same => n,VoiceMail(5085@default,s)
 same => n,Hangup()


;==================================================
; CONTEXT: default
;==================================================
[default]
exten => _X.,1,Hangup()
```

✅ **Kết thúc Phần C.** Sang Phần D.

---

# 🅳 PHẦN D — FILE ÂM THANH TIẾNG VIỆT

## D1. `sounds/script.txt` — kịch bản lời thoại

```
welcome.wav        | Chào mừng quý khách gọi đến công ty ABC. Vui lòng nhấn phím 1 để kết nối với phòng bán hàng. Nhấn phím 2 để được hỗ trợ kỹ thuật. Nhấn phím 3 để biết thông tin tuyển dụng. Nhấn phím 4 để để lại lời nhắn hoặc góp ý cho Ban Giám Đốc. Nhấn phím 5 để nghe lại lời chào.
sales-welcome.wav  | Chào mừng bạn đã đến phòng bán hàng. Vui lòng đợi trong giây lát để được kết nối với điện thoại viên.
sales-busy.wav     | Xin lỗi, hiện tại các điện thoại viên đều bận. Vui lòng để lại lời nhắn sau tiếng pip, hoặc thực hiện lại cuộc gọi.
tech-busy.wav      | Xin lỗi, hiện tại các kỹ thuật viên đều bận. Vui lòng chờ trong giây lát để thực hiện lại cuộc gọi.
feedback-thanks.wav| Xin chân thành cảm ơn bạn đã góp ý cho công ty chúng tôi. Vui lòng để lại lời nhắn sau tiếng pip.
```

## D2. `scripts/gen_audio.py` — sinh WAV bằng Google TTS

```python
#!/usr/bin/env python3
"""Sinh WAV tieng Viet cho IVR Asterisk (8kHz, mono, PCM 16-bit)."""
import os
import subprocess
from gtts import gTTS

SCRIPTS = {
    "welcome": "Chào mừng quý khách gọi đến công ty ABC. "
               "Vui lòng nhấn phím 1 để kết nối với phòng bán hàng. "
               "Nhấn phím 2 để được hỗ trợ kỹ thuật. "
               "Nhấn phím 3 để biết thông tin tuyển dụng. "
               "Nhấn phím 4 để để lại lời nhắn hoặc góp ý cho Ban Giám Đốc. "
               "Nhấn phím 5 để nghe lại lời chào.",
    "sales-welcome": "Chào mừng bạn đã đến phòng bán hàng. "
                     "Vui lòng đợi trong giây lát để được kết nối với điện thoại viên.",
    "sales-busy": "Xin lỗi, hiện tại các điện thoại viên đều bận. "
                  "Vui lòng để lại lời nhắn sau tiếng pip, hoặc thực hiện lại cuộc gọi.",
    "tech-busy": "Xin lỗi, hiện tại các kỹ thuật viên đều bận. "
                 "Vui lòng chờ trong giây lát để thực hiện lại cuộc gọi.",
    "feedback-thanks": "Xin chân thành cảm ơn bạn đã góp ý cho công ty chúng tôi. "
                       "Vui lòng để lại lời nhắn sau tiếng pip.",
}

OUTPUT_DIR = "../sounds"
os.makedirs(OUTPUT_DIR, exist_ok=True)

for name, text in SCRIPTS.items():
    mp3_path = f"{OUTPUT_DIR}/{name}.mp3"
    wav_path = f"{OUTPUT_DIR}/{name}.wav"

    print(f"[TTS] {name}...")
    tts = gTTS(text=text, lang="vi", slow=False)
    tts.save(mp3_path)

    # Convert sang format Asterisk: 8000 Hz, mono, signed 16-bit PCM
    subprocess.run([
        "sox", mp3_path,
        "-r", "8000", "-c", "1", "-b", "16",
        wav_path
    ], check=True)
    os.remove(mp3_path)
    print(f"  -> {wav_path}")

print("Done. Copy *.wav vao /var/lib/asterisk/sounds/custom/")
```

## D3. Chạy sinh audio (trên máy host hoặc trong VM)

```bash
pip install gtts                          # cần Internet để gọi Google TTS
# sox đã cài cùng asterisk-mp3 ở Phần B
cd scripts && python3 gen_audio.py
```

Kết quả: 5 file `.wav` trong `sounds/`.

✅ **Kết thúc Phần D.**

---

# 🅴 PHẦN E — TRIỂN KHAI + TEST

## E1. `scripts/deploy_configs.sh` — deploy lên VM

```bash
#!/bin/bash
set -e

PROJECT_DIR="$(dirname "$(realpath "$0")")/.."
CONFIG_DIR="$PROJECT_DIR/configs"
SOUNDS_DIR="$PROJECT_DIR/sounds"

echo "[1/4] Backup config cu..."
sudo cp -r /etc/asterisk /etc/asterisk.bak.$(date +%s)

echo "[2/4] Copy file .conf..."
for f in sip.conf iax.conf extensions.conf voicemail.conf meetme.conf musiconhold.conf; do
    sudo cp "$CONFIG_DIR/$f" "/etc/asterisk/$f"
done
sudo chown asterisk:asterisk /etc/asterisk/*.conf
sudo chmod 640 /etc/asterisk/*.conf

echo "[3/4] Copy file am thanh..."
sudo mkdir -p /var/lib/asterisk/sounds/custom
sudo cp "$SOUNDS_DIR"/*.wav /var/lib/asterisk/sounds/custom/
sudo chown -R asterisk:asterisk /var/lib/asterisk/sounds/custom

echo "[4/4] Reload Asterisk..."
sudo asterisk -rx "core reload"
sudo asterisk -rx "dialplan reload"
sudo asterisk -rx "sip reload"
sudo asterisk -rx "iax2 reload"
sudo asterisk -rx "voicemail reload"
sudo asterisk -rx "meetme reload" 2>/dev/null || true

echo "Deploy xong. Kiem tra: sudo asterisk -rvvv"
```

Copy thư mục dự án lên VM (từ Windows):
```powershell
scp -r voip-project-group08 ubuntu@192.168.1.123:~/
```

Hoặc làm trực tiếp trên VM bằng SSH.

Chạy deploy:
```bash
ssh ubuntu@192.168.1.123
cd ~/voip-project-group08
chmod +x scripts/*.sh
bash scripts/deploy_configs.sh
```

## E2. Đăng ký softphone 🛑 USER

### Trên PC (Windows) — cài MicroSIP + Linphone

**MicroSIP** (ext 6086 SIP):
- Add Account → SIP
- SIP Server: `192.168.1.123`
- Username: `6086`
- Password: `Pass6086!`
- Display name: `HR Staff`

**Linphone** (ext 5085 IAX2):
- Settings → Accounts → Add
- Username: `5085`
- Password: `Pass5085!`
- Domain: `192.168.1.123:4569`
- Transport: IAX2

### Trên điện thoại Android — cài Zoiper 5

Đăng ký 1 trong 2:
- Ext 8080 (Sales) — để test ring sales
- Ext `external-trunk` (giả lập số ngoài) — để test IVR gọi vào

Username/Password tương ứng.

## E3. Test 15 kịch bản

```
[1]  SIP→SIP        : 6086 gọi 8080  -> đổ chuông, nghe được
[2]  SIP→IAX        : 6086 gọi 5085  -> đổ chuông
[3]  IAX→SIP        : 5085 gọi 7082  -> đổ chuông
[4]  IAX→IAX        : 5085 gọi 7081  -> đổ chuông
[5]  Voicemail      : 6086 gọi 8088, không bấm 20s -> vào VM
[6]  Check VM       : Từ 8088 gọi 500 -> PIN 1234 -> nghe VM
[7]  Conference     : 3 máy gọi 4084, PIN 654321 -> nghe nhau
[8]  Gọi ra ngoài   : từ 6086 quay 9-0951000008 -> rung external softphone
[9]  IVR phím 1     : external gọi 0952014308 -> welcome -> bấm 1 -> ring sales -> nghe
[10] IVR phím 2     : ... bấm 2 -> ring 7081 rồi 7082
[11] IVR phím 3     : ... bấm 3 -> ring 6086
[12] IVR phím 4     : ... bấm 4 -> cảm ơn -> để lại VM -> check ở 5085
[13] IVR phím 5     : ... bấm 5 -> nghe lại welcome
[14] IVR timeout    : không bấm 10s -> hangup
[15] Invalid digit  : bấm 9 -> "invalid" -> quay lại IVR
```

**Kiểm tra trên CLI:**
```bash
sudo asterisk -rvvv
asterisk*CLI> sip show peers
asterisk*CLI> iax2 show peers
asterisk*CLI> dialplan show internal
asterisk*CLI> dialplan show ivr-main
asterisk*CLI> voicemail show users
asterisk*CLI> meetme list
```

→ Chụp screenshot cho báo cáo.

✅ **Kết thúc Phần E.** 🛑 USER quay video demo.

---

# 🅵 PHẦN F — BÁO CÁO + NỘP BÀI

## F1. Cấu trúc thư mục nộp

```
Nhom08_VoIP/
├── BaoCao_Nhom08.pdf
├── configs/
│   ├── sip.conf
│   ├── iax.conf
│   ├── extensions.conf
│   ├── voicemail.conf
│   ├── meetme.conf
│   └── musiconhold.conf
├── sounds/
│   ├── welcome.wav
│   ├── sales-welcome.wav
│   ├── sales-busy.wav
│   ├── tech-busy.wav
│   └── feedback-thanks.wav
├── scripts/
│   ├── install_asterisk.sh
│   ├── deploy_configs.sh
│   └── gen_audio.py
├── screenshots/
│   └── (ảnh chụp test)
├── demo.mp4
└── README.md
```

## F2. `report/checklist.md`

| STT | Chức năng | Trạng thái | Ghi chú |
|----:|-----------|:----------:|---------|
| 1 | Tạo, quản lý các số nội bộ | ✅ | 7 ext (4 SIP + 3 IAX2) |
| 2 | Liên lạc giữa các số nội bộ | ✅ | SIP↔SIP, SIP↔IAX, IAX↔IAX |
| 3 | Họp nội bộ (room conference) | ✅ | Ext 4084, user 654321 / admin 123456 |
| 4 | Gọi ra ngoài (prefix 9) | ✅ | Pattern `_9.` |
| 5 | IVR chào mừng khi gọi vào | ✅ | welcome.wav |
| 6 | Phím 1 → Bán hàng | ✅ | Ring all 3 ext, VM nếu busy |
| 7 | Phím 2 → Kỹ thuật | ✅ | Ring tuần tự 7081 → 7082 |
| 8 | Phím 3 → Nhân sự | ✅ | Dial 6086 |
| 9 | Phím 4 → Thông điệp cảm ơn | ✅ | feedback-thanks.wav |
| 10 | Ghi VM vào hộp thư GĐ | ✅ | VoiceMail(5085@default,s) |
| 11 | Nghe VM khi gọi 500 | ✅ | VoiceMailMain() |
| 12 | Phím 5 → phát lại lời chào | ✅ | Goto(ivr-main,s,1) |
| 13 | Chức năng khác: MoH, CallerID | ✅ | Nhạc chờ + caller ID |

## F3. `report/report.md` — convert sang PDF khi nộp

```markdown
# BÁO CÁO PROJECT 2 — CẤU HÌNH TỔNG ĐÀI THOẠI VOIP

**Môn:** Công nghệ Truyền thông Đa phương tiện
**GVHD:** ThS. Đỗ Thị Hương Lan
**Nhóm:** 08

## I. Thành viên & phân công
| MSSV | Họ tên | Công việc |
|------|--------|-----------|
| ... | ... | Cài VM + Asterisk, viết sip/iax.conf |
| ... | ... | Viết extensions.conf, IVR, voicemail |
| ... | ... | Sinh audio, test, quay video demo |
| ... | ... | Viết báo cáo, slide, presentation |

## II. Kiến trúc hệ thống

```
        ┌─────────────────────────────────┐
        │  Asterisk PBX (Ubuntu 24.04)    │
        │  IP: 192.168.1.123              │
        │  - chan_sip on UDP 5060         │
        │  - chan_iax2 on UDP 4569        │
        │  - MeetMe room 4084             │
        │  - 7 extensions                 │
        │  - 5 audio files (vi-VN)        │
        └────┬────────────────────────┬───┘
             │                        │
       ┌─────┴─────┐           ┌──────┴──────┐
       │ Softphone │           │  Softphone  │
       │ MicroSIP  │           │   Zoiper    │
       │  6086     │           │ 8080(ngoài) │
       └───────────┘           └─────────────┘
```

- Server: Ubuntu 24.04 LTS + Asterisk 20 LTS
- 7 extension nội bộ (4 SIP + 3 IAX2)
- 1 phòng họp MeetMe ext 4084
- IVR 5 lựa chọn cho cuộc gọi vào số public 0952014308

## III. Cấu hình chi tiết
### 3.1. SIP (sip.conf) — paste nội dung + giải thích từng block
### 3.2. IAX2 (iax.conf) — tương tự
### 3.3. Dialplan (extensions.conf) — vẽ flowchart IVR
### 3.4. Voicemail / MeetMe
### 3.5. Audio files — bảng kèm script lời thoại

## IV. Sơ đồ luồng IVR

```
Caller (0951000008) → 0952014308
   ↓
[welcome.wav]
   ├─ 1 → ivr-sales   → ring 8080+8086+8088 đồng loạt → VM nếu busy
   ├─ 2 → ivr-tech    → ring 7081 → 7082 tuần tự → "đều bận"
   ├─ 3 → ivr-hr      → ring 6086 → VM
   ├─ 4 → ivr-feedback → "Cảm ơn" → VM của 5085
   └─ 5 → quay lại ivr-main
```

## V. Kết quả test
[Chèn screenshot cho 15 test case]

## VI. Bảng chức năng hoàn thành
[Copy từ checklist.md]

## VII. Khó khăn & giải pháp
- MeetMe cần DAHDI timing → cài `dahdi-dkms`, modprobe `dahdi_dummy`
- chan_sip deprecated trong Asterisk 20 → ép load trong modules.conf, disable chan_pjsip
- NAT làm âm thanh 1 chiều → `nat=force_rport,comedia` + mở RTP 10000-20000
- TTS gTTS xuất MP3 → resample về 8kHz mono PCM bằng `sox`

## VIII. Tài liệu tham khảo
- https://docs.asterisk.org/
- https://wiki.asterisk.org/wiki/display/AST/Dialplan+Applications
- https://ubuntu.com/server/docs

```

## F4. Convert MD → PDF + đóng gói

Trên máy host (cài pandoc):
```bash
sudo apt install -y pandoc texlive-xetex
pandoc report/report.md -o BaoCao_Nhom08.pdf --pdf-engine=xelatex \
       -V mainfont="Times New Roman" -V geometry:margin=2cm

# Zip nộp bài
zip -r Nhom08_VoIP.zip \
    BaoCao_Nhom08.pdf configs/ sounds/ scripts/ screenshots/ demo.mp4 README.md
```

✅ **Kết thúc Phần F.**

---

# 🅶 PHẦN G — TROUBLESHOOTING

## G1. Softphone không register
```bash
sudo asterisk -rx "sip set debug on"
sudo asterisk -rx "sip show peers"
# Status phải OK, không phải UNREACHABLE
```
- Firewall: `sudo ufw status` → mở 5060/udp
- Sai password trong sip.conf
- Trên Zoiper bật "Use STUN" → `stun.l.google.com:19302`

## G2. Gọi được nhưng không nghe (one-way audio)
- Mở RTP: `sudo ufw allow 10000:20000/udp`
- Codec mismatch → ép `ulaw`/`alaw` trong sip.conf
- Nếu VM phía sau NAT (ít gặp với Bridged): thêm vào sip.conf
```ini
[general]
externaddr=<PUBLIC_IP>
localnet=192.168.0.0/16
```

## G3. MeetMe lỗi "no timing source"
```bash
sudo modprobe dahdi_dummy
lsmod | grep dahdi
# Vẫn lỗi? Đảm bảo dahdi-dkms đã build cho kernel hiện tại:
sudo dpkg-reconfigure dahdi-dkms
```

## G4. Audio không phát ("file not found")
```bash
ls -la /var/lib/asterisk/sounds/custom/
# File phải là 8000Hz mono PCM 16-bit:
file /var/lib/asterisk/sounds/custom/welcome.wav
# kỳ vọng: "RIFF (little-endian) data, WAVE audio, ... 8000 Hz"
sudo chown -R asterisk:asterisk /var/lib/asterisk/sounds/
```

## G5. IVR không nhận DTMF
- sip.conf đã có `dtmfmode=rfc2833` ✅
- Trên Zoiper: Account → Advanced → DTMF = RFC2833
- Debug: `sudo asterisk -rx "core set verbose 5"` → quan sát "DTMF Received"

## G6. chan_sip không load trong Asterisk 20
```bash
sudo asterisk -rx "module load chan_sip.so"
# Nếu báo deprecated nhưng vẫn load được -> OK
# Nếu module không tìm thấy:
sudo apt install -y asterisk-modules
sudo systemctl restart asterisk
```

## G7. Lỗi "no route to host" từ softphone
- Sai IP server trong cấu hình softphone
- VM dùng NAT thay vì Bridged → tạo lại VM với Bridged
- Trên VM: `sudo ufw status` xem có chặn không

---

# 🅷 PHẦN H — CHECKLIST CHO AI AGENT

Agent thực hiện tuần tự, dừng tại 🛑:

### Phase Setup (agent + user)
- [ ] **🛑 USER:** Cài VMware Workstation Pro 25H2
- [ ] **🛑 USER:** Tải Ubuntu 24.04 Server ISO
- [ ] **🛑 USER:** Tạo VM với RAM 2GB, 2 CPU, disk 20GB, network Bridged
- [ ] **🛑 USER:** Cài Ubuntu Server (Easy Install)
- [ ] **🛑 USER:** Login VM, lấy IP, đặt IP tĩnh, mở firewall
- [ ] **🛑 USER:** SSH từ Windows vào VM
- [ ] **🛑 USER:** Snapshot VM trạng thái sạch

### Phase Asterisk (qua SSH)
- [ ] **🛑 USER:** `apt install asterisk + dahdi + sox` (script Phần B)
- [ ] **🛑 USER:** Modprobe dahdi_dummy, enable asterisk service
- [ ] **🛑 USER:** Sửa modules.conf để dùng chan_sip thay chan_pjsip

### Phase Configs (AGENT làm trên máy host)
- [ ] **Task 1:** Tạo cây thư mục `voip-project-group08/`
- [ ] **Task 2:** Ghi 6 file `.conf` vào `configs/` (Phần C)
- [ ] **Task 3:** Ghi `gen_audio.py` + `deploy_configs.sh` vào `scripts/`
- [ ] **Task 4:** Ghi `sounds/script.txt`
- [ ] **Task 5:** Chạy `python3 gen_audio.py` → sinh 5 file `.wav`
- [ ] **Task 6:** Ghi `report/checklist.md` và `report/report.md`
- [ ] **Task 7:** Ghi `README.md` ở root dự án

### Phase Deploy + Test
- [ ] **🛑 USER:** `scp` thư mục lên VM
- [ ] **🛑 USER:** Chạy `bash scripts/deploy_configs.sh`
- [ ] **🛑 USER:** Đăng ký softphone (MicroSIP + Linphone + Zoiper)
- [ ] **🛑 USER:** Chạy 15 test case, chụp screenshot
- [ ] **🛑 USER:** Quay video demo

### Phase Báo cáo
- [ ] **Task 8:** Chèn screenshot vào `report.md`, thêm Troubleshooting nếu có lỗi
- [ ] **Task 9:** Convert `report.md` → `BaoCao_Nhom08.pdf` bằng pandoc
- [ ] **Task 10:** Zip toàn bộ thành `Nhom08_VoIP.zip`

---

## Đổi GROUP_NUMBER (nếu nhóm khác 08)

```bash
GN=03   # đổi sang số nhóm của bạn
find voip-project-group08 -type f \( -name "*.conf" -o -name "*.md" -o -name "*.sh" -o -name "*.py" \) \
  -exec sed -i "s/5085/5${GN}5/g; s/6086/6${GN}6/g; s/7081/7${GN}1/g; s/7082/7${GN}2/g; \
                s/8080/8${GN}0/g; s/8086/8${GN}6/g; s/8088/8${GN}8/g; \
                s/4084/4${GN}4/g; s/0951000008/095100000${GN}/g; \
                s/0952014308/095201430${GN}/g; s/group08/group${GN}/g; \
                s/Nhom 08/Nhom ${GN}/g; s/Nhom08/Nhom${GN}/g" {} +
mv voip-project-group08 voip-project-group${GN}
```

---

**HẾT BLUEPRINT.**

Agent đọc xong file này có đầy đủ thông tin để:
1. Hướng dẫn user dựng môi trường (Phần A → B)
2. Tự sinh toàn bộ file cấu hình + audio + script (Phần C → D → Task 1–7)
3. Hướng dẫn user deploy & test (Phần E)
4. Tự soạn báo cáo + đóng gói nộp bài (Phần F + Task 8–10)
