## Hashcat Setup and Notes

### Installed on: **Main Windows PC**

**Note:** Do not run on the virtual machine, use your main machine for speed.

---

### Basic Hashcat Syntax

```bash
hashcat.exe -m [hash mode] -a [attack mode] -o cracked.txt hashes.txt wordlist.txt
```

**Example – MD5 with rockyou:**

```bash
hashcat.exe -m 0 -a 0 -o cracked.txt hashes.txt rockyou.txt
```

- `m 0` = MD5
- `a 0` = Dictionary attack
- `hashes.txt` = File containing hashes
- `rockyou.txt` = Wordlist
- `cracked.txt` = Output file

---

### Attack Modes

| Attack Mode | Code | Description |
| --- | --- | --- |
| Straight | 0 | Wordlist (dictionary) attack |
| Combination | 1 | Wordlist + Wordlist |
| Brute-force | 3 | Mask attack (custom patterns) |
| Hybrid 1 | 6 | Wordlist + Mask |
| Hybrid 2 | 7 | Mask + Wordlist |

---

### Hash Identifiers

Use a hash identifier **or** know where the hash came from. Then:

1. Go to [Hashcat Hash Modes](https://hashcat.net/wiki/doku.php?id=hashcat#hash_modes)
2. Or use [Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)
3. Use the correct mode with Hashcat

**Hash Identification Tools:**

- https://hashes.com/en/tools/hash_identifier
- https://www.tunnelsup.com/hash-analyzer/
- https://www.onlinehashcrack.com/hash-identification.php

**SO BASICALLY:**
Use a hash identifier, get the mode number, and run Hashcat on your Windows PC.

---

## Hashes Cracked

### 1. `48bb6e862e54f2a795ffc4e541caed4d`

- Type: MD5 (`m 0`)
- Method: CrackStation
- Password: `[REDACTED]`

---

### 2. `CBFDAC6008F9CAB4083784CBD1874F76618D2A97`

- Type: SHA1 (`m 100`)
- Method: CrackStation
- Password: `[REDACTED]`

---

### 3. `1C8BFE8F801D79745C4631D09FFF36C82AA37FC4CCE4FC946683D7B336B63032`

- Type: SHA256 (`m 1400`)
- Method: CrackStation
- Password: `[REDACTED]`

---

### 4. `$2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom`

- Type: Bcrypt (`m 3200`)
- Method: CrackStation was **wrong**, had to use hash identifier
- Attack: Mask Attack (`a 3`) for four lowercase characters
- Command:
    
    ```bash
    hashcat.exe -m 3200 -a 3 hashes.txt ?l?l?l?l
    ```
    
- Password: `[REDACTED]`

---

### 5. `279412f945939ba78ce0758d3fd83daa`

- Type: MD4 (`m 900`)
- Password in rockyou
- Password: `[REDACTED]`
---

### 6. `F09EDCB1FCEFC6DFB23DC3505A882655FF77375ED8AA2D1C13F640FCCC2D0C85`

- Type: SHA256 (`m 1400`)
- Method: Hashcat
- Password: `[REDACTED]`

---

### 7. `1DFECA0C002AE40B8619ECF94819CC1B`

- Type: NTLM (`m 1000`)
- Method: Hashcat
- Password: `[REDACTED]`

---

### 8. `$6$aReallyHardSalt$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqM`

- Type: SHA512crypt (`m 1800`)
- Format includes the salt at the beginning of the hash
- Password: `[REDACTED]`

---

### 9. `e5d8870e5bdd26602cab8dbe07a942c8669e56d6`

- Type: SHA1 with Salt (`m 120`)
- Salt: `tryhackme`
- Format for Hashcat:
    
    ```
    e5d8870e5bdd26602cab8dbe07a942c8669e56d6:tryhackme
    ```
- Password: `[REDACTED]`
