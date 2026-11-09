# [Cyberphobia: SpOoKy KeYsTrEaMs](https://spooky.ctfd.io/challenges#Cyberphobia:%20SpOoKy%20KeYsTrEaMs%20-21)

Nothing is secure nowadays, not even the own ciphers my machine generates. Take a look for yourself, I don't know how people on the internet were able to retrieve my encrypted flag with just these files...

## Step 1: Read the source

```bash
cat -n backup.py
```

```py
1  from Crypto.Cipher import AES
2
3
4  def encrypt(message, key):
5      cipher = AES.new(key, AES.MODE_CTR, nonce=b"ghost123", initial_value=0)
6      return cipher.encrypt(message.ljust(64, b" "))
```

Line 5 is the bug. The nonce is the constant `ghost123` and the counter always starts at 0. AES-CTR turns AES into a stream cipher by encrypting `nonce || counter` for each block and XORing the result with the plaintext. With the same key, nonce and starting counter, every call produces the same keystream `KS`. This is a two-time pad.

Line 6 pads every message to 64 bytes with spaces. That fixes all ciphertexts at 64 bytes, and the flag will come back with trailing spaces.

## Step 2: Read the data

```bash
cat -n data.txt
```

```
6  Known message: 546865206861756e746564207661756c7420636c6f736573206174206d69646e696768742e2053746179206f7574736964652e20202020202020202020202020
7  Encrypted message: e4a8bb39921d6f0e3940a040ae22a1afc698fdeb3058ea8e1002fe25142227c6f7f018ddf3eb9ce75e69dbdc59720fcffd98a9c0bb9f03f0d644f34c68a4c306
8  Encrypted flag: c3b0b176910561033e16f151920887be9298bea77f0bafdd1043aa25596b6388beb75089fdebefb31f30db930c265c86b9dda7c0bb9f03f0d644f34c68a4c306
```

The known message decodes to "The haunted vault closes at midnight. Stay outside." plus space padding. Call the three values `P1` (known message), `C1` (encrypted message) and `C2` (encrypted flag).

## Step 3: Work out the math

CTR encryption is `C = P xor KS`, and both messages used the same `KS`:

```
C1 = P1 xor KS
C2 = Flag xor KS
```

XOR is its own inverse, so:

```
KS   = P1 xor C1
Flag = C2 xor KS
```

The key is never needed.

## Step 4: Run it

`data.txt` mentions a `solve.py`, but it wasn't among the provided files, so I did the XOR with a one-off command. The `-I` flag runs Python in isolated mode, so it doesn't pick up stray modules from the current directory.

```bash
python3 -I -c "
p=bytes.fromhex('546865206861756e746564207661756c7420636c6f736573206174206d69646e696768742e2053746179206f7574736964652e20202020202020202020202020')
c=bytes.fromhex('e4a8bb39921d6f0e3940a040ae22a1afc698fdeb3058ea8e1002fe25142227c6f7f018ddf3eb9ce75e69dbdc59720fcffd98a9c0bb9f03f0d644f34c68a4c306')
f=bytes.fromhex('c3b0b176910561033e16f151920887be9298bea77f0bafdd1043aa25596b6388beb75089fdebefb31f30db930c265c86b9dda7c0bb9f03f0d644f34c68a4c306')
ks=bytes(a^b for a,b in zip(p,c))
print(len(p),len(c),len(f))
print(bytes(a^b for a,b in zip(f,ks)))
"
```

Line by line:

- `p`, `c`, `f` are the three hex strings decoded to bytes (`P1`, `C1`, `C2`).
- `ks` is `P1 xor C1`, which is the keystream.
- The first `print` checks that all three inputs are 64 bytes. They are, so `zip` truncates nothing.
- The second `print` XORs `C2` with `ks` to recover the flag.

Output:

```
64 64 64
b'spooky{cs351JKS}                                                '
```

The trailing spaces come from the `ljust(64, b" ")` padding on line 6 of `backup.py` and are not part of the flag.
