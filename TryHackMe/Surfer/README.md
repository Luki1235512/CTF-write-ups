# [Surfer](https://tryhackme.com/room/surfer)

## Surf some internal webpages to find the flag!

Woah, check out this radical app! Isn't it narly dude? We've been surfing through some webpages and we want to get you on board too! They said this application has some functionality that is only available for internal usage -- but if you catch the right wave, you can probably find the sweet stuff!

### Uncover the flag on the hidden application page.

1. Start with directory enumeration on the target to discover hidden paths:

```bash
gobuster dir -u http://<TARGET_IP>  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/assets               (Status: 301) [Size: 317] [--> http://<TARGET_IP>/assets/]
/vendor               (Status: 301) [Size: 317] [--> http://<TARGET_IP>/vendor/]
/backup               (Status: 301) [Size: 317] [--> http://<TARGET_IP>/backup/]
/internal             (Status: 301) [Size: 319] [--> http://<TARGET_IP>/internal/]
/server-status        (Status: 403) [Size: 279]
```

2. Enumerate the `/internal` directory further, this time also searching for PHP files:

```bash
gobuster dir -u http://<TARGET_IP>/internal -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -t 50
```

**Results:**

```
/admin.php            (Status: 200) [Size: 39]
```

3. Enumerate the `/backup` directory for text files that may contain useful information:

```bash
gobuster dir -u http://<TARGET_IP>/backup  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -t 50
```

**Results:**

```
/chat.txt             (Status: 200) [Size: 365]
```

4. Visit `http://<TARGET_IP>/backup/chat.txt` to read the conversation log:

```
Admin: I have finished setting up the new export2pdf tool.
Kate: Thanks, we will require daily system reports in pdf form
Admin: Yes, I am updated about that.
Kate: Have you finished adding the internal server.
Admin: Yes, it should be serving flag from now.
Kate: Also Don't forget to change the creds, plz stop using your username as password.
Kate: Hello.. ?
```

5. Log in to `http://<TARGET_IP>/login.php` with `admin:admin` credentials.

6. After logging in, scroll to the bottom of the dashboard page and click the **Export to PDF** button. This redirects to `http://<TARGET_IP>/export2pdf.php` and generates a PDF of the current page by fetching a URL server-side.

7. Intercept the **Generate PDF** POST request using Burp Suite. Change the `url` parameter value to point to the internal admin page, exploiting the SSRF vulnerability.

[SCREEN01]

This causes the server to fetch `http://127.0.0.1/internal/admin.php` on your behalf, bypassing the restriction that only allows access from localhost.

8. Forward the modified request. The server fetches the internal admin page and renders its content into a PDF, which is returned in the response. Open the PDF to find the flag.

[SCREEN02]
