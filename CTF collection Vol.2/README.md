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

![SCREEN01](https://github.com/user-attachments/assets/36ea1f50-3477-4ca2-a75f-af88d19620b7)

2. Navigate to `http://IP/robots.txt` or `http://IP/robots`

![SCREEN02](https://github.com/user-attachments/assets/796916b9-1a32-4b66-bce4-4386a795e308)

3. Within the robots.txt file, you'll find a hexadecimal string. Copy this string to [CyberChef](https://gchq.github.io/CyberChef/) and apply the **From Hex** operation to decode it

![SCREEN03](https://github.com/user-attachments/assets/d3c4d095-e1c7-4fd1-8140-b771159a665d)

---

### Easter 2

_Decode the base64 multiple times. Don't forget there are something being encoded._

1. Copy the encoded hash from `http://IP/robots.txt`, and decode it using **Base64** four consecutive times in [CyberChef](https://gchq.github.io/CyberChef/)

![SCREEN04](https://github.com/user-attachments/assets/b434012a-22e6-4183-bcdc-22752d8828f4)

2. The decoded text points to a hidden directory `IP/DesKel_secret_base`. On this page, the flag is hidden using white text on a white background. To reveal it, either use "Select All" (Ctrl+A) or inspect the page source

![SCREEN05](https://github.com/user-attachments/assets/67d939cc-8666-4e93-92ef-8271a120145c)

---

### Easter 3

_Directory buster with common.txt might help._

1. Navigate to `http://IP/login` and view the page source code (right-click and select "View Page Source")

![SCREEN06](https://github.com/user-attachments/assets/07cd945b-c91f-43a5-82d7-2799d0ac704d)

---

### Easter 4

_time-based sqli_

1. In Burp Suite, intercept a POST request from the login form

   - Right-click and select "Save item" to save the request for sqlmap

2. Use sqlmap to identify available databases

```bash
sqlmap -r request --dbs --batch
```

![SCREEN07](https://github.com/user-attachments/assets/eb4ca958-2123-4dcd-b80d-becc06b3c6a2)

3. Once the `THM_f0und_m3` database is identified, enumerate its tables

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 --tables
```

![SCREEN08](https://github.com/user-attachments/assets/80273ad8-0a5d-43a0-a53e-c2ea7ba2ecd3)

4. Examine the structure of the `nothing_inside` table

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 -T nothing_inside --columns
```

![SCREEN09](https://github.com/user-attachments/assets/8280aa6e-3722-4553-b81b-3d71f5b31cf6)

5. Extract the flag from the `Easter_4` column

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 -T nothing_inside -C Easter_4 --sql-query "select Easter_4 from nothing_inside"
```

![SCREEN10](https://github.com/user-attachments/assets/cee5448f-2556-406f-b77d-3a300cf5248f)

---

### Easter 5

_Another sqli_

1. Enumerate columns in the `user` table

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 -T user --columns
```

![SCREEN11](https://github.com/user-attachments/assets/08b732ee-5033-4287-9ca5-760d75d30f87)

2. Extract username and password data

```bash
sqlmap -r request --dbs --batch -D THM_f0und_m3 -T user --sql-query "select password, username from user"
```

![SCREEN12](https://github.com/user-attachments/assets/201bfb79-4f50-4506-b935-508c0d258666)

3. The password hash **05f3672ba34409136aa71b8d00070d1b** can be cracked using [CrackStation](https://crackstation.net/)

![SCREEN13](https://github.com/user-attachments/assets/49d99851-5718-4d79-aed2-e3ffc6bf334d)

4. Log in at `http://IP/login/` to retrieve the flag

![SCREEN14](https://github.com/user-attachments/assets/23d529c9-9372-4082-a06a-5a33cfa1e56a)

---

### Easter 6

_Look out for the response header._

1. Use `curl` with the `-I` flag to examine the HTTP headers

```bash
curl -I http://IP
```

![SCREEN15](https://github.com/user-attachments/assets/6b12bab3-366c-44d2-acda-d6627eb5f34c)

---

### Easter 7

_Cookie is delicious_

1. Visit `http://IP/` and open the browser's Developer Tools (F12)
   - Navigate to the Storage tab
   - Locate the cookie named "Invited" with a value of "0"
   - Change the cookie value from "0" to "1"
   - Refresh the page to reveal the flag

![SCREEN16](https://github.com/user-attachments/assets/0fba764c-cdc9-445c-b58a-f55a072303e1)

### Easter 8

_Mozilla/5.0 (iPhone; CPU iPhone OS 13_1_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.1 Mobile/15E148 Safari/604.1_

1. In Developer Tools `add a custom device` with the specified `User-Agent string`

![SCREEN17](https://github.com/user-attachments/assets/bafd031f-c01a-4b24-b815-819704650b15)

---

### Easter 9

_Something is redirected too fast. You need to capture it._

1. After goint to `http://IP/ready/` we are being redirected further. We can click view source fast enough or just use `curl`

```bash
curl http://IP/ready/
```

![SCREEN18](https://github.com/user-attachments/assets/9512753a-e871-49ad-83ef-3e067e2db74c)

---

### Easter 10

_Look at THM URL without https:// and use it as a referrer._

1. In Burp Suite, capture a request to `http://IP/free_sub/` and modify the `Referer` header to **tryhackme.com**

![SCREEN19](https://github.com/user-attachments/assets/48cfa00e-cdcb-4781-b192-2d434094de22)

---

### Easter 11

_Temper the html_

1. On the main page, there's a menu selection where choosing "salad" gives the message **Mmmmmm... what a healthy choice, I prefer an egg**

![SCREEN20](https://github.com/user-attachments/assets/1b123bf4-c0a9-4654-bc46-30c0374a111e)

2. Capture this request in Burp Suite and modify the `dinner` parameter from "salad" to "egg"

![SCREEN21](https://github.com/user-attachments/assets/f693cee8-c92f-4571-896a-899574234aea)

---

### Easter 12

_Fake js file_

1. Inspect the main page and notice a request to `jquery-9.1.2.js`

![SCREEN22](https://github.com/user-attachments/assets/6ae18413-331f-4941-9c6c-e72c0dba5caf)

2. Convert the `strl` value to hex using [CyberChef](https://gchq.github.io/CyberChef/)

![SCREEN23](https://github.com/user-attachments/assets/d74911e4-f000-43b4-8ec7-9887eff390d9)

---

### Easter 13

1. Navigate directly to `http://IP/ready/gone.php` to find the flag

![SCREEN24](https://github.com/user-attachments/assets/2ea88473-1592-4bea-90db-198a19109821)

### Easter 14

_Embed image code_

1. In the main page source, find a comment containing an image `src` attribute with a base64-encoded string. Use [CyberChef](https://gchq.github.io/CyberChef/) with the "Render Image" option to reveal the flag

![SCREEN25](https://github.com/user-attachments/assets/6fc5c9b5-84b1-428e-b641-670356de4351)

# Easter 15

_Try guest the alphabet and the hash code_

1. Navigate to `http://IP/game1/`

2. nput the full alphabet in both uppercase and lowercase **ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz**

3. Based on the hints provided, determine that the correct input is **GaveOver**

![SCREEN26](https://github.com/user-attachments/assets/3f3285eb-b0f6-4627-a87e-235e85e287f9)

### Easter 16

1. Capture the request from `http://IP/game2/`, and modify the request to include additional parameters

![SCREEN27](https://github.com/user-attachments/assets/0a837aff-0b1d-4d2d-8e0c-bc7928d7dd0e)

---

### Easter 17

_bin -> dec -> hex -> ascii_

1. Examine the source code on the page and locate the `catz` function

2. Extract the binary input from this function

3. Convert the [binary to decimal, then to hexadecimal](https://www.rapidtables.com/convert/number/binary-to-decimal.html), and finally [to ASCII](https://www.rapidtables.com/convert/number/hex-to-ascii.htm)

![SCREEN28](https://github.com/user-attachments/assets/dee2a43f-075b-4497-8054-8148d7b09904)

---

### Easter 18

_Request header. Format is egg:Yes_

1. Add a custom header `egg: Yes` to request in Burp Suite

![SCREEN29](https://github.com/user-attachments/assets/9a35e696-b0ff-46c8-bdfc-db361eac4e74)

---

### Easter 19

_A thick dark line_

1. The image from `http://IP/small.png` contains flag

![SCREEN30](https://github.com/user-attachments/assets/e2e946bc-a954-4c3b-b239-4142157a32a5)

---

### Easter 20

_You need to POST the data instead of GET. Burp suite or curl might help._

1. Modify request to look like the one below

![SCREEN31](https://github.com/user-attachments/assets/08f8e0b2-608f-468e-a91b-d6d2d37f6adb)
