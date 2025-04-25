# [CTF collection Vol.1](https://tryhackme.com/room/ctfcollectionvol1)

## Sharpening up your CTF skill with the collection. The first volume is designed for beginner.

### Can you decode the following? VEhNe2p1NTdfZDNjMGQzXzdoM19iNDUzfQ==

1. This is Base64, we can decode this in [CyberChef](https://gchq.github.io/CyberChef/)

[SCREEN01]

### Meta! meta! meta! meta...

1. Download the image attached and check **Owner Name** with `exiftool`

```bash
exiftool Find_me_1577975566801.jpg
```

[SCREEN02]

### Something is hiding. That's all you need to know.

1. Download the image attached, and extract the flag with `steghide`

```bash
steghide extract -sf Extinction_1577976250757.jpg
```

[SCREEN03]

### Huh, where is the flag?

1. Highlight the text, or inspect in devTools

[SCREEN04]

###

1. Use [QR Code Scanner Online](https://scanqr.org/) to scan this QR code

[SCREEN05]

### Both works, it's all up to you.

1. Read content with `cat` or use `strings` to find the flag

```bash
cat hello_1577977122465.hello
strings hello_1577977122465.hello
```

[SCREEN06]

### Can you decode it? 3agrSy1CewF9v8ukcSkPSYm3oKUoByUpKG4L

1. This is Base58, we can decode this in [CyberChef](https://gchq.github.io/CyberChef/)

[SCREEN07]

### Solve this MAF{atbe_max_vtxltk}

1. This is ROT13 with 7 rotations, we can decode this in [CyberChef](https://gchq.github.io/CyberChef/)

[SCREEN08]

### No downloadable file, no ciphered or encoded text

1. Inspect the element in devTools

[SCREEN09]

### Can you fix it?

1. Get image hex data

```bash
xxd --plain spoil_1577979329740.png
```

2. Put the output in [CyberChef](https://gchq.github.io/CyberChef/), select `Render Image` and replace first 8 characters with **89504e47** (PNG magic numbers) to fix the image

[SCREEN10]

### Some hidden flag inside Tryhackme social account

1. Search for `site:'reddit.com' & intext:"THM"` it should not be far
   - In case you can not find it, this is the [post](https://www.reddit.com/r/tryhackme/comments/eizxaq/new_room_coming_soon/)

[SCREEN11]

### What is this?

`++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>++++++++++++++.------------.+++++.>+++++++++++++++++++++++.<<++++++++++++++++++.>>-------------------.---------.++++++++++++++.++++++++++++.<++++++++++++++++++.+++++++++.<+++.+.>----.>++++.`

1. Go to the [dCode](https://www.dcode.fr/brainfuck-language) and put the strings there

[SCREEN12]

### Exclusive strings for everyone!

1. Got to any [online XOR calculator](https://xor.pw/#)

[SCREEN13]

### Binary walk

1. Use `binwalk` on the image. Flag will be exported to `hello_there.txt`

```bash
binwalk -e hell_1578018688127.jpg
```

[SCREEN14]

(If binwalk throws errors on THM AttackBox use command below)

```bash
sed -i 's/CS_ARCH_ARM64/CS_ARCH_AARCH64/g' /usr/lib/python3/dist-packages/binwalk/modules/disasm.py
```

### There is something lurking in the dark

1. Use `stegsolve`, the flag is visible on **Blue plane 1**

```bash
wget http://www.caesum.com/handbook/Stegsolve.jar -O stegsolve.jar
java -jar stegsolve.jar
```

[SCREEN15]

### A sounding QR

1. The [QR code](https://scanqr.org/) gives us [soundcloud link](https://soundcloud.com/user-86667759/thm-ctf-vol1) with audio of spelled out flag

[SCREEN16]

### Dig up the past

1. Got to the [Wayback Machine](https://web.archive.org/web/20200102131252/https://www.embeddedhacker.com/) and select 2 january 2020

[SCREEN17]

### Uncrackable

MYKAHODTQ{RVG_YVGGK_FAL_WXF}

1. This is Vigenère cipher. Since we know flag format, key can be easly found even by brute forcing in [CyberChef](https://gchq.github.io/CyberChef/)

[SCREEN18]

### Decode the following text:

581695969015253365094191591547859387620042736036246486373595515576333693

1. Convert it from [Decimal to Hex](https://www.rapidtables.com/convert/number/decimal-to-hex.html), and then from [Hex to String](https://www.rapidtables.com/convert/number/hex-to-ascii.html)

[SCREEN19]
[SCREEN20]

### Read the packet

1. Load the `.pcapng` file into wireshark, search for `/flag`, and then right click `Follow` -> `HTTP Stream` to get the flag

[SCREEN21]
