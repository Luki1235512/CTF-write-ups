# [The Silent Transfer](https://tryhackme.com/room/operationsilenttransfer)

## THM Security Services has been engaged for a Threat Hunting activity for Helios Software Group.

# Case Briefing

**A quiet transfer. A widening breach.**

You have been brought on at THM Security Services (TSS). You are a **THREAT HUNTER**.

Your case briefing is waiting in the TSS Operations Hub. Head to the **Active Case** tab for everything you need before you begin. The other tabs are there if you want to explore the company, the team, or previous cases.

# Active Case

The Silent Transfer - Active investigation

| CLIENT                | INDUSTRY   | LEAD ANALYST |
| --------------------- | ---------- | ------------ |
| Helios Software Group | Technology | Lukas Vogel  |

## Situation Report

At 22:41 on 14 November, Helios Software Group's security team escalated suspicious encrypted outbound traffic from a developer workstation in its internal network. A Snort alert identified possible command-and-control traffic, and the firewall showed follow-on connections that did not match the workstation's normal development activity.

Helios has isolated the workstation but cannot yet explain how the activity started, whether the host was used for internal movement, or whether data left the network. TSS has been engaged to reconstruct the network activity and determine the scope of the incident.

## Internal Message

```
FROM: Lukas Vogel, Lead Threat Hunter
TO: You
SUBJECT: RE: Helios — Silent Transfer
```

I need you on the network evidence bundle for the isolated developer workstation. The alert gives us the starting point, but I want the full story around it: what communicated, when it started, whether the traffic stayed on one host, and whether the pattern supports data transfer rather than ordinary development activity.

The packet capture, alert logs and network telemetry are available in the room. Work from the evidence, not the alert name, and separate confirmed network behaviour from assumptions about intent.

## Mission Parameters

1. Establish whether the alert represents real command-and-control activity.
2. Identify the affected workstation and the external infrastructure involved.
3. Determine what happened before and after the first suspicious connection.
4. Assess whether activity extended beyond the initial workstation.
5. Confirm whether data was transferred out and estimate the exposure.

## Assigned Personnel

Lukas Vogel - `Threat Hunt Lead`
You - `Assigned Analyst`

# The Investigation

A forensic workstation is available for this investigation. It includes Wireshark, TShark, Zui, Zeek command-line tools such as `zeek-cut`, and standard terminal utilities.

All evidence is stored in `/home/ubuntu/capstone/`:

- `snort_alerts.log`: Snort detection output
- `zeek_logs/`: Zeek connection, DNS, TLS, HTTP, file, and notice logs
- `investigation.pcap`: Packet capture for packet-level validation
- `fortigate_traffic.log`: Firewall traffic covering internal and cross-subnet activity
- `references/`: Local threat intelligence and MITRE ATT&CK reference material

Use the questions below to guide your analysis of the available evidence.

### Review the detection evidence around 03:47 UTC and correlate it with the repeated C2 traffic. Which internal IP address originated that traffic?

1. Open `investigation.pcap` in Wireshark.

2. Because Wireshark shows relative time by default, switch to absolute time so the alert timestamp lines up with what you see on screen: `View -> Time Display Format -> Date and Time of Day`.

3. Filter on the relevant protocol and narrow to the alert window. Since the alert flags HTTP-based beaconing, filter by `http.request` and scroll to `03:47:25`.

<img width="1401" height="836" alt="SCREEN01" src="https://github.com/user-attachments/assets/a2f3f7c2-6226-406c-a2ee-6a90c448ecce" />

**Answer:** `10.14.30.88`

---

### Working backwards from the C2 activity, which domain was used to deliver the initial dropper to the compromised workstation?

_The delivery domain resolves to an IP in the same /24 subnet as the confirmed C2._

1. Pivot from the C2 IP's resolved subnet. Since the delivery domain resolves into the same /24 as the C2 server, filter DNS traffic first and look for **queries**, not responses, to see what the workstation was actually asking for: `dns.flags.response == 0`

2. Sort by time and look for the earliest suspicious-looking domain queried by `10.14.30.88`, prior to the first C2 beacon at 03:47. In this case, packet `129` shows the query.

<img width="1403" height="870" alt="SCREEN02" src="https://github.com/user-attachments/assets/14973b03-da46-4c95-9998-03b930eb48b1" />

**Answer:** `cdn-updates.microsoftservice.net`

---

### Identify the file downloaded from the delivery domain. What is its SHA256 hash?

_files.log records every file Zeek saw on the wire. Two files are logged._

1. `files.log` is the fastest way to enumerate every file object Zeek extracted from the traffic:

```bash
cat zeek_logs/files.log | zeek-cut sha256
```

**Results:**

```
7f3b2e1a9c8d4f5e6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f90
a3f8e2c1d4b7a9e0f2c3d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6
```

**Answer:** `7f3b2e1a9c8d4f5e6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f90`

---

### Which source port did the compromised workstation use for its first connection to the C2 server?

1. The Fortigate firewall log captures every session traversing the perimeter, including source port, so it's a faster source of truth here than re-deriving it from the pcap:

```bash
grep -i "10.14.30.88" fortigate_traffic.log | head -n 10
```

**Results:**

```
date=2025-11-14 time=01:23:00 devname="FW-DEV-01" devid="FGT60E4Q17000000" logid="0000000013" type="traffic" subtype="forward" level="notice" vd="root" eventtime=1763083380 srcip=10.14.30.88 srcport=51000 srcintf="dev-lan" srcintfrole="lan" dstip=194.165.16.56 dstport=443 dstintf="wan1" dstintfrole="wan" sessionid=757736 proto=6 action="accept" policyid=5 policytype="policy" service="HTTPS" dstcountry="Netherlands" srccountry="Reserved" duration=2 sentbyte=964 rcvdbyte=258 sentpkt=1 rcvdpkt=1
...
```

**Answer:** `51000`

---

### Review the TLS activity between the compromised workstation and the C2 server. What JA4 fingerprint identifies the C2 client?

1. `ssl.log` already has the JA4 fingerprint computed for every TLS session Zeek observed:

```bash
cat zeek_logs/ssl.log | grep "194.165.16.56"
```

**Results:**

```
1763083380.000000	CDhMvog1ELRT522A0A	10.14.30.88	51000	194.165.16.56	443	TLSv13	TLS_AES_256_GCM_SHA384	x25519	-	F	-	h2	T	ChSsNsc	-	-	F	self signed certificate	t13d190900_9dc949149365_97f8aa674fd9
...
```

2. The consistent JA4 value across all sessions to `194.165.16.56` confirms the same client tooling is reused for every beacon, reinforcing that this is malware, not a browser or legitimate agent.

**Answer:** `t13d190900_9dc949149365_97f8aa674fd9`

---

### After C2 was established, how many unique internal destination IP addresses did the compromised workstation contact during its SMB discovery activity?

1. In Wireshark, filter to only the workstation's SMB-related traffic: `ip.src == 10.14.30.88 && (tcp.port == 445 || tcp.port == 139)`

2. Use the built-in endpoint statistics: `Statistics -> Endpoints -> IPv4`. This aggregates every unique destination the host talked to over the filtered traffic.

<img width="1404" height="863" alt="SCREEN03" src="https://github.com/user-attachments/assets/cbf7bc58-1878-409f-a7e1-260766a56997" />

**Answer:** `23`

---

### Following the SMB activity, the attacker established an RDP connection to an internal server. What is the destination IP address?

1. Narrow the workstation's outbound traffic to RDP's standard port: `ip.src == 10.14.30.88 && tcp.port == 3389`

<img width="1401" height="860" alt="SCREEN04" src="https://github.com/user-attachments/assets/17f648fd-177c-46ca-9f79-36545ac73f45" />

**Answer:** `10.14.0.12`

---

### Review the DNS activity originating from the RDP destination. Which domain did the server resolve immediately before the large outbound transfer?

1. Pivot the filter to the newly-compromised internal server rather than the original workstation, since the attacker is now operating from this host: `ip.src == 10.14.0.12 && dns`

<img width="1402" height="865" alt="SCREEN05" src="https://github.com/user-attachments/assets/bb449b7f-9a9d-4794-96a8-34499eabee23" />

**Answer:** `backup.corpfiles-sync.com`

---

### Identify the archive transferred from the internal server to the external endpoint. What is its SHA256 hash?

1. Return to `files.log`, this time focusing on the second logged file rather than the first:

```bash
cat zeek_logs/files.log | zeek-cut sha256
```

**Results:**

```
7f3b2e1a9c8d4f5e6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f90
a3f8e2c1d4b7a9e0f2c3d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6
```

**Answer:** `a3f8e2c1d4b7a9e0f2c3d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6`

---

### Inspect the application-layer contents of the C2 traffic. What command did the attacker issue to the compromised workstation?

1. Filter to the workstation's HTTP-based C2 channel: `ip.src == 10.14.30.88 && http`

2. Right-click any matching packet and select `Follow -> HTTP Stream`. Multiple streams exist across the session; switch to stream `294`, which contains a POST/response pair carrying a command in a JSON-style body.

3. The command is passed as a cmd parameter, and its value is Base64-encoded. Simple obfuscation technique to avoid signature-based detection on the wire. Decode it to recover the plaintext instruction.

<img width="1401" height="868" alt="SCREEN06" src="https://github.com/user-attachments/assets/d22b19c1-1b6a-45e2-8578-8a476267df19" />

3. Decode value in `cmd` from Base64

**Answer:** `whoami`
