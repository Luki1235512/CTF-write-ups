# [Xenoglossophobia](https://spooky.ctfd.io/challenges#Xenoglossophobia-4)

so many languages ... so many ways to interpret data ... please help me! looking at this challenge makes me feel like I am cutting onions right now 🤮

## 1. Look at the log

```bash
ls -la a00afa0c-serial-capture.log
file a00afa0c-serial-capture.log
wc -l a00afa0c-serial-capture.log
head -c 1500 a00afa0c-serial-capture.log
tail -c 800 a00afa0c-serial-capture.log
```

The file is plain ASCII text, about 2.2 MB and 41,779 lines. It is a hexdump of a serial line. Each record has a direction (`<` or `>`), a timestamp, a length, a byte range, and then hex plus an ASCII column:

```
< 2026/10/07 12:17:40.615989  length=1 from=0 to=0
 15                                               .
--
> 2026/10/07 12:17:49.550551  length=132 from=0 to=131
 01 01 fe ff d8 ff e0 00 10 4a 46 49 46 00 01 01  .........JFIF...
```

Things to notice:

- `>` records are 132 bytes and start with `01 01 fe`. That is an XMODEM (checksum variant) frame: `SOH (01) | block number | 255 - block number | 128 data bytes | 8 bit sum`. 4 + 128 = 132.
- `<` records are single bytes: `15` is NAK (the receiver asking to start), `06` is ACK, and `04` is EOT (sent by the sender at the end).
- The data after `01 01 fe` begins with `ff d8 ff e0 ... JFIF`, which is a JPEG header.

The last records of the capture show two EOT (`04`) bytes and a second NAK at the start of a second transfer, so there are two files in the capture. The block number resets to 1 at the second transfer.

## 2. Reassemble both files

`extract.py` reads the log, keeps only the 132 byte `>` records, parses the hex column (characters 1 to 48 of each continuation line, which avoids the ASCII column), strips the 3 byte header and the checksum byte from each frame, and concatenates the 128 byte payloads. The second transfer is found where the block number goes back to 1 (frame 2948). XMODEM pads the last block with `0x1A`, so that padding is stripped.

```python
import re, sys

log = sys.argv[1]
hdr = re.compile(r'^> \S+ \S+\s+length=132 ')
frames = []
cur = None
for line in open(log):
    line = line.rstrip('\n')
    if line.startswith('>') or line.startswith('<'):
        cur = bytearray() if hdr.match(line) else None
        if cur is not None:
            frames.append(cur)
    elif cur is not None and line.startswith(' '):
        for tok in line[1:49].split():
            cur.append(int(tok, 16))

assert all(len(f) == 132 and f[0] == 1 for f in frames)
data = [bytes(f[3:131]) for f in frames]

split = [i for i, f in enumerate(frames) if f[1] == 1 and i > 0 and frames[i-1][1] != 0][0]
print("frames:", len(frames), "second transfer starts at frame", split)

open('image.jpg', 'wb').write(b''.join(data[:split]).rstrip(b'\x1a'))
open('note.txt', 'wb').write(b''.join(data[split:]).rstrip(b'\x1a'))
```

Run it and check the results:

```bash
python3 -I extract.py a00afa0c-serial-capture.log
file image.jpg note.txt
cat note.txt
```

Output:

```
frames: 2950 second transfer starts at frame 2948
image.jpg: JPEG image data, JFIF standard 1.01, ... baseline, precision 8, 2560x1440, components 3
note.txt:  ASCII text

Ut clavem invenias ad plura de "loco" cognoscenda, intellegere debes quid imago tibi significet.

Est id quod "supermercado" vocant!
```

The second file is Latin. Roughly: "To find the key to learn more about the 'location', you must understand what the image means to you. It is what they call 'supermercado'!"

## 3. Use the key on the image

Open `image.jpg`. It is a screenshot of the game Supermarket Simulator (shelves, a shopper, a cardboard box of cereal). The note says the image means "supermercado" (Spanish for supermarket), and calls it the key. A JPEG that comes with a "key" is a good candidate for steghide, which embeds data in JPEGs behind a passphrase.

Check if the image has an embedded payload slot:

```bash
steghide info image.jpg -p ""
```

`steghide info` reports the format and capacity (about 20.6 KB). With an empty passphrase it cannot extract anything, so try the word from the note:

```bash
steghide extract -sf image.jpg -p supermercado -xf hidden.txt -f
```

Output: `wrote extracted data to "hidden.txt".`

## 4. Decode the hidden file

```bash
cat hidden.txt
```

The content is Portuguese text written as `\xNN` escapes (Latin-1 bytes such as `\xe9` for `é`), and the last part of the flag is not hex but space separated decimal ASCII:

```
\x4d\x65\x75\x20\x61\x6d\x69\x67\x6f\x21 ... \x73\x70\x6f\x6f\x6b\x79\x7b\x6d\x75\x31\x74\x69\x70\x6c\x33\x5f 108 64 110 103 117 64 103 101 115 125
```

`decode.py` decodes the hex escapes as Latin-1, decodes the trailing decimal numbers as ASCII, and joins the last word of the first part with the second part:

```python
import re
s = open('hidden.txt', 'rb').read().decode('latin-1')
m = re.match(r'((?:\\x[0-9a-f]{2})+)\s*(.*)$', s, re.S)
part1 = bytes(int(h, 16) for h in re.findall(r'\\x([0-9a-f]{2})', m.group(1))).decode('latin-1')
part2 = bytes(int(n) for n in m.group(2).split()).decode()
print(part1)
print(part2)
print(part1.split('\n')[-1].split()[-1] + part2)
```

```bash
python3 -I decode.py
```

Output:

```
Meu amigo! Parabéns por encontrar esta comunicação! Eu sei que pode ter levado um tempo para chegar até aqui.

O bandeira é ...
```

The Portuguese says "My friend! Congratulations on finding this message! I know it may have taken a while to get here. The flag is ...".
