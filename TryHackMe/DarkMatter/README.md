# [DarkMatter](https://tryhackme.com/room/hfb1darkmatter)

## Practice how to exploit a weak RSA implementation to recover the private key and decrypt a ransomware-encrypted files.

The Hackfinitiy high school has been hit by DarkInjector's ransomware, and some of its critical files have been encrypted. We need you and Void to use your crypto skills to find the RSA private key and restore the files. After some research and reverse engineering, you discover they have forgotten to remove some debugging from their code. The ransomware saves this data to the tmp directory.

Can you find the RSA private key?

### What is the flag?

1. **Enumerate the `/tmp` directory to find any files left behind by the ransomware.**

```bash
ls /tmp/
```

**Results:**

```
dock-replace.log
encrypted_aes_key.bin
public_key.txt
```

Three files were left behind. The most interesting one is `public_key.txt` - RSA public key. The `encrypted_aes_key.bin` is likely the AES session key encrypted with RSA, which we'll need to recover.

2. **Read the public key to extract the RSA parameters `n` and `e`.**

```bash
cat /tmp/public_key.txt
```

```
n=340282366920938460843936948965011886881
e=65537
```

The public key consists of:

- `n` - the modulus, a product of two secret primes `p` and `q`
- `e` - the public exponent

Notice that `n` is only **128 bits** long. Modern RSA uses 2048-4096 bits. A modulus this small can be factored in seconds, which is the core vulnerability we'll exploit. If we can factor `n` into `p` and `q`, we can reconstruct the private key.

3. **Factor `n` and compute the RSA private key `d`.**

```py
from Crypto.Util.number import *
import sympy

n = 340282366920938460843936948965011886881
e = 65537

factors = sympy.factorint(n)
p, q = factors.keys()

phi_n = (p - 1) * (q - 1)

d = inverse(e, phi_n)

print(f"d: {d}")
```

- `sympy.factorint(n)` factors `n` into its prime components `p` and `q`
- Euler's totient `φ(n) = (p−1)(q−1)` tells us how many numbers are coprime to `n`
- The private exponent `d` satisfies `e·d ≡ 1 (mod φ(n))` computed via modular inverse
- With `d` in hand, we can decrypt anything encrypted with the corresponding public key

4. **Use the recovered private key `d` to decrypt the encrypted AES key.**\
   When prompted by the challenge interface with `Enter the decryption key:`, paste the value of `d` printed by the script. The system uses your recovered private key to RSA-decrypt `encrypted_aes_key.bin`, revealing the AES session key that was used to encrypt the school's files. This is standard **hybrid encryption**: RSA protects the AES key, and AES encrypts the actual data.

5. **Open `student_grades.docx` to retrieve the flag.**\
   With the AES key now recovered, the ransomware-encrypted file `student_grades.docx` can be decrypted and opened. Inside, you'll find the flag.

[SCREEN01]
