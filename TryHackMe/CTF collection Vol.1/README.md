# [CTF collection Vol.1](https://tryhackme.com/room/ctfcollectionvol1)

## Sharpening up your CTF skill with the collection. The first volume is designed for beginner.

### Can you decode the following? VEhNe2p1NTdfZDNjMGQzXzdoM19iNDUzfQ==

1. This is Base64, we can decode this in [CyberChef](https://gchq.github.io/CyberChef/)

![SCREEN01](https://github.com/user-attachments/assets/76d46adb-568c-4c6d-84a7-e0a98a8c74fb)

---

### Meta! meta! meta! meta...

1. Download the image attached and check **Owner Name** with `exiftool`

```bash
exiftool Find_me_1577975566801.jpg
```

![SCREEN02](https://github.com/user-attachments/assets/9302190c-14bb-4ffd-ba79-45812aed728a)

---

### Something is hiding. That's all you need to know.

1. Download the image attached, and extract the flag with `steghide`

```bash
steghide extract -sf Extinction_1577976250757.jpg
```

![SCREEN03](https://github.com/user-attachments/assets/a2daf859-2f15-42ea-bfa2-2af1463a8c13)

---

### Huh, where is the flag?

1. Highlight the text, or inspect in devTools

![SCREEN04](https://github.com/user-attachments/assets/4314fdea-c03c-4ed4-b9b4-3ee7a3ca594e)

---

###

1. Use [QR Code Scanner Online](https://scanqr.org/) to scan this QR code

![SCREEN05](https://github.com/user-attachments/assets/078f4603-7a9d-41fa-9c70-62577ad80442)

---

### Both works, it's all up to you.

1. Read content with `cat` or use `strings` to find the flag

```bash
cat hello_1577977122465.hello
strings hello_1577977122465.hello
```

![SCREEN06](https://github.com/user-attachments/assets/3da9b675-91a7-44c7-aeca-83ec05ec2996)

---

### Can you decode it? 3agrSy1CewF9v8ukcSkPSYm3oKUoByUpKG4L

1. This is Base58, we can decode this in [CyberChef](https://gchq.github.io/CyberChef/)

![SCREEN07](https://github.com/user-attachments/assets/25fcdeaf-ca12-4101-bdea-c5ce31a32749)

---

### Solve this MAF{atbe_max_vtxltk}

1. This is ROT13 with 7 rotations, we can decode this in [CyberChef](https://gchq.github.io/CyberChef/)

![SCREEN08](https://github.com/user-attachments/assets/1d55be5c-2b1b-44fb-b805-96d7b9a669c6)

---

### No downloadable file, no ciphered or encoded text

1. Inspect the element in devTools

![SCREEN09](https://github.com/user-attachments/assets/70982dc8-caf4-45fe-9757-24cb5312998f)

---

### Can you fix it?

1. Get image hex data

```bash
xxd --plain spoil_1577979329740.png
```

2. Put the output in [CyberChef](https://gchq.github.io/CyberChef/), select `Render Image` and replace first 8 characters with **89504e47** (PNG magic numbers) to fix the image

![SCREEN10](https://github.com/user-attachments/assets/3e9e80be-afae-4ba4-94f7-837d9a5b3865)

---

### Some hidden flag inside Tryhackme social account

1. Search for `site:'reddit.com' & intext:"THM"` it should not be far
   - In case you can not find it, this is the [post](https://www.reddit.com/r/tryhackme/comments/eizxaq/new_room_coming_soon/)

![SCREEN11](https://github.com/user-attachments/assets/56982f48-76d7-464d-b119-8a7b25900d39)

---

### What is this?

`++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>++++++++++++++.------------.+++++.>+++++++++++++++++++++++.<<++++++++++++++++++.>>-------------------.---------.++++++++++++++.++++++++++++.<++++++++++++++++++.+++++++++.<+++.+.>----.>++++.`

1. Go to the [dCode](https://www.dcode.fr/brainfuck-language) and put the strings there

![SCREEN12](https://github.com/user-attachments/assets/86051e78-5b66-40c6-9029-058ae013d4d4)

---

### Exclusive strings for everyone!

1. Got to any [online XOR calculator](https://xor.pw/#)

![SCREEN13](https://github.com/user-attachments/assets/f457b31f-b264-49a0-90e4-ff2e6e42bb01)

---

### Binary walk

1. Use `binwalk` on the image. Flag will be exported to `hello_there.txt`

```bash
binwalk -e hell_1578018688127.jpg
```

![SCREEN14](https://github.com/user-attachments/assets/81437073-8ade-42a6-9b57-af058b76bd8b)

(If binwalk throws errors on THM AttackBox use command below)

```bash
sed -i 's/CS_ARCH_ARM64/CS_ARCH_AARCH64/g' /usr/lib/python3/dist-packages/binwalk/modules/disasm.py
```

---

### There is something lurking in the dark

1. Use `stegsolve`, the flag is visible on **Blue plane 1**

```bash
wget http://www.caesum.com/handbook/Stegsolve.jar -O stegsolve.jar
java -jar stegsolve.jar
```

![SCREEN15](https://github.com/user-attachments/assets/8202a34a-797d-4378-81c7-a462bcf9a8ee)

---

### A sounding QR

1. The [QR code](https://scanqr.org/) gives us [soundcloud link](https://soundcloud.com/user-86667759/thm-ctf-vol1) with audio of spelled out flag

![SCREEN16](https://github.com/user-attachments/assets/1f86a744-69f9-4b66-843e-f788e2968bd0)

---

### Dig up the past

1. Got to the [Wayback Machine](https://web.archive.org/web/20200102131252/https://www.embeddedhacker.com/) and select 2 january 2020

![SCREEN17](https://github.com/user-attachments/assets/f5b1ead6-d7ba-4a2f-8b24-776fa907b681)

---

### Uncrackable

MYKAHODTQ{RVG_YVGGK_FAL_WXF}

1. This is Vigenère cipher. Since we know flag format, key can be easly found even by brute forcing in [CyberChef](https://gchq.github.io/CyberChef/)

![SCREEN18](https://github.com/user-attachments/assets/439164e8-1fc9-4699-b615-461ff6647c20)

---

### Decode the following text:

581695969015253365094191591547859387620042736036246486373595515576333693

1. Convert it from [Decimal to Hex](https://www.rapidtables.com/convert/number/decimal-to-hex.html), and then from [Hex to String](https://www.rapidtables.com/convert/number/hex-to-ascii.html)

![SCREEN19](https://github.com/user-attachments/assets/4323d654-3ec3-42f6-9912-886fd53d0e58)
![SCREEN20](https://github.com/user-attachments/assets/882a47f9-6bd4-4bc2-a451-7f2c348cd3d5)

---

### Read the packet

1. Load the `.pcapng` file into wireshark, search for `/flag`, and then right click `Follow` -> `HTTP Stream` to get the flag

![SCREEN21](https://github.com/user-attachments/assets/e9560749-c253-452e-80c5-6735fefa7be5)
