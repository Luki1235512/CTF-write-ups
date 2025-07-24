# [Anthem](https://tryhackme.com/room/anthem)

## Exploit a Windows machine in this beginner level challenge.

# Website Analysis

## This task involves you, paying attention to details and finding the 'keys to the castle'. This room is designed for beginners, however, everyone is welcomed to try it out! Enjoy the Anthem. In this room, you don't need to brute force any login page. Just your preferred browser and Remote Desktop. Please give the box up to 5 minutes to boot and configure.

### Let's run nmap and check what ports are open.

1. Execute an nmap scan to identify open ports and running services

```bash
nmap -sV <TARGET_IP>
```

<img width="725" height="178" alt="SCREEN01" src="https://github.com/user-attachments/assets/710c4803-26d1-4eee-aa74-e183e6b78805" />

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

<img width="512" height="262" alt="SCREEN02" src="https://github.com/user-attachments/assets/512193b6-c313-4122-9a26-b91a2c848198" />

---

### What CMS is the website using?

1. The robots.txt file contains directory references to `/umbraco/`, indicating the website uses the Umbraco Content Management System

---

### What is the domain of the website?

1. The domain name `anthem.com` is displayed in the website header at `http://<TARGET_IP>/`

<img width="407" height="94" alt="SCREEN03" src="https://github.com/user-attachments/assets/43b1c39d-1faa-4ed3-8644-b033ccbc778d" />

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

<img width="678" height="241" alt="SCREEN04" src="https://github.com/user-attachments/assets/3bcffacf-333e-40d6-a1ab-a1371d518166" />

---

### What is flag 2?

_Search for it_

1. The flag is located within the placeholder text of an input field on the website

<img width="690" height="198" alt="SCREEN05" src="https://github.com/user-attachments/assets/64b492b9-e2db-45e1-865d-a6de16507af3" />

---

### What is flag 3?

_Profile_

1. Navigate to Jane Doe's profile page at `http://<TARGET_IP>/authors/jane-doe/`. The flag is embedded within the profile content

<img width="713" height="525" alt="SCREEN06" src="https://github.com/user-attachments/assets/3cdd5b54-4e38-4e31-9ab9-dab812edc965" />

---

### What is flag 4?

_Have we inspected all the pages yet?_

1. Examine the page source of `http://<TARGET_IP>/archive/a-cheers-to-our-it-department/`. The flag is embedded within the HTML source code

<img width="796" height="253" alt="SCREEN07" src="https://github.com/user-attachments/assets/c9ae8c5d-cf34-426f-8678-f4f618954068" />

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

<img width="400" height="128" alt="SCREEN08" src="https://github.com/user-attachments/assets/376969e5-f56d-4cc3-84de-3247fe1d967c" />

---

### Can we spot the admin password?

_It is hidden._

1. Enable the display of hidden files and folders in Windows Explorer

<img width="763" height="141" alt="SCREEN09" src="https://github.com/user-attachments/assets/1a1b3a04-1a0d-4c7c-9524-127cda841559" />

2. Navigate to `C:\backup` directory
3. Locate the `restore.txt` file and modify its permissions:
   - Right-click on `restore.txt`
   - Select Properties > Security > Edit
   - Click Add, enter 'sg', click Check Names, then OK
   - Apply the changes

<img width="403" height="529" alt="SCREEN10" src="https://github.com/user-attachments/assets/bf325062-6673-4d03-8613-6330fde3ab47" />

4. The administrator password is `ChangeMeBaby1MoreTime`

---

### Escalate your privileges to root, what is the contents of root.txt?

1. Using the discovered administrator credentials, access the root flag located at `C:\Users\Administrator\Desktop\root.txt`.

<img width="633" height="191" alt="SCREEN11" src="https://github.com/user-attachments/assets/55eefb29-1925-42c2-8ba0-ab4fa0e6b538" />
