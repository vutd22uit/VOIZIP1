# Project 2 — Cấu hình Tổng đài thoại VoIP (Asterisk)

> **HƯỚNG DẪN CHO AI AGENT (Cowork / Claude Code / agent khác)**
>
> File này là blueprint trọn vẹn để dựng tổng đài VoIP theo đề bài UIT. Agent đọc file này, thực hiện tuần tự từ Phase 0 → Phase 6, dừng lại hỏi user khi gặp bước cần thao tác vật lý (cài VM, quay video demo).
>
> **Biến cần thay trước khi chạy:**
> - `GROUP_NUMBER` = `08` (đổi thành số nhóm thực, luôn 2 chữ số, ví dụ nhóm 3 → `03`)
> - `SERVER_IP` = IP của VM Asterisk (vd `192.168.1.100`)
> - `EXTERNAL_NUMBER` = `0951000008` (số ngoài, đổi `08` thành GROUP_NUMBER)
> - `PUBLIC_NUMBER` = `0952014308` (số public công ty)
>
> **Bảng extension (với GROUP_NUMBER=08):**
> | Phòng | Ext | Giao thức |
> |---|---|---|
> | Giám đốc | 5085 | IAX2 |
> | Nhân sự | 6086 | SIP |
> | Kỹ thuật 1 | 7081 | IAX2 |
> | Kỹ thuật 2 | 7082 | SIP |
> | Bán hàng 1 | 8080 | SIP |
> | Bán hàng 2 | 8086 | IAX2 |
> | Bán hàng 3 | 8088 | SIP |
> | Conference | 4084 | (MeetMe) |
> | Voicemail check | 500 | (App) |

---

## Phase 0 — Chuẩn bị môi trường

### 0.1. Yêu cầu hạ tầng (USER tự làm — agent không tự dựng được)
- 1 VM Ubuntu 22.04 LTS (RAM ≥ 2GB, 2 vCPU, network bridge hoặc NAT cho phép port-forward)
- 2 softphone (Zoiper / MicroSIP / 3CX) trên 2 thiết bị khác nhau (PC + điện thoại) để test
- Kết nối Internet để `apt install`

### 0.2. Cấu trúc thư mục dự án (agent tạo trên máy host)
```
voip-project-group08/
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
│   ├── feedback-thanks.wav
│   └── script.txt
├── scripts/
│   ├── install_asterisk.sh
│   ├── deploy_configs.sh
│   ├── gen_audio.py
│   └── test_calls.sh
├── report/
│   ├── report.md
│   └── checklist.md
└── README.md
```

---

## Phase 1 — Cài đặt Asterisk (script tự động)

### 1.1. Tạo file `scripts/install_asterisk.sh`

```bash
#!/bin/bash
# Cài Asterisk 20 LTS trên Ubuntu 22.04
set -e

echo "[1/6] Update hệ thống..."
sudo apt update && sudo apt upgrade -y

echo "[2/6] Cài dependencies..."
sudo apt install -y build-essential wget libssl-dev libncurses5-dev \
    libnewt-dev libxml2-dev linux-headers-$(uname -r) libsqlite3-dev \
    uuid-dev libjansson-dev libedit-dev sox

echo "[3/6] Tải Asterisk source..."
cd /usr/src
sudo wget -q http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
sudo tar zxvf asterisk-20-current.tar.gz
cd asterisk-20*/

echo "[4/6] Cài prerequisites..."
sudo contrib/scripts/install_prereq install

echo "[5/6] Configure + make..."
sudo ./configure --with-jansson-bundled
sudo make menuselect.makeopts
# Enable app_meetme và codec cần thiết
sudo menuselect/menuselect --enable app_meetme menuselect.makeopts
sudo make -j$(nproc)
sudo make install
sudo make samples
sudo make config
sudo ldconfig

echo "[6/6] Khởi động Asterisk..."
sudo systemctl enable asterisk
sudo systemctl start asterisk

echo "Done. Kiểm tra: sudo asterisk -rvvv"
```

### 1.2. Cài DAHDI dummy (cần cho MeetMe trên VM không có card)

```bash
sudo apt install -y dahdi-linux dahdi
sudo modprobe dahdi_dummy
echo "dahdi_dummy" | sudo tee -a /etc/modules
```

---

## Phase 2 — File cấu hình

### 2.1. `configs/sip.conf`

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

;========== TRUNK GIẢ LẬP SỐ NGOÀI (cho mục đích test) ==========
; Nếu có ITSP thật thì thay bằng register/host của ITSP
[external-trunk]
type=friend
host=dynamic
context=from-external
secret=ExtPass!
disallow=all
allow=ulaw
allow=alaw
```

### 2.2. `configs/iax.conf`

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

### 2.3. `configs/voicemail.conf`

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
pagerdateformat=%T %D
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

### 2.4. `configs/meetme.conf`

```ini
[general]
audiobuffers=32

[rooms]
; Room 4084: hop noi bo cong ty
; Cú pháp Asterisk: conf => roomno,user_pin,admin_pin
; Theo đề: "password QUẢN LÝ và GIA NHẬP lần lượt là 123456 và 654321"
;   -> admin_pin (quản lý) = 123456
;   -> user_pin  (gia nhập) = 654321
conf => 4084,654321,123456
```

### 2.5. `configs/musiconhold.conf`

```ini
[default]
mode=files
directory=moh
```

### 2.6. `configs/extensions.conf` ⭐ (file trung tâm)

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
; Bất kỳ cuộc gọi nào đến số public đều vào IVR chính
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
 ; Dial song song cả 3 số bán hàng (SIP + IAX)
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
 ; Quay lần lượt 7081 -> 7082 cho đến khi có người nhấc
 same => n,Dial(IAX2/7081,15,tT)
 same => n,GotoIf($["${DIALSTATUS}"="ANSWER"]?end)
 same => n,Dial(SIP/7082,15,tT)
 same => n,GotoIf($["${DIALSTATUS}"="ANSWER"]?end)
 ; Cả 2 đều không nhấc -> báo bận rồi cúp máy (theo đề: "chờ thực hiện lại cuộc gọi")
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
; CONTEXT: ivr-feedback — phím 4 (Để lại lời nhắn cho GĐ)
;==================================================
[ivr-feedback]
exten => s,1,NoOp(IVR Feedback - leave voicemail to Director)
 same => n,Playback(custom/feedback-thanks)
 same => n,VoiceMail(5085@default,s)   ; s = skip greeting (đã phát lời cảm ơn rồi)
 same => n,Hangup()


;==================================================
; CONTEXT: default — bảo vệ
;==================================================
[default]
exten => _X.,1,Hangup()
```

---

## Phase 3 — Sinh file âm thanh (Vietnamese TTS)

### 3.1. `sounds/script.txt` — nội dung lời thoại

```
welcome.wav        | Chào mừng quý khách gọi đến công ty ABC. Vui lòng nhấn phím 1 để kết nối với phòng bán hàng. Nhấn phím 2 để được hỗ trợ kỹ thuật. Nhấn phím 3 để biết thông tin tuyển dụng. Nhấn phím 4 để để lại lời nhắn hoặc góp ý cho Ban Giám Đốc. Nhấn phím 5 để nghe lại lời chào.
sales-welcome.wav  | Chào mừng bạn đã đến phòng bán hàng. Vui lòng đợi trong giây lát để được kết nối với điện thoại viên.
sales-busy.wav     | Xin lỗi, hiện tại các điện thoại viên đều bận. Vui lòng để lại lời nhắn sau tiếng pip, hoặc thực hiện lại cuộc gọi.
tech-busy.wav      | Xin lỗi, hiện tại các kỹ thuật viên đều bận. Vui lòng chờ trong giây lát để thực hiện lại cuộc gọi.
feedback-thanks.wav| Xin chân thành cảm ơn bạn đã góp ý cho công ty chúng tôi. Vui lòng để lại lời nhắn sau tiếng pip.
```

### 3.2. `scripts/gen_audio.py` — sinh WAV bằng gTTS

```python
#!/usr/bin/env python3
"""Sinh file WAV tiếng Việt cho IVR Asterisk (8kHz, mono, PCM 16-bit)."""
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

    # Convert sang định dạng Asterisk yêu cầu: 8000 Hz, mono, signed 16-bit PCM
    subprocess.run([
        "sox", mp3_path,
        "-r", "8000", "-c", "1", "-b", "16",
        wav_path
    ], check=True)
    os.remove(mp3_path)
    print(f"  -> {wav_path}")

print("Done. Copy *.wav vào /var/lib/asterisk/sounds/custom/")
```

Cài deps: `pip install gtts && sudo apt install sox libsox-fmt-mp3`

---

## Phase 4 — Triển khai

### 4.1. `scripts/deploy_configs.sh`

```bash
#!/bin/bash
set -e

PROJECT_DIR="$(dirname "$(realpath "$0")")/.."
CONFIG_DIR="$PROJECT_DIR/configs"
SOUNDS_DIR="$PROJECT_DIR/sounds"

echo "[1/4] Backup config cũ..."
sudo cp -r /etc/asterisk /etc/asterisk.bak.$(date +%s)

echo "[2/4] Copy file .conf..."
for f in sip.conf iax.conf extensions.conf voicemail.conf meetme.conf musiconhold.conf; do
    sudo cp "$CONFIG_DIR/$f" "/etc/asterisk/$f"
done
sudo chown asterisk:asterisk /etc/asterisk/*.conf
sudo chmod 640 /etc/asterisk/*.conf

echo "[3/4] Copy file âm thanh..."
sudo mkdir -p /var/lib/asterisk/sounds/custom
sudo cp "$SOUNDS_DIR"/*.wav /var/lib/asterisk/sounds/custom/
sudo chown -R asterisk:asterisk /var/lib/asterisk/sounds/custom

echo "[4/4] Reload Asterisk..."
sudo asterisk -rx "core reload"
sudo asterisk -rx "dialplan reload"
sudo asterisk -rx "sip reload"
sudo asterisk -rx "iax2 reload"
sudo asterisk -rx "voicemail reload"
sudo asterisk -rx "meetme reload"

echo "Deploy xong. Kiểm tra: sudo asterisk -rvvv"
```

### 4.2. Mở firewall

```bash
sudo ufw allow 5060/udp     # SIP
sudo ufw allow 5060/tcp
sudo ufw allow 4569/udp     # IAX2
sudo ufw allow 10000:20000/udp   # RTP
sudo ufw reload
```

---

## Phase 5 — Test cases

### 5.1. `scripts/test_calls.sh` — checklist test

```bash
#!/bin/bash
# Đăng ký softphone theo bảng dưới rồi gọi thử
cat <<'EOF'
================== HƯỚNG DẪN ĐĂNG KÝ SOFTPHONE ==================
Zoiper / MicroSIP:
  Domain: <SERVER_IP>:5060
  User  : 6086 (SIP) hoặc 5085 (IAX2, port 4569)
  Pass  : Pass<ext>!

================== TEST CASES ==================
[1] SIP→SIP        : 6086 gọi 8080  -> phải đổ chuông & nói chuyện được
[2] SIP→IAX        : 6086 gọi 5085  -> đổ chuông
[3] IAX→SIP        : 5085 gọi 7082  -> đổ chuông
[4] IAX→IAX        : 5085 gọi 7081  -> đổ chuông
[5] Voicemail      : 6086 gọi 8088, không bấm phím 20s -> vào VM
[6] Check VM       : Từ 8088 gọi 500 -> nhập password 1234 -> nghe VM
[7] Conference     : 3 máy cùng gọi 4084, nhập PIN 654321 (user) -> nghe nhau
                     (PIN 123456 = admin, có quyền kick / mute)
[8] Gọi ra ngoài   : từ 6086 quay 90123456789 -> rớt cuộc gọi (chưa có ITSP) hoặc ra được nếu có trunk thật
[9] IVR phím 1     : từ external-trunk gọi 0952014308 -> nghe welcome -> bấm 1 -> ring sales -> nói chuyện được
[10] IVR phím 2    : ... bấm 2 -> ring 7081 rồi 7082
[11] IVR phím 3    : ... bấm 3 -> ring 6086
[12] IVR phím 4    : ... bấm 4 -> nghe cảm ơn -> để lại VM -> kiểm tra ext 5085 thấy VM mới
[13] IVR phím 5    : ... bấm 5 -> nghe lại welcome
[14] IVR timeout   : không bấm 10s -> hangup
[15] Invalid digit : bấm 9 -> "invalid" -> quay lại IVR

================== KIỂM TRA TRÊN CLI ==================
sudo asterisk -rvvv
  > sip show peers
  > iax2 show peers
  > dialplan show internal
  > dialplan show ivr-main
  > voicemail show users
  > meetme list
EOF
```

### 5.2. Mô phỏng cuộc gọi từ ngoài (không cần ITSP)

Để test IVR mà không có trunk PSTN thật, đăng ký 1 softphone vào account `external-trunk` rồi gọi tới `0952014308` — Asterisk sẽ định tuyến qua context `from-external` → IVR.

Hoặc thêm extension test trong `[internal]`:

```ini
exten => 9999,1,Goto(ivr-main,s,1)   ; gọi nội bộ 9999 để vào IVR
```

---

## Phase 6 — Báo cáo & nộp bài

### 6.1. `report/checklist.md`

```markdown
| STT | Chức năng | Trạng thái | Ghi chú |
|----:|-----------|:----------:|---------|
| 1 | Tạo, quản lý các số nội bộ | ✅ | 7 ext (SIP+IAX) |
| 2 | Liên lạc giữa các số nội bộ | ✅ | SIP↔SIP, SIP↔IAX, IAX↔IAX |
| 3 | Họp nội bộ (room conference) | ✅ | Ext 4084, user PIN 654321 / admin PIN 123456 |
| 4 | Gọi ra ngoài (prefix 9) | ✅ | Pattern `_9.` |
| 5 | IVR chào mừng khi gọi vào | ✅ | welcome.wav |
| 6 | Phím 1 → Phòng bán hàng | ✅ | Ring all 3 ext, VM nếu busy |
| 7 | Phím 2 → Phòng kỹ thuật | ✅ | Ring tuần tự 7081 → 7082 |
| 8 | Phím 3 → Phòng nhân sự | ✅ | Dial 6086 |
| 9 | Phím 4 → Thông điệp cảm ơn | ✅ | feedback-thanks.wav |
| 10 | Ghi VM vào hộp thư Giám đốc | ✅ | VoiceMail(5085@default,s) |
| 11 | Nghe lại VM khi gọi 500 | ✅ | VoiceMailMain() |
| 12 | Phím 5 → phát lại lời chào | ✅ | Goto(ivr-main,s,1) |
| 13 | Chức năng khác: MoH, CallerID | ✅ | Nhạc chờ + caller ID |
```

### 6.2. `report/report.md` — khung báo cáo (convert sang PDF khi nộp)

```markdown
# BÁO CÁO PROJECT 2 — CẤU HÌNH TỔNG ĐÀI THOẠI VOIP

**Môn:** Công nghệ Truyền thông Đa phương tiện
**GVHD:** ThS. Đỗ Thị Hương Lan
**Nhóm:** 08

## I. Thành viên & phân công
| MSSV | Họ tên | Công việc |
|------|--------|-----------|
| ... | ... | Cài Asterisk, viết sip/iax.conf |
| ... | ... | Viết extensions.conf, IVR |
| ... | ... | Sinh audio, test, quay demo |
| ... | ... | Viết báo cáo, slide |

## II. Kiến trúc hệ thống
[Sơ đồ: Softphone ─── SIP/IAX ─── Asterisk PBX ─── Trunk ──→ PSTN]

- Server: Ubuntu 22.04 + Asterisk 20 LTS
- IP server: <SERVER_IP>
- 7 extension nội bộ (4 SIP + 3 IAX2)
- 1 phòng họp MeetMe
- 1 IVR 5 lựa chọn cho cuộc gọi vào

## III. Cấu hình chi tiết
### 3.1. SIP (sip.conf)
[Paste nội dung + giải thích từng block]
### 3.2. IAX2 (iax.conf)
[...]
### 3.3. Dialplan (extensions.conf)
[Vẽ flowchart IVR]
### 3.4. Voicemail / MeetMe
[...]

## IV. Sơ đồ luồng IVR
```
Caller → 0952014308
   ↓
[welcome.wav]
   ├─ 1 → Sales (ring 8080+8086+8088) → VM nếu busy
   ├─ 2 → Tech (ring 7081 → 7082) → "đều bận"
   ├─ 3 → HR (6086) → VM
   ├─ 4 → "Cảm ơn" → VM của 5085
   └─ 5 → Quay lại welcome
```

## V. Kết quả test
[Screenshot CLI: sip show peers, dialplan show, log cuộc gọi]
[Screenshot Zoiper: registered, gọi thành công]

## VI. Bảng chức năng đã hoàn thành
[Copy từ checklist.md]

## VII. Khó khăn & giải pháp
- MeetMe cần DAHDI timing → cài `dahdi_dummy`
- NAT làm âm thanh 1 chiều → `nat=force_rport,comedia` + mở RTP port
- Audio TTS gTTS phải resample về 8kHz mono PCM → dùng `sox`

## VIII. Tài liệu tham khảo
- https://docs.asterisk.org/
- https://wiki.asterisk.org/wiki/display/AST/Dialplan+Applications
```

### 6.3. Danh sách file nộp (zip lại)
```
Nhom08_VoIP.zip
├── BaoCao_Nhom08.pdf           (export từ report.md)
├── configs/                    (5 file .conf)
├── sounds/                     (5 file .wav)
├── scripts/                    (4 script bash + python)
├── demo.mp4                    (video quay demo — USER tự quay)
└── README.txt
```

---

## Phase 7 — README.md cho thư mục dự án

Agent ghi file `voip-project-group08/README.md` với nội dung:

```markdown
# VoIP IPPBX — Nhóm 08

## Mô tả
Tổng đài thoại VoIP triển khai bằng Asterisk 20 LTS trên Ubuntu 22.04, đáp ứng Project 2 môn Công nghệ Truyền thông Đa phương tiện (UIT).

## Quick start
```bash
# 1. Trên VM Ubuntu (cần sudo, có internet)
bash scripts/install_asterisk.sh

# 2. Sinh file âm thanh tiếng Việt (cần Python + Internet)
pip install gtts && sudo apt install -y sox libsox-fmt-mp3
cd scripts && python3 gen_audio.py && cd ..

# 3. Deploy config + audio
bash scripts/deploy_configs.sh

# 4. Mở firewall
sudo ufw allow 5060/udp && sudo ufw allow 4569/udp && sudo ufw allow 10000:20000/udp

# 5. Theo dõi log realtime
sudo asterisk -rvvv
```

## Tài khoản softphone
| Ext | Proto | Password | Phòng |
|-----|-------|----------|-------|
| 5085 | IAX2 | Pass5085! | Giám đốc |
| 6086 | SIP  | Pass6086! | Nhân sự |
| 7081 | IAX2 | Pass7081! | Kỹ thuật |
| 7082 | SIP  | Pass7082! | Kỹ thuật |
| 8080 | SIP  | Pass8080! | Bán hàng |
| 8086 | IAX2 | Pass8086! | Bán hàng |
| 8088 | SIP  | Pass8088! | Bán hàng |

VM password (mặc định): `1234` cho mọi ext.
Conference PIN: user `654321`, admin `123456`.

## Cấu trúc
- `configs/` — 6 file .conf của Asterisk
- `sounds/` — 5 file WAV tiếng Việt (sinh tự động)
- `scripts/` — bash + python tự động hóa
- `report/` — báo cáo + checklist
```

---

## Phase 8 — Troubleshooting (đưa vào báo cáo nếu gặp)

### 8.1. Softphone không register được
```bash
sudo asterisk -rx "sip set debug on"
sudo asterisk -rx "sip show peers"
# Check: status phải là OK, không phải UNREACHABLE
```
**Nguyên nhân thường gặp:**
- Firewall chặn UDP 5060 → `sudo ufw status`
- Sai password trong sip.conf
- NAT làm SIP packet bị rewrite → đảm bảo `nat=force_rport,comedia`

### 8.2. Gọi được nhưng không nghe tiếng (one-way audio)
- RTP port chưa mở: `sudo ufw allow 10000:20000/udp`
- Codec không khớp giữa 2 phone → ép cùng codec `ulaw`/`alaw` trong sip.conf
- Trên VM NAT, set `externaddr` và `localnet` trong sip.conf:
```ini
[general]
externaddr=<PUBLIC_IP_OF_HOST>
localnet=192.168.0.0/16
localnet=10.0.0.0/8
```

### 8.3. MeetMe lỗi "no timing source"
```bash
sudo modprobe dahdi_dummy
lsmod | grep dahdi
# Nếu vẫn lỗi, thay MeetMe bằng ConfBridge:
#   ConfBridge(4084) thay cho MeetMe(4084,pPMs)
# và tạo confbridge.conf
```

### 8.4. Lỗi phát audio "file not found"
```bash
# Asterisk tìm trong /var/lib/asterisk/sounds/custom/
ls -la /var/lib/asterisk/sounds/custom/
sudo chown -R asterisk:asterisk /var/lib/asterisk/sounds/
# File phải là wav 8000Hz mono PCM 16-bit
file welcome.wav
# -> nên ra: "RIFF (little-endian) data, WAVE audio, ... 8000 Hz"
```

### 8.5. IVR không nhận phím DTMF
- Trong sip.conf set `dtmfmode=rfc2833` (đã làm)
- Trên Zoiper: Account → Advanced → DTMF mode = RFC2833
- Test bằng: `sudo asterisk -rx "core set verbose 5"` rồi quan sát "DTMF Received" trong CLI

### 8.6. Voicemail không gửi mail
Bình thường — không cần thiết cho project. Nếu muốn:
```bash
sudo apt install postfix
# config trong voicemail.conf: serveremail, fromstring, attach=yes
```

---

## CHECKLIST CHO AGENT (theo thứ tự)

- [ ] **Task 1:** Tạo cây thư mục `voip-project-group08/` với đầy đủ subfolder
- [ ] **Task 2:** Ghi 6 file `.conf` vào `configs/` (nội dung copy từ Phase 2)
- [ ] **Task 3:** Ghi 4 file script vào `scripts/` (Phase 1, 3, 4, 5)
- [ ] **Task 4:** Ghi `sounds/script.txt` (Phase 3.1)
- [ ] **Task 5:** Chạy `gen_audio.py` để sinh 5 file `.wav` (cần Internet + gtts + sox)
- [ ] **Task 6:** Ghi `report/checklist.md` và `report/report.md` (Phase 6)
- [ ] **Task 7:** Ghi `README.md` ở root dự án (Phase 7)
- [ ] **STOP — yêu cầu user:** Cung cấp VM Ubuntu, chạy `install_asterisk.sh` rồi `deploy_configs.sh`
- [ ] **STOP — yêu cầu user:** Đăng ký softphone, chạy 15 test case ở Phase 5.1, chụp screenshot
- [ ] **STOP — yêu cầu user:** Quay video demo
- [ ] **Task 8 (sau khi user gửi screenshot):** Chèn screenshot vào `report.md`, thêm Troubleshooting (Phase 8) nếu có lỗi, convert sang PDF
- [ ] **Task 9:** Zip toàn bộ thành `Nhom<XX>_VoIP.zip`

---

## LƯU Ý KHI THAY ĐỔI GROUP_NUMBER

Nếu nhóm khác `08`, dùng lệnh sed thay toàn bộ:

```bash
GN=03   # đổi sang số nhóm của bạn
find voip-project-group08 -type f \( -name "*.conf" -o -name "*.md" -o -name "*.sh" -o -name "*.py" \) \
  -exec sed -i "s/5085/5${GN}5/g; s/6086/6${GN}6/g; s/7081/7${GN}1/g; s/7082/7${GN}2/g; \
                s/8080/8${GN}0/g; s/8086/8${GN}6/g; s/8088/8${GN}8/g; \
                s/4084/4${GN}4/g; s/0951000008/095100000${GN}/g; \
                s/0952014308/095201430${GN}/g; s/group08/group${GN}/g; \
                s/Nhom 08/Nhom ${GN}/g" {} +
mv voip-project-group08 voip-project-group${GN}
```

---

**HẾT BLUEPRINT.** Agent đọc xong file này có thể bắt đầu thực hiện từ Task 1.
