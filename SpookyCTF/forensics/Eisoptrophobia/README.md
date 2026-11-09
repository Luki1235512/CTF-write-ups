# [Eisoptrophobia](https://spooky.ctfd.io/challenges#Eisoptrophobia-22)

Something strange is happening in the mirror. What appears to be an ordinary reflection may be hiding something more than meets the eye. Look closely, investigate the image, and uncover what lies beneath the surface.

1. Run `zsteg` against the PNG to check every LSB and bit-plane combination:

```bash
zsteg -a Eisoptrophobiactf.png
```

2. One line stands out. The LSBs of the R, G and B channels, read in row order, give a base64 string:

```
b1,rgb,lsb,xy       .. text: "c3Bvb2t5e3RoZV9yZWZsZWN0aW9uX25ldmVyX2xpZXN9"
```

3. Decode it:

```bash
echo 'c3Bvb2t5e3RoZV9yZWZsZWN0aW9uX25ldmVyX2xpZXN9' | base64 -d
```

[SCREEN01]
