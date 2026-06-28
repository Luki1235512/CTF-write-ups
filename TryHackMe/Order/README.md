# [Order](https://tryhackme.com/room/hfb1order)

## Perform a known-plaintext attack to recover a repeating-key XOR key and decrypt a hidden message.

We intercepted one of Cipher's messages containing their next target. They encrypted their message using a repeating-key XOR cipher. However, they made a critical error—every message always starts with the header:

`ORDER:`
Can you help void decrypt the message and determine their next target?
Here is the message we intercepted:

`1c1c01041963730f31352a3a386e24356b3d32392b6f6b0d323c22243f6373`

`1a0d0c302d3b2b1a292a3a38282c2f222d2a112d282c31202d2d2e24352e60`

### What is the flag?

#### Understanding the cipher

A **repeating-key XOR cipher** works by XOR-ing each byte of the plaintext with a byte from the key, cycling through the key repeatedly. XOR has a critical property that makes this cipher vulnerable: **it is its own inverse**. So if we know any portion of the plaintext, we can recover the corresponding bytes of the key directly.

#### Exploiting the known plaintext

We are told that **every message begins with the header `ORDER:`**, which is 6 bytes long. This is our known plaintext. By XOR-ing the first 6 bytes of the ciphertext with the ASCII bytes of `ORDER:`, we recover the first 6 bytes of the key and since the key repeats, those 6 bytes are the full key.

#### Solution

1. **Concatenate** the two hex strings into one continuous ciphertext.
2. **Decode** the hex string into raw bytes.
3. **Recover the key** by XOR-ing the first `len("ORDER:")` = 6 bytes of the ciphertext with the known plaintext header.
4. **Decrypt** the full ciphertext by XOR-ing every byte with the corresponding key byte.

```py
cipher_hex = "1c1c01041963730f31352a3a386e24356b3d32392b6f6b0d323c22243f63731a0d0c302d3b2b1a292a3a38282c2f222d2a112d282c31202d2d2e24352e60"
ciphertext = bytes.fromhex(cipher_hex)
header = b"ORDER:"

key = bytes(c ^ p for c, p in zip(ciphertext, header))

plaintext = bytes(ciphertext[i] ^ key[i % len(key)] for i in range(len(ciphertext)))
print(plaintext.decode())
```

<img width="525" height="58" alt="SCREEN01" src="https://github.com/user-attachments/assets/26247af8-9f94-4332-9792-163bd84b4149" />
