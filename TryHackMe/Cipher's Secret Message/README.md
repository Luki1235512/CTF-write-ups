# [Cipher's Secret Message](https://tryhackme.com/room/hfb1cipherssecretmessage)

## Sharpen your cryptography skills by analyzing code to get the flag.

One of the Ciphers' secret messages was recovered from an old system alongside the encryption algorithm, but we are unable to decode it.

**Order:** Can you help void to decode the message?

**Message:** a_up4qr_kaiaf0_bujktaz_qm_su4ux_cpbq_ETZ_rhrudm

**Encryption algorithm:**

```py
from secret import FLAG

def enc(plaintext):
    return "".join(
        chr((ord(c) - (base := ord('A') if c.isupper() else ord('a')) + i) % 26 + base)
        if c.isalpha() else c
        for i, c in enumerate(plaintext)
    )

with open("message.txt", "w") as f:
    f.write(enc(FLAG))
```

1. The function uses enumerate(plaintext), so each character is processed with its index i.
2. Alphabetic characters are shifted by +i positions:
   - uppercase letters use base = ord('A')
   - lowercase letters use base = ord('a')
3. The shift is modulo 26, so the alphabet wraps around.
4. Non-alphabetic characters like \_, 4, and 0 are left unchanged.
5. This is effectively a progressive Caesar cipher: the first letter is shifted by 0, the second by 1, the third by 2, etc.

### What is the flag?

1. Reverse the operation by subtracting the position `i` from each alphabetic character:

```py
def dec(ciphertext):
    return "".join(
        chr((ord(c) - (base := ord('A') if c.isupper() else ord('a')) - i) % 26 + base)
        if c.isalpha() else c
        for i, c in enumerate(ciphertext)
    )

ciphertext = "a_up4qr_kaiaf0_bujktaz_qm_su4ux_cpbq_ETZ_rhrudm"
print("THM{" + dec(ciphertext) + "}")
```

[SCREEN01]
