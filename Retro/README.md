# [Retro](https://tryhackme.com/room/retro)

## New high score!

# Pwn

## Can you time travel? If not, you might want to think about the next best thing.

### A web server is running on the target. What is the hidden directory which the website lives on?

_dirbuster 2.3 medium_

1. The initial phase involves standard enumeration by performing a directory scan using `Gobuster` to discover hidden paths on the webserver
   - The scan reveals a hidden directory called **/retro**

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![SCREEN01](https://github.com/user-attachments/assets/c1fe1f7e-04e3-4565-a559-ed4346012363)

---

### user.txt

_Don't leave sensitive information out in the open, even if you think you have control over it._

1. After discovering the WordPress site, detailed enumeration helps identify potential vulnerabilities or information leaks
   - Navigating to the author profile at `http://IP/retro/index.php/author/wade/` provides information about one of the site users
   - Reviewing recent posts leads to a comment at `http://IP/retro/index.php/2019/12/09/ready-player-one/#comment-2` containig his password - **parzival**

![SCREEN02](https://github.com/user-attachments/assets/489f59e4-f936-4b13-b251-e51a8d4a988c)

2. Now we could do [this php reverse shell](https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php) in any of the php files in `http://IP/retro/wp-admin/theme-editor.php` but it is easier to just connect via RDP with the same credentials

3. With these credentials, two potential attack vectors emerge
   - **WordPress Admin Panel**: Logging into wp-admin allows modification of theme files to include a PHP reverse shell, as referenced in the write-up using [this PHP reverse shell](https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php)
   - **Remote Desktop Protocol (RDP)**: Given Windows systems often have RDP enabled, direct remote access provides a more comprehensive interface

![SCREEN03](https://github.com/user-attachments/assets/99590b3b-9c6c-436f-8569-f883ab272304)

3. The RDP approach offers a full GUI environment. Connection can be established using xfreerdp

```bash
xfreerdp /u:wade /p:parzival /v:IP
```

4. There is a `user.txt` file with user flag on the desktop

![SCREEN04](https://github.com/user-attachments/assets/3e7a4849-eccc-4c6a-8f30-82c16cd02259)

---

### root.txt

_Figure out what the user last was trying to find. Otherwise, put this one on ice and get yourself a better shell, perhaps one dipped in venom._

1. In `FreeRDP` Open `Google Chrome` and click on the Bookmark with **CVE**

![SCREEN05](https://github.com/user-attachments/assets/ea89996c-fa01-4021-b059-04f057038ec4)

2. Following the exploit path referenced in [the CVE documentation](https://github.com/nobodyatall648/CVE-2019-1388?tab=readme-ov-file#cve-2019-1388--abuse-uac-windows-certificate-dialog), an executable is needed to trigger the vulnerability. Checking the Recycle Bin reveals the required executable that had been discarded but not permanently deleted

3. Read the flag

```bash
type root.txt.txt
```
