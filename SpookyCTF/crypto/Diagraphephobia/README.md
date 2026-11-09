# [Diagraphephobia](https://spooky.ctfd.io/challenges#Diagraphephobia-32)

The Federation has found a redacted file from spying on the Klingon. They want to know the contents of it. Can you find it?

1. Inspect the PDF metadata. `pdfinfo` or `exiftool` shows `Subject: lojmIt yIpoSmoH!`, which is Klingon for "open the door". This is the key.

2. Extract the page text. `pdftotext Diagraphephobia.pdf -` returns a Klingon word and a base64 blob:

```bash
pdftotext Diagraphephobia.pdf - | grep -E '^[A-Za-z0-9+/=]{20,}$' > b64.txt
```

```
B7o731QSZYJtbxIHjz045jhZoAhjG/MHqir4PZANSCWd75Jp/KGdAevRj2JOBEN3eME
W9Sto9ek0bGum9jXYPHiQ/3xQM8ZTrrB6a2OZJNAYCgFizrrNO4enzaXB2ZAGdN+fB
KQPPe530ZxrXlWBTV2QCyqJH8BmHGXKMU3XASpqaWeTXZzvhx1FJOkhV2pw2bif
mMo0Pk4i3XznS1UYsQ==
```

3. Base64-decode it into `ct.bin`. It is 160 bytes, a multiple of 16, which points to a block cipher.

```bash
base64 -d b64.txt > ct.bin
```

4. Decrypt with AES-128-ECB. Use the 16-byte subject string as the raw key, with no hashing:

```bash
printf 'lojmIt yIpoSmoH!' | xxd -p
# 6c6f6a6d497420794970 6f536d6f4821
openssl enc -d -aes-128-ecb -K 6c6f6a6d4974207949706f536d6f4821 -in ct.bin
```

5. The plaintext is a Klingon message with the flag inside:

<img width="1064" height="114" alt="SCREEN01" src="https://github.com/user-attachments/assets/5ee86377-59ee-4932-b3ca-9e452810b0eb" />
