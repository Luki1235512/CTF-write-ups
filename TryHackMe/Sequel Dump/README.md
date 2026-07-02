# [Sequel Dump](https://tryhackme.com/room/hfb1sequeldump)

## Can you decipher the captured traffic of an SQL Injection attack?

A wave of suspicious web requests has been detected, hammering our database-driven application. Analysts suspect an automated SQL injection attack has been launched using sqlmap, leading to potential data exfiltration. Investigate the provided packet capture (PCAP) file to uncover the attacker's actions and determine what was stolen!

### What is the flag?

1. **Open `challenge.pcapng` in Wireshark and get a feel for the traffic.**

Filter for HTTP traffic with `http` in the display filter bar. You'll immediately notice a flood of GET requests, all targeting the same endpoint. The query parameters are URL-encoded and very long. Hallmark of automated SQL injection tooling like sqlmap.

2. **Identify the injection technique.**

Inspecting a few decoded query strings reveals a **binary search** / **boolean-based blind SQL injection** pattern. The attacker is not dumping data directly. Instead, each request asks a true/false question of the form:

```sql
... AND ORD(MID(CAST(`column` AS NCHAR), <position>, 1)) > <value> --
```

This is the classic **bisection** approach sqlmap uses to reconstruct string values character by character:

- If the server responds with results -> the condition is **true**.
- If the server responds with `"No results found"` -> the condition is **false**.

By repeatedly halving the search space, sqlmap narrows down each character's ASCII value in about 7–8 requests per character. The attacker is extracting entire columns row by row, character by character.

3. **Understand the response oracle.**

The two possible server responses act as the boolean oracle:

- **Hit**: the page body does **not** contain `"No results found"`. The character's ASCII code is greater than the tested value.
- **Miss**: the page body contains `"No results found"`. The ASCII code is less than or equal to the tested value.

4. **Correlate requests to responses using TCP sequence numbers.**

HTTP in this capture is over plain TCP. Each HTTP request and its matching response share a TCP stream. The key insight for correlation is the **request's** `TCP.ack` field equals the **response's** `TCP.seq` field.

This is how the script below links each response back to the request that triggered it, so the boolean result can be attributed to the right tuple.

5. **Reconstruct the exfiltrated data with a Python script.**

```py
import re
from collections import defaultdict
from scapy.all import rdpcap, IP, TCP, Raw
from urllib.parse import unquote, parse_qs
from scapy.layers.http import HTTPRequest, HTTPResponse

PCAP_FILE = "challenge.pcapng"
requests = {}
bounds = {}

for pkt in rdpcap(PCAP_FILE):
    if pkt.haslayer(HTTPRequest):
        try:
            path = pkt[HTTPRequest].Path.decode()
            query = parse_qs(unquote(path.split("?", 1)[1])).get("query", [None])[0]
            condition, val = query.rsplit(">", 1)
            pos = int(re.search(r"MID\(.*?,(\d+),1\)", condition, re.I).group(1))
        except:
            continue
        key = (pkt[IP].src, pkt[IP].dst, pkt[TCP].sport, pkt[TCP].dport, pkt[TCP].ack)
        requests[key] = (condition, pos, int(val))

    elif pkt.haslayer(HTTPResponse) and pkt.haslayer(Raw):
        key = (pkt[IP].dst, pkt[IP].src, pkt[TCP].dport, pkt[TCP].sport, pkt[TCP].seq)
        if key not in requests:
            continue
        condition, pos, val = requests[key]
        hit = b"No results found" not in pkt[Raw].load
        field = re.search(r"CAST\(`?(\w+)`?\s+AS\s+NCHAR", condition, re.I)
        row   = re.search(r"LIMIT\s+(\d+),1", condition, re.I)
        if not field or not row:
            continue
        b = bounds.setdefault((row.group(1), field.group(1), pos), [0, 255])
        b[0] = max(b[0], val) if hit else b[0]
        b[1] = min(b[1], val) if not hit else b[1]

table = defaultdict(dict)
for (r, col, pos), (lo, _) in bounds.items():
    if 32 <= lo <= 126:
        table[(r, col)][pos] = chr(lo + 1)

for r in sorted({r for r, _ in table}, key=int):
    print(f"Row {r}:")
    for col in ("id", "name", "description"):
        val = "".join(table[(r, col)][p] for p in sorted(table[(r, col)])) if (r, col) in table else "[missing]"
        print(f"  {col:12}: {val}")
    print()
```

The script prints the reconstructed database rows. The last `description` field contains the flag:

[SCREEN01]
