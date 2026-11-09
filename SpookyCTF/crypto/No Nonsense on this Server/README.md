# [No Nonsense on this Server](https://spooky.ctfd.io/challenges#No%20Nonsense%20on%20this%20Server-34)

an abandoned research laboratory has recovered an old remote-access server running a proprietary protocol called SP00KY. the server has its original users and filesystem, but its administrator credentials have been lost. you have been given access to a general account. recover the contents of /home/admin/flag.txt. host: `` port: 19160 account: `general` the general private key is supplied as `general.pem`. build the reference client from the participant files (`make client`) or reimplement the protocol yourself.

## Step 1: Unpack the participant files

```bash
unzip 21b10add-no-nonsense-participant.zip
cd no-nonsense-participant
```

This gives a C client (`client/main.c`), shared protocol/crypto code (`common/protocol.c`, `common/crypto.c`), the protocol spec (`docs/PROTOCOL.md`), and the `general` account's PEM key in `data/general.pem`.

## Step 2: Read the protocol spec

`docs/PROTOCOL.md` describes SP00KY as a binary TCP protocol with 4 byte headers (version, opcode, 2 byte length) followed by a payload. Authentication works like this:

1. Client sends `AUTH_START` with the username.
2. Server replies with `AUTH_CHALLENGE`, a 32 byte random, single use challenge.
3. Client signs a canonical message with ECDSA over P-256/SHA-256 and sends it as `AUTH_PROVE`:

```
ASCII("SP00KY-AUTH-v1") || uint8(username_length) || ASCII(username) || challenge[32]
```

4. Server verifies the signature against the user's registered public key.

The spec also documents a second, separate signing scheme used for audit log entries in `/var/log/auth.log`:

```
ASCII("SP00KY-LOG-v1") || uint64(timestamp_ms, big-endian) || uint8(username_length) || ASCII(username) || uint16(event_length, big-endian) || event_bytes
```

Each signed log line carries `r` and `s` (the raw ECDSA signature components) plus the signer's public key.

## Step 3: Build the reference client

The README says `make client`, but the Makefile's actual target is `spooky-client`. Build it directly:

```bash
make spooky-client
```

This compiles `client/main.o`, `common/protocol.o`, `common/crypto.o` and links against `libcrypto` (needs `libssl-dev` installed). If OpenSSL headers are missing:

```bash
apt install libssl-dev
```

## Step 4: Connect as `general`

```bash
./spooky-client --host 18.222.247.20 --port 19160 --user general --key data/general.pem
```

This authenticates automatically (the client handles the challenge/sign/prove exchange) and drops into a shell-like prompt: `general@spooky:~$`.

## Step 5: Look around

```bash
ls
cat welcome.txt
ls /var/log
cat /var/log/auth.log
```

`welcome.txt` confirms the admin home directory isn't readable from this account, but authentication history is kept under `/var/log`. `ls /var/log` shows `auth.log` and `syslog`. `cat /var/log/auth.log` dumps a long audit trail: each line has a timestamp, username, hex encoded event description, and for successful signature events, a public key plus `r` and `s` values.

The log contains entries for both `general` and `admin`, each signed with that user's own key, following the `SP00KY-LOG-v1` canonical format from the spec.

## Step 6: Spot the nonce reuse

ECDSA security depends on using a fresh random nonce `k` for every signature. If two different messages are signed with the same key and the same `k`, they produce signatures with the same `r` value, and that leaks the private key.

Extract all admin-signed lines and check for repeated `r`:

```bash
grep "username=admin" auth.log | grep " r=" > admin_sig_lines.txt
grep -oP 'r=\K[0-9a-f]+' admin_sig_lines.txt | sort | uniq -c | sort -rn | head
```

One `r` value shows up twice: `ca7b852e0061ee6223df1ef9df733cb6a6df89daabe51b3a4236e75980afce49`. Pull both full lines:

```bash
grep "r=ca7b852e0061ee6223df1ef9df733cb6a6df89daabe51b3a4236e75980afce49" auth.log
```

Two records come back, both `from 10.7.0.14`, one timestamped `1705029267000` ("accepted signature for admin"), one timestamped `1705029336000` ("session opened for admin"). Same `r`, different `s`, different event text, same public key. That's nonce reuse.

## Step 7: Recover the nonce and private key

Decode the hex event fields to get the plaintext event strings:

```python
bytes.fromhex("6163636570746564207369676e617475726520666f722061646d696e2066726f6d2031302e372e302e3134").decode()
bytes.fromhex("73657373696f6e206f70656e656420666f722061646d696e2066726f6d2031302e372e302e3134").decode()
```

Rebuild each full canonical message per the `SP00KY-LOG-v1` format (prefix, big-endian uint64 timestamp, uint8 username length, username, big-endian uint16 event length, event bytes), then hash each with SHA-256 to get `z1` and `z2`.

With the same `r` and nonce `k` across both signatures, the recovery math is:

```
k = (z1 - z2) * inverse(s1 - s2) mod n
d = (s1*k - z1) * inverse(r) mod n
```

where `n` is the order of the P-256 curve. I wrote this out in Python:

```python
import hashlib, struct

n = 0xffffffff00000000ffffffffffffffffbce6faada7179e84f3b9cac2fc632551

def canonical_log_msg(timestamp_ms, username, event_bytes):
    prefix = b"SP00KY-LOG-v1"
    uname = username.encode()
    return (prefix
            + struct.pack(">Q", timestamp_ms)
            + struct.pack(">B", len(uname))
            + uname
            + struct.pack(">H", len(event_bytes))
            + event_bytes)

def hash_to_int(msg):
    h = hashlib.sha256(msg).digest()
    return int.from_bytes(h, "big") % n

m1 = canonical_log_msg(1705029267000, "admin", bytes.fromhex("6163636570746564207369676e617475726520666f722061646d696e2066726f6d2031302e372e302e3134"))
m2 = canonical_log_msg(1705029336000, "admin", bytes.fromhex("73657373696f6e206f70656e656420666f722061646d696e2066726f6d2031302e372e302e3134"))
z1 = hash_to_int(m1)
z2 = hash_to_int(m2)

r = int("ca7b852e0061ee6223df1ef9df733cb6a6df89daabe51b3a4236e75980afce49", 16)
s1 = int("778a3ba4196e34539ae032d2314bbc26d37b5734ca30b3e2a6b66c110e9b774e", 16)
s2 = int("7ee53fa43f6313b354903a77c0823f50ae5d57a32a3cf4a28c03be93a7cf0227", 16)

k = ((z1 - z2) % n) * pow((s1 - s2) % n, -1, n) % n
d = ((s1 * k - z1) % n) * pow(r, -1, n) % n
```

I sanity checked `d` by recomputing `s1` and `s2` from `k`, `d`, `r`, `z1`, `z2` and confirming they matched the log's actual `s` values. Both matched.

## Step 8: Build a usable private key

The admin account's public key is given in the log lines as a raw uncompressed EC point: `04` followed by the 32 byte X and 32 byte Y coordinates. Using Python's `cryptography` library:

```python
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import serialization

d = 0xb4ade2a2e9404047f56f64d4e10bed17932bb5863180881f7506e90a9c3ebbb1

priv = ec.derive_private_key(d, ec.SECP256R1())
pub_numbers = priv.public_key().public_numbers()
```

Both X and Y matched the admin account's registered public key exactly, confirming `d` is correct. Then export it as a PEM the client can use:

```python
pem = priv.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.TraditionalOpenSSL,
    encryption_algorithm=serialization.NoEncryption(),
)
with open("admin.pem", "wb") as f:
    f.write(pem)
```

## Step 9: Authenticate as admin and read the flag

```bash
./spooky-client --host 18.222.247.20 --port 19160 --user admin --key admin.pem
```

At the prompt:

```
cat /home/admin/flag.txt
```

[SCREEN01]
