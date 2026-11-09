# [The Haunted Portrait](https://spooky.ctfd.io/challenges#The%20Haunted%20Portrait-1)

An old portrait was recently discovered in a storage room at NJIT just before Halloween. Students claim the painting changes when viewed at night, and some believe a secret message is hidden inside it. The university's security team examined the image but couldn't find anything unusual. Can you uncover what the portrait has been hiding all these years?

1. Render the blue LSB plane as an image

```python
from PIL import Image; import numpy as np
a = np.array(Image.open('haunted_portrait.png').convert('RGB'))
Image.fromarray(((a[:,:,2] & 1) * 255).astype(np.uint8)).save('B_bit0.png')"
```

Open `B_bit0.png`. The text is readable in the image. In stegsolve, the same view is "Blue plane 0".

2. Decode the ROT13 text

```bash
echo 'TEXT_FROM_IMAGE' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Why this works

The blue channel’s LSB was 0 for about 96% of pixels (530,355 even values against 19,845 odd). A natural photo is close to 50/50, so that lopsided plane meant something had been drawn into it. Byte-stream extractors like zsteg miss it because the message is an image of text, not encoded bytes.
