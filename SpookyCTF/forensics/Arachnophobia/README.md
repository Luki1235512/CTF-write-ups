# [Arachnophobia](https://spooky.ctfd.io/challenges#Arachnophobia-6)

Something about the SpookyCTF logo feels off… like something's tangled in the webs

1. `exiftool tangled.png` warns `Trailer data after PNG IEND chunk`, so something is appended to the image.
2. `binwalk tangled.png` shows a zip archive at offset `0xEE80D`, right after `IEND`.
3. Extract it and read the flag:

```bash
binwalk -e tangled.png
cat _tangled.png.extracted/flag.txt
```

<img width="795" height="236" alt="SCREEN01" src="https://github.com/user-attachments/assets/c869a698-15f9-4bac-ab0c-87fcea0674d8" />
