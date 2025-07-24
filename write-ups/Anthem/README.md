# [Anthem](https://tryhackme.com/room/anthem)

## Exploit a Windows machine in this beginner level challenge.

# Website Analysis

## This task involves you, paying attention to details and finding the 'keys to the castle'. This room is designed for beginners, however, everyone is welcomed to try it out! Enjoy the Anthem. In this room, you don't need to brute force any login page. Just your preferred browser and Remote Desktop. Please give the box up to 5 minutes to boot and configure.

### Let's run nmap and check what ports are open.

1. Execute an nmap scan to identify open ports and running services

```bash
nmap -sV <TARGET_IP>
```

[SCREEN01]

---

### What port is for the web server?

1. **Answer:** `80`

---

### What port is for remote desktop service?

1. **Answer:** `3389`

---

### What is a possible password in one of the pages web crawlers check for?

_fill in the gap \*\*\*\*\*\*.txt_

1. Navigate to `http://<TARGET_IP>/robots.txt`. The robots.txt file contains the string `UmbracoIsTheBest!` which appears to be a potential password

[SCREEN02]

---

### What CMS is the website using?

1. The robots.txt file contains directory references to `/umbraco/`, indicating the website uses the Umbraco Content Management System

---

### What is the domain of the website?

1. The domain name `anthem.com` is displayed in the website header at `http://<TARGET_IP>/`

[SCREEN03]

---

### What's the name of the Administrator

_Consult the Oracle.(your favourite search engine)_

1. Navigate to `http://<TARGET_IP>/archive/a-cheers-to-our-it-department/` and locate the poem content. Search for this poem using a search engine to identify the author as `Solomon Grundy`

---

### Can we find find the email address of the administrator?

1. On `http://<TARGET_IP>/archive/we-are-hiring/`, an email from Jane Doe is displayed as `JD@anthem.com`. Following the established naming convention (first initial + last initial), Solomon Grundy's email address would be `SG@anthem.com`

---

# Spot the flags

## Our beloved admin left some flags behind that we require to gather before we proceed to the next task..

### What is flag 1?

_Have we inspected the pages yet?_

1. View the page source of `http://<TARGET_IP>/archive/we-are-hiring/`. The flag is embedded within the HTML meta tags

[SCREEN04]

---

### What is flag 2?

_Search for it_

1. The flag is located within the placeholder text of an input field on the website

[SCREEN05]

---

### What is flag 3?

_Profile_

1. Navigate to Jane Doe's profile page at `http://<TARGET_IP>/authors/jane-doe/`. The flag is embedded within the profile content

[SCREEN06]

---

### What is flag 4?

_Have we inspected all the pages yet?_

1. Examine the page source of `http://<TARGET_IP>/archive/a-cheers-to-our-it-department/`. The flag is embedded within the HTML source code

[SCREEN07]

---

# Final stage

## Let's get into the box using the intel we gathered.

### Let's figure out the username and password to log in to the box.(The box is not on a domain)

### Gain initial access to the machine, what is the contents of user.txt?

1. Establish RDP connection using the discovered credentials

```bash
xfreerdp /f /u:sg /p:UmbracoIsTheBest! /v:10.10.174.188
```

2. The `user.txt` file is located on the desktop of the sg user account

[SCREEN08]

---

### Can we spot the admin password?

_It is hidden._

1. Enable the display of hidden files and folders in Windows Explorer

[SCREEN09]

2. Navigate to `C:\backup` directory
3. Locate the `restore.txt` file and modify its permissions:
   - Right-click on `restore.txt`
   - Select Properties > Security > Edit
   - Click Add, enter 'sg', click Check Names, then OK
   - Apply the changes

[SCREEN10]

4. The administrator password is `ChangeMeBaby1MoreTime`

---

### Escalate your privileges to root, what is the contents of root.txt?

1. Using the discovered administrator credentials, access the root flag located at `C:\Users\Administrator\Desktop\root.txt`.

[SCREEN11]
