# [Klediophobia](https://spooky.ctfd.io/challenges#Klediophobia-33)

Some keys are better left buried...

After a mysterious midnight key-generation ceremony, SpookyTrust proudly announced the creation of four new intermediate CA certificates. Each passed verification with flying colors.

Whispers of an unusual event buried deep within the HSM logs have raised suspicions about the ceremony. The engineers insist everything went according to plan, but can their certificates really be trusted?

You've been given access to the certificates, signing requests, and audit records. Can you uncover SpookyTrust's darkest secret?

SpookyTrust's audit server is waiting for proof. Recover the compromised signing key and use it to sign a fresh audit message. The server expects an ECDSA-SHA384 signature over the exact UTF-8 message, encoded as Base64 (DER). If you can convince the server that you possess the key, it will surrender the flag.

## 1. Read the audit log

`ceremony.log` shows four keys generated on the HSM. Three go straight from `KEY GENERATE` to `CSR CREATE`. OBJ-0003 does something else:

```
KEY  GENERATE  object=OBJ-0003 curve=P-384 status=OK
HSM  WRAP      object=OBJ-0003 mechanism=AES256-GCM status=OK
HSM  UNWRAP    source=OBJ-0003 object=OBJ-0003-R handler=compat-restore status=OK
CSR  CREATE    object=OBJ-0003-R certificate=ST-ISS-03 status=OK
```

The certificate ST-ISS-03 was issued for the restored key (OBJ-0003-R), not the original. `key-inventory.csv` agrees: ST-ISS-03 maps to OBJ-0003-R. The `compat-restore` handler is the suspicious part, and the restore is the only step in the ceremony that rebuilt a key from scratch. The hypothesis was that the restore produced a degenerate private key (for example a tiny scalar), and everything downstream (CSR, root signature, chain verification) would still pass because the public key is still a valid curve point.

## 2. Test the hypothesis

If the private scalar d is small, the public key is Q = d·G with d small, so brute force over small d recovers it. Script `find_weak.py`:

```python
import sys
from cryptography import x509
from cryptography.hazmat.primitives.asymmetric import ec
LIMIT = 5000
for path in sys.argv[1:]:
    pub = x509.load_pem_x509_certificate(open(path, "rb").read()).public_key().public_numbers()
    hit = None
    for d in range(1, LIMIT):
        if ec.derive_private_key(d, ec.SECP384R1()).public_key().public_numbers() == pub:
            hit = d
            break
    print(path, "->", hit if hit else f"no d below {LIMIT}")
```

Run it against all four intermediate certificates:

```
python3 -I find_weak.py ST-ISS-01.pem ST-ISS-02.pem ST-ISS-03.pem ST-ISS-04.pem
```

Output:

```
ST-ISS-01.pem -> no d below 5000
ST-ISS-02.pem -> no d below 5000
ST-ISS-03.pem -> 1963
ST-ISS-04.pem -> no d below 5000
```

The private key of ST-ISS-03 is d = 1963. The other three keys are not weak.

## 3. Get a challenge from the server

The client defaults to `127.0.0.1:8000`, which gives "Connection refused". `--server` is a top-level argument, so it goes before the subcommand:

```bash
python3 client.py --server http://3.149.129.229:8000 challenge
```

Response:

```json
{
  "session_id": "9e26228754f6f6df81ef2eedbfbef10b",
  "message": "SPOOKYTRUST-AUDIT-V1:c1b6e687ab592d9d28b39809646bc574",
  "signing_algorithm": "ECDSA-P384-SHA384",
  "signature_encoding": "base64(DER)",
  "expires_in_seconds": 300
}
```

## 4. Sign the message with d = 1963

Script `sign.py`. It rebuilds the private key from d, asserts that it matches the public key in ST-ISS-03.pem, then signs the message bytes with ECDSA over SHA-384 and prints the DER signature in Base64:

```python
import sys, base64
from cryptography import x509
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec
c = x509.load_pem_x509_certificate(open("ST-ISS-03.pem", "rb").read())
k = ec.derive_private_key(1963, ec.SECP384R1())
assert k.public_key().public_numbers() == c.public_key().public_numbers(), "mismatch"
print("d=1963 matches ST-ISS-03 public key", file=sys.stderr)
if len(sys.argv) > 1:
    sig = k.sign(sys.argv[1].encode(), ec.ECDSA(hashes.SHA384()))
    print(base64.b64encode(sig).decode())
```

```bash
python3 -I sign.py "SPOOKYTRUST-AUDIT-V1:c1b6e687ab592d9d28b39809646bc574"
```

Output:

```
MGUCMHJDVH2W0CgLQs/tFzpIKxRr76EXp4aeQms5XOpIqu4wpfTbhxcBuBnfXhX2t1iyFAIxAKhdHa6UZO9aTGZ3RqWLj6n02jxSBPRgasj6jhogNawqARD0VaVG9jQpdTopWExmrA==
```

## 5. Submit the signature

```bash
python3 client.py --server http://3.149.129.229:8000 submit 9e26228754f6f6df81ef2eedbfbef10b 'MGUCMHJDVH2W0CgLQs/tFzpIKxRr76EXp4aeQms5XOpIqu4wpfTbhxcBuBnfXhX2t1iyFAIxAKhdHa6UZO9aTGZ3RqWLj6n02jxSBPRgasj6jhogNawqARD0VaVG9jQpdTopWExmrA=='
```

Response:

<img width="1205" height="83" alt="SCREEN01" src="https://github.com/user-attachments/assets/6bbff69c-8db7-40b8-af92-15199f6b750d" />
