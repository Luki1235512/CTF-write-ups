# [Cleithrophobia](https://spooky.ctfd.io/challenges#Cleithrophobia-3)

You are stuck inside of the Enterprise with no way out. The rest of the crew has gone missing. They need help finding a way back to a Federation outpost but the ship is stuck dead in space. Maybe the packet file they have can find a way to start the ship.

1. Get an overview of the capture.

```bash
tshark -r Cleithrophobia.pcap -q -z io,phs
```

Most of the 2000 packets are noise (TLS handshakes, ICMP, NTP, ARP, FTP logins that all succeed). The one useful signal is HTTP `POST /bridge/ignition`, which matches the "start the ship" prompt.

2. List the ignition requests and their responses.

```bash
tshark -r Cleithrophobia.pcap -Y 'http.request.method=="POST"' -T fields -e frame.number -e ip.src
```

There are 20 POSTs. 19 get `401 Unauthorized` (`IGNITION DENIED - INVALID COMMAND CREDENTIALS`). Only frame 404 gets `200 OK`, and its reply is longer than the rest.

3. Dump the successful request and response.

```bash
tshark -r Cleithrophobia.pcap -Y 'frame.number==404 || frame.number==405' -x
```

The request carries `Authorization: Basic Y2FwdGFpbi5raXJrOkVudGVycHJpc2UtMTcwMS1B`, which decodes to `captain.kirk:Enterprise-1701-A`. The failed attempts use the same password with other usernames.

The response has three custom headers, each holding a base64 fragment:

```
X-Bridge-Telemetry:  deflector_part1=U3Bvb2t5e0JlMA==
X-Deflector-Status:  deflector_part2=bV9NZV9VcF9TYw==
X-Warp-Core-Reading: deflector_part3=MHR0eV8yMjY2fQ==
```

4. Decode and join the parts.

```bash
echo -n U3Bvb2t5e0JlMA== | base64 -d
echo -n bV9NZV9VcF9TYw== | base64 -d
echo -n MHR0eV8yMjY2fQ== | base64 -d
```

<img width="401" height="96" alt="SCREEN01" src="https://github.com/user-attachments/assets/6bbcefc5-28df-43c3-b86d-1cb43a80852d" />
