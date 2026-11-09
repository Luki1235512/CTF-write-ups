# [Peering Eyes](https://spooky.ctfd.io/challenges#Peering%20Eyes%20-15)

recently EYE have been feeling super paranoid using my computer. almost like someone is watching. Can you make sure EYE am not going crazy!? i swear it feels like people are just taking pictures of me...

## Setup

Install Volatility 3 and the Pillow and numpy packages that the recovery script needs. The kernel is `7.1.5+kali-amd64`, which has no ISF in the public symbol packs, so the provided `symbols.json.xz` has to be placed in a symbol directory.

```bash
python3 -m venv venv
./venv/bin/pip install volatility3 pillow numpy
mkdir -p sym/linux
cp symbols.json.xz sym/linux/
```

## 1. Find the suspicious process

```bash
./venv/bin/vol -q -s sym -f spooked2.lime linux.pslist.PsList > pslist.txt
./venv/bin/vol -q -s sym -f spooked2.lime linux.psaux.PsAux  > psaux.txt
grep -i python psaux.txt
```

The relevant line is PID 28906, parent 2311 (a `zsh`), command line `python3 EYEsee.py`. The name matches the challenge text ("EYE", "taking pictures").

To confirm it uses the camera and the screen, list its memory mappings. The mappings contain `cv2`, `libavdevice`, `PIL` and `numpy` from `/home/kali/.venv`.

```bash
./venv/bin/vol -q -s sym -f spooked2.lime linux.proc.Maps --pid 28906 > maps28906.txt
grep -E 'cv2|libavdevice|PIL|numpy' maps28906.txt | head
```

## 2. First half of the flag

A plain search for `spooky{...}` with a closing brace finds nothing, because the flag is split. Search for the prefix alone, with byte offsets:

```bash
grep -aob 'spooky{[^"]*' spooked2.lime | head
```

Output:

```
578695889:spooky{th3y_ar3_
578695960:spooky{th3y_ar3_
```

Look at the surrounding bytes of the first hit to see what this memory is:

```bash
dd if=spooked2.lime bs=1 skip=$((0x227e32d1 - 600)) count=1200 2>/dev/null | strings -n 4
```

(`0x227e32d1` is the same offset as 578695889 above, written in hex.) The neighbours are `Error: Failed to grab frame.`, `eye see you`, `%Y%m%d_%H%M%S`, `screenshot` and `screenshots/latest_screenshot_`. These are the constants of the `EYEsee.py` code object, so `spooky{th3y_ar3_` is a string literal from the script, and the rest of the flag is not in the script.

## 3. Second half of the flag

The script grabs a webcam frame, shows it in an OpenCV window, takes a screenshot, and sleeps 300 seconds (the traceback in the terminal scrollback shows `time.sleep(300)` at line 33). The note therefore only exists as pixels.

### 3.1 What does not work

Carving JPEG or PNG signatures from the raw dump gives nothing useful (the only hits are icons and corrupted fragments), and the page-cache plugins do not return the screenshot files. The process heap contains screenshots, but the part that holds the webcam window is overwritten with unrelated pages in the lower rows.

### 3.2 What works

The OpenCV Qt window is a 32-bit BGRA buffer, so every byte at offset 3 of each 4-byte pixel is `0xff`. Pages of such data can be found by scanning physical memory for pages whose alpha bytes are all `0xff` and which are not flat. Physically contiguous runs of those pages are image buffers. A run of image data has a fixed row width, which shows up as the row width that minimizes the difference between a pixel and the pixel one row below it.

Save the script below as `recover_note.py`. It does these steps:

1. Parses the LiME range headers (magic `0x4C694D45`, then version, start and end addresses).
2. Marks every 4 KiB page of the main RAM range that is BGRA-like and textured.
3. Collects contiguous runs of at least 100 such pages.
4. For each run, tries every row width from 200 to 1924 pixels and prints the best one.
5. Takes the run with row width 650, extends it over adjacent flat BGRA pages, and writes it as a PNG, rolled by 72 pixels so the window border is on the left edge.

```python
import struct, sys, mmap
import numpy as np
from PIL import Image

lime_path, out_path = sys.argv[1], sys.argv[2]
f = open(lime_path, 'rb')
mm = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)

pos, ranges = 0, []
while pos < len(mm):
    magic, ver, s, e = struct.unpack('<IIQQ', mm[pos:pos+24])
    assert magic == 0x4C694D45
    ranges.append((s, e, pos + 32))
    pos += 32 + (e - s + 1)
print('ranges:', [(hex(s), hex(e)) for s, e, _ in ranges])

def page(pa):
    for s, e, fo in ranges:
        if s <= pa <= e:
            return np.frombuffer(mm[fo + pa - s: fo + pa - s + 4096], dtype=np.uint8)

s, e, fo = ranges[1]
npg = (e - s + 1) // 4096
isbgra = np.zeros(npg, bool)
textured = np.zeros(npg, bool)
for k in range(npg):
    p = np.frombuffer(mm[fo + k*4096: fo + (k+1)*4096], dtype=np.uint8)
    if (p[3::4] == 255).all() and p.any():
        isbgra[k] = True
        textured[k] = len(set(p[::7].tolist())) > 40

runs, k = [], 0
while k < npg:
    if textured[k]:
        j = k
        while j < npg and textured[j]: j += 1
        runs.append((k, j - k)); k = j
    else:
        k += 1
runs = [r for r in runs if r[1] >= 100]
print('long textured runs:', [(hex(s + a*4096), n) for a, n in runs])

best = None
for a, n in runs:
    px = np.frombuffer(mm[fo + a*4096: fo + (a+n)*4096], dtype=np.uint8).reshape(-1, 4)[:, :3].astype(np.int16)
    sc = []
    for w in range(200, 1925):
        m = len(px) - w
        sc.append((np.abs(px[:m:3] - px[w:w+m:3]).mean(), w))
    sc.sort()
    print(hex(s + a*4096), n, 'pages, best width', sc[0][1], 'diff', round(float(sc[0][0]), 2))
    if sc[0][1] == 650 and (best is None or sc[0][0] < best[2]): best = (a, n, sc[0][0])

a, n, _ = best
lo, hi = a, a + n
while lo > 0 and isbgra[lo-1]: lo -= 1
while hi < npg and isbgra[hi]: hi += 1
px = np.frombuffer(mm[fo + lo*4096: fo + hi*4096], dtype=np.uint8).reshape(-1, 4)[:, :3]
h = len(px) // 650
img = px[:h*650].reshape(h, 650, 3)[:, :, ::-1]
img = np.roll(img, -72, axis=1)
Image.fromarray(np.ascontiguousarray(img)).save(out_path)
print('wrote', out_path, img.shape, 'physical start', hex(s + lo*4096))
```

Run it. It takes a few minutes because it reads every page of the 1 GB file.

```bash
./venv/bin/python -I recover_note.py spooked2.lime note_frame.png
```

The relevant lines of output:

```
0x88c6000 314 pages, best width 1920 diff 0.48
0x9800000 184 pages, best width 650 diff 2.87
0xeb87000 121 pages, best width 650 diff 3.9
0x17739000 199 pages, best width 1920 diff 0.73
wrote note_frame.png (311, 650, 3) physical start 0x9800000
```

Runs with width 1920 are full desktop screenshots (the 1920 pixel wide screen, mostly wallpaper). The two runs with width 650 are the OpenCV window. The one at `0x9800000` is the bottom of a frame in which the sheet is held high and fully in view. The one at `0xeb87000` is the top of a different frame (it includes the Qt toolbar) and does not show the text.

### 3.3 Read the note

Open `note_frame.png` in any image viewer. It shows a hand holding a squared sheet of paper with `w2tch1ng}` written on it. To make sure of the second character (2, a or 4), crop and enlarge the text:

```python
from PIL import Image, ImageEnhance
im = Image.open('note_frame.png')
c = im.crop((360, 40, 600, 100)).resize((1440, 360), Image.LANCZOS)
ImageEnhance.Contrast(c).enhance(2.2).save('note_zoom.png')
```

The enlarged image reads `w2tch1ng` followed by a tall curly `}` at the right edge of the paper. The second character is a 2 with a flat base, and the paper has nothing written to the left of the `w`.

## 4. Assemble

```
spooky{th3y_ar3_   (string constant in the EYEsee.py process, section 2)
w2tch1ng}          (handwritten note in the webcam window buffer, section 3)
```
