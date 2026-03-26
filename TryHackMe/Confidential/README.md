# [Confidential](https://tryhackme.com/room/confidential)

## We got our hands on a confidential case file from some self-declared "black hat hackers"... it looks like they have a secret invite code.

We got our hands on a confidential case file from some self-declared "black hat hackers"... it looks like they have a secret invite code available within a QR code, but it's covered by some image in this PDF! If we want to thwart whatever it is they are planning, we need your help to uncover what that QR code says!

1. Extract all images from the PDF using below command or just right click on the document and `Save Image As`.

```bash
pdfimages -j Repdf.pdf image
```

2. Use QR code scanner (for example `https://scanqr.org/`) to read the flag.

<img width="1254" height="363" alt="SCREEN01" src="https://github.com/user-attachments/assets/50c3c0f1-ab6d-4852-8b39-ac053bd8c605" />
