# [Frankenstein's Password Generator](https://spooky.ctfd.io/challenges#Frankenstein's%20Password%20Generator-13)

Dr. Frankenstein left behind a strange password generator in an abandoned NJIT laboratory. The machine refuses to start without an activation phrase. His last notebook entry reads, "A bolt at midnight gave it life." Can you figure out what the program expects and wake the creature?

**1. Recon**

`file` shows a stripped x86-64 ELF. `strings` shows a prompt for an activation phrase, then either "Incorrect phrase" or "IT'S ALIVE!" followed by "The creature whispers: %s". The entry point is 0x401090, and the address loaded into `rdi` before `__libc_start_main` is `main` (0x4012a5).

**2. Phrase check (function at 0x401176)**

`main` reads the input with `fgets`, strips the newline, and passes it to this function. It requires length 14. For each index `i` it computes:

```
((in[i] ^ 0x5d) + 7*i + 0xb) & 0xff == t[i]
```

`t` is the 14 bytes at 0x402010: `2a 24 2a 29 43 37 45 50 5c 5d 65 72 74 6f`.

Inverting each byte gives the phrase:

```python
t = bytes.fromhex("2a242a29433745505c5d6572746f")
p = bytes((((t[i] - 0xb - 7*i) & 0xff) ^ 0x5d) for i in range(14))
# b'BOLTATMIDNIGHT'
```

This matches the hint "A bolt at midnight".

**3. Flag generation (function at 0x4011f9)**

After a correct phrase, `main` calls this function with the phrase. For `i` in 0..28 it computes:

```
out[i] = (13*i + 0x39) ^ data[i] ^ phrase[i % 14]
```

`data` is the 29 bytes at 0x402020. The multiply by `0x4924924924924925` in the disassembly is the compiler's way of computing `i % 14`.

```python
d = bytes.fromhex("0879705b4757b1b49193adeef1dfdbd61a23163b1d6a7744515eabb89a")
out = bytes(((13*i + 0x39) & 0xff) ^ (d[i] ^ p[i % 14]) for i in range(len(d)))
```

**4. Result**

```bash
echo BOLTATMIDNIGHT | ./frankenstein
```

<img width="567" height="153" alt="SCREEN01" src="https://github.com/user-attachments/assets/e3ebbac1-eadc-41bd-b03e-ee9226be86e3" />
