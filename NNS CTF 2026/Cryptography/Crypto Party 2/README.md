# [Crypto Party 2](https://nnsc.tf/challenges?challenge=crypto_Crypto+Party+2)

**Description:**

Thanks to whoever forged invites last year, now this year's budget is ruined. We have patched the invitation system. You can still bring some friends this year, but don't push it!

## 1. What the service gives you

Connecting to the service prints a ciphertext `ct`, then lets you name up to 6 "friends". Each name you submit gets signed and returned as `r:s`, an ECDSA signature.

```
ct: <big number>
You can invite up to 6 friends.
Enter the name of your friend: alice
Invitation code = r:s
...
```

## 2. Reading the source

The challenge ships `chall.py`. The relevant part:

```python
from ecdsa import curves
G = curves.NIST256p.generator
n = curves.NIST256p.order
secret_key = secrets.randbelow(n - 1) + 1

key = long_to_bytes(secret_key, 32)
cipher = AES.new(key, AES.MODE_ECB)
ct = bytes_to_long(cipher.encrypt(pad(flag, 16)))

def invite(m):
    h = bytes_to_long(sha256(m.encode()).digest())
    k = bytes_to_long(str(uuid.uuid4())[:32].encode())
    P = k * G
    r = P.x() % n
    s = (pow(k, -1, n) * (h + r * secret_key)) % n
    return r, s
```

So the flag is AES-ECB encrypted under a key derived directly from `secret_key`, the ECDSA private key used to sign the invite names. If we recover `secret_key`, we decrypt the flag.

Everything else is textbook ECDSA on NIST P-256, except one line:

```python
k = bytes_to_long(str(uuid.uuid4())[:32].encode())
```

The nonce `k` isn't a random 256-bit integer. It's the ASCII bytes of a UUID4's string form, truncated to 32 characters. That line is the entire vulnerability.

## 3. Why that line breaks ECDSA

`str(uuid.uuid4())` looks like:

```
750cff6d-a67f-43b4-aa3b-804b19e38d8d
```

36 characters, laid out as `8-4-4-4-12` hex groups separated by hyphens. `[:32]` keeps the first 32 characters, which covers the first four groups in full and only the first 8 of the last group's 12 characters.

Indexing that 32-character string:

- Positions 8, 13, 18, 23: always `-`
- Position 14: always `4`
- Position 19: one of `8`, `9`, `a`, `b`
- All other 26 positions: a fully random hex character `0-9a-f`
  `bytes_to_long` treats this string as a big-endian byte string, so position 0 is the most significant byte of `k`.

The important detail is what a "random hex character" means at the byte level. ASCII `'0'-'9'` is `0x30-0x39`. ASCII `'a'-'f'` is `0x61-0x66`. So a byte at a free position isn't uniform over 256 values, it's confined to 16 specific values clustered in two narrow ranges. Each free byte therefore leaks about 4 to 4.3 bits less entropy than a real random byte, and it does so at every single byte position across the entire 256-bit value, including the most significant ones. A truly random 256-bit nonce has 256 bits of entropy. This one has about 106.

When the nonce isn't uniformly random, and you can collect several signatures made with the same private key, you can usually recover the private key with a lattice attack, because each signature leaks a little information about `x`, and a handful of leaky signatures is enough to pin it down exactly. That's why the challenge lets you request up to 6 invites: 6 leaky signatures is the intended amount of data to solve this.

## 4. Setting up the equations

For each signature you get `(r, s)` for a known message, so you can compute `h = sha256(name)` yourself. ECDSA signing says:

```
s = k^-1 * (h + r * x) (mod n)
```

which rearranges to:

```
k = s^-1 * h + s^-1 * r * x (mod n)
```

Define `A = s^-1 * r mod n` and `B = s^-1 * h mod n`. Then for every signature:

```
k ≡ A*x + B (mod n)
```

`A` and `B` are things we can compute directly from the data we got over the wire. `x` is the secret key we want. `k` is the biased nonce from section 3, which we don't know exactly, but we know its structure very precisely.

## 5. Building the lattice

For signature `i`, write `k_i = K + sum_j (w_j * u_{i,j})`, where:

- `K` is the baseline value of the 32-byte template with every unknown byte set to `'0'`
- `w_j = 256^(31-idx)` is the positional weight of unknown byte at string index `idx`
- `u_{i,j}` is the actual offset of that byte for that signature, bounded in `[0, 64)`

Substituting into `k_i ≡ A_i*x + B_i (mod n)` gives, per signature:

```
sum_j (w_j * u_{i,j}) - A_i * x ≡ (B_i - K) (mod n)
```

Across 6 signatures and 27 ambiguous byte positions each, that's 162 small unknowns plus the one big unknown `x`, tied together by 6 modular equations. Build a lattice where:

- one row per equation encodes `n`
- one row encodes `x`'s coefficients across all 6 equations, plus a scaled diagonal entry in a dedicated "x column"
- one row per `(signature, byte position)` pair encodes that byte's weight in its equation, plus a scaled diagonal entry in its own dedicated column

The diagonal scaling factor is the standard trick from this style of attack: multiply by `delta / 2^width` so that regardless of the actual unknown value, that column's contribution stays confined to a small, predictable range. This is what lets LLL treat "the right combination of unknowns" as a genuinely short vector instead of getting lost among unrelated combinations. The paper gives a formula for how small `delta` needs to be based on the lattice dimension; it comes out tiny, not something to guess casually.

Set up a target vector: the equation coordinates get the known `(B_i - K) mod n` values, and every other coordinate gets the constant `delta / 2`, since that's the expected midpoint of whatever value ends up there.

Then embed the whole thing and run LLL. Somewhere in the reduced basis there will be a row ending in `+-1`; that row, combined with the target, gives you the closest lattice point to what you were looking for. Read off the "x column" entry of that point, divide out the known scaling factor, reduce mod `n`, and that's your recovered secret key.

A subtlety worth calling out explicitly: get the bit-width you assign to `x` exactly right. The whole "expected midpoint is delta/2" trick depends on the true range of each unknown matching the range implied by its assigned width. Pad it by even one bit and the midpoint assumption is wrong, and the attack silently returns a confident-looking but incorrect answer instead of failing loudly.

## 6. Step by step, concretely

1. **Connect to the service and collect data.** Use 6 distinct, simple names:

```bash
ncat --ssl TARGET_IP 1337
```

Record `ct` and each `name:r:s` triple in order.

2. **Run the attack.** Fill in `n`, `ct`, and the six `(name, r, s)` triples at the top of the script below, install `fpylll`, and run it. It builds the equations, constructs the lattice, recovers `x`, and decrypts the flag in one pass:

```python
import math
from hashlib import sha256
from fpylll import IntegerMatrix, LLL
from ecdsa import curves
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes

n = curves.NIST256p.order

# Fill in with data from the live connection
ct = 1253546417541081515081365971702178531913551160080080529743305483492953125293256716906415965101146800431642250079975
sigs = [
    ("alice", 18812793890079175327590870326363573536306790833486997298152943326591905664604, 87435663684723919235279365042463943127484888126797722952535860619669503800808),
    ("bob",   102521139359477881545787928376385801283942769462553600776027086587259778311061, 104536456697384705067603620240549106034434673707986684524727239433143244945487),
    ("carol", 33738271115447791563718391343398601625663292522536733938204249600662713230244, 89215573398344099612440129297167169122649326790936506900966712009157510167566),
    ("dave",  12461556882868284580960193099041883866675276254774681516554364779756957579585, 95465509397515933972423210346826147001923628547350708227598455999035674253888),
    ("erin",  78053476582010823090642766535053784573571017782164189480679671823447643918753, 19785212997721494456841171085316746645987311818495657183729585080170294605688),
    ("frank", 57614647062210786735545484075377210090889543288068481627136784234955734005671, 86175039182133900683418264748196491722076304191880798030276942501938527289039),
]

# Per-signature A_i, B_i so that k_i = A_i*x + B_i (mod n)
equations_raw = []
for name, r, s in sigs:
    h = int.from_bytes(sha256(name.encode()).digest(), 'big')
    A = pow(s, -1, n) * r % n
    B = pow(s, -1, n) * h % n
    equations_raw.append((A, B))

# Model the 32-char uuid4-truncated template
def string_template():
    tmpl = [None] * 32
    fixed = {8: '-', 13: '-', 14: '4', 18: '-', 23: '-'}
    for i, c in fixed.items():
        tmpl[i] = ('fixed', ord(c))
    tmpl[19] = ('variant',)
    for i in range(32):
        if tmpl[i] is None:
            tmpl[i] = ('free',)
    return tmpl

def baseline_and_positions():
    tmpl = string_template()
    K_bytes = [0] * 32
    unknown = []
    for i, t in enumerate(tmpl):
        if t[0] == 'fixed':
            K_bytes[i] = t[1]
        elif t[0] == 'free':
            K_bytes[i] = ord('0')
            unknown.append(i)
        elif t[0] == 'variant':
            K_bytes[i] = ord('8')
            unknown.append(i)
    return int.from_bytes(bytes(K_bytes), 'big'), unknown

K, unknown_positions = baseline_and_positions()
weight = lambda idx: 256 ** (31 - idx)
CHUNK_WIDTH = 6  # bound=64, covers the true max range of 54
chunks_template = [(weight(idx), CHUNK_WIDTH) for idx in sorted(unknown_positions)]

# Equation form for the solver: sum(w_j*u_j) + a_i*x = b_i (mod n)
equations = [((-A) % n, (B - K) % n, chunks_template) for A, B in equations_raw]

# Extended-HNP lattice solver
def solve_hnp_multi_chunk(n, equations, x_width, delta_prec=64):
    d = len(equations)
    m = 1
    L = sum(len(chs) for _, _, chs in equations)
    D = d + m + L

    KDf = 2 ** (D / 4) * math.sqrt(m + L)
    KDf = (KDf + 1) / 2
    deltaf = 1 / (2 * KDf)
    delta_num = max(1, int(deltaf * (1 << delta_prec)))

    maxwidth = max([x_width] + [w for _, _, chs in equations for (_, w) in chs])
    def entry(width):
        return delta_num << (maxwidth - width)

    Dtot = D + 1
    B = IntegerMatrix(Dtot, Dtot)
    DEN = 1 << (delta_prec + maxwidth)

    for i in range(d):
        B[i, i] = n * DEN

    x_col = d
    for i, (a_i, b_i, chs) in enumerate(equations):
        B[x_col, i] = (a_i % n) * DEN
    B[x_col, x_col] = entry(x_width)

    target = [0] * Dtot
    kcol = d + m
    for i, (a_i, b_i, chs) in enumerate(equations):
        target[i] = (b_i % n) * DEN
        for (w, width) in chs:
            B[kcol, i] = (w % n) * DEN
            B[kcol, kcol] = entry(width)
            kcol += 1

    half = (delta_num << maxwidth) // 2
    for j in range(d, D):
        target[j] = half

    for c in range(D):
        B[D, c] = target[c]
    B[D, D] = 1

    LLL.reduction(B)

    scale_x = entry(x_width)
    candidates = []
    for r in range(Dtot):
        last = B[r, D]
        if abs(last) == 1:
            close_xcol = target[x_col] - last * B[r, x_col]
            candidates.append((close_xcol // scale_x) % n)
    return candidates

x_width = n.bit_length()
candidates = solve_hnp_multi_chunk(n, equations, x_width)
print("recovered x candidates:", candidates)

# Try each candidate as the AES key and decrypt
for x in candidates:
    key = long_to_bytes(x, 32)
    pt = AES.new(key, AES.MODE_ECB).decrypt(long_to_bytes(ct))
    print(x, "->", pt)
```

3. **Running the script above on the real data prints:**

```
recovered x candidates: [16736750074410728292486910617146331716748155795893055879109016518995674924306]
16736750074410728292486910617146331716748155795893055879109016518995674924306 -> b'NNS{***_*****_***_********_**********}\n...'
```
