# [Stash the Hash](https://spooky.ctfd.io/challenges#Stash%20the%20Hash-16)

A file was found that seems to contain detailed information about an infamous and high-profile cryptid. There account is known to contain forbidden supernatural information. If you can find out what their password is you may be able to obtain this information for yourself

**1. Crack the password hash**

`UserAccountInfo.txt` has a 32-character hex password, which is an unsalted MD5. Crack it with rockyou:

```bash
echo ad7bcd7ab3cc113f496e1840bc582bbd > hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Result: `newjerseydevils`

**2. Extract the archive**

The password is also the password for the 7z file:

```bash
7z x secretKnowledgeFolder.7z -p'newjerseydevils' -o./out
```

This gives `out/secretKnowledge.png`.

**3. Pull out the LSB planes**

The PNG is a 700x700 RGBA image. Alpha is constant (255), so the hidden data is in the color channels. Dump the least significant bit of each channel as its own image:

```python
from PIL import Image
im = Image.open("secretKnowledge.png").convert("RGBA")
for n, c in zip("rgba", im.split()):
    c.point(lambda v: 255 * (v & 1)).save(f"lsb_{n}.png")
```

Open `lsb_r.png`. The flag is readable in the red channel's LSB plane.

[SCREEN01]
