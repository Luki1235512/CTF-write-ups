# [The Impossible Challenge](https://tryhackme.com/room/theimpossiblechallenge)

# Submit the flag

### flag is in the format THM{}

_Hmm, I'm thinking Steg_

1. The challenge description contains an encoded message that requires a multi-stage decryption process
   - **Cipher sequence:** ROT13 → ROT47 → Hexadecimal decoding → Base64 decoding
   - **Result:** The decoded message reveals: `It's inside the text, in front of your eyes!`

<img width="1678" height="530" alt="SCREEN02" src="https://github.com/user-attachments/assets/8a107b79-807f-405d-bd78-3fc7f115bab5" />

2. The description contains what appears to be standard text ("Hmm"), which upon closer inspection contains hidden data encoded using unicode steganography techniques. Input the text into the [unicode steganography decoder](https://330k.github.io/misc_tools/unicode_steganography.html)
   - **Result:** The hidden password extracted is: `hahaezpz`

<img width="1358" height="446" alt="SCREEN01" src="https://github.com/user-attachments/assets/ad4124a5-3eff-4e3c-9401-126451ae43dd" />

3. The challenge provides a password-protected ZIP archive containing the flag file. Extract `flag.txt` using password: `hahaezpz`.

<img width="652" height="125" alt="SCREEN03" src="https://github.com/user-attachments/assets/44aec7ea-4095-415f-a28c-9a08bfabf289" />
