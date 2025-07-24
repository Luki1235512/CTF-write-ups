# [The Impossible Challenge](https://tryhackme.com/room/theimpossiblechallenge)

# Submit the flag

### flag is in the format THM{}

_Hmm, I'm thinking Steg_

1. The challenge description contains an encoded message that requires a multi-stage decryption process
   - **Cipher sequence:** ROT13 → ROT47 → Hexadecimal decoding → Base64 decoding
   - **Result:** The decoded message reveals: `It's inside the text, in front of your eyes!`

[SCREEN02]

2. The description contains what appears to be standard text ("Hmm"), which upon closer inspection contains hidden data encoded using unicode steganography techniques. Input the text into the [unicode steganography decoder](https://330k.github.io/misc_tools/unicode_steganography.html)
   - **Result:** The hidden password extracted is: `hahaezpz`

[SCREEN01]

3. The challenge provides a password-protected ZIP archive containing the flag file. Extract `flag.txt` using password: `hahaezpz`.

[SCREEN03]
