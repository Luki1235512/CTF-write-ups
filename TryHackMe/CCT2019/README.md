# [CCT2019](https://tryhackme.com/room/cct2019)

## Legacy challenges from the US Navy Cyber Competition Team 2019 Assessment sponsored by US TENTH Fleet

## This is a pcap-focused challenge originally created for the U.S. Navy Cyber Competition Team 2019 Assessment. Although the assessment is over, the created challenges are provided for community consumption here.

## If you find the right clues, they will guide you to the next step. I did include some red herrings in this challenge, but you can stay on track by focusing on pcap-related skills.

_It's a pcap challenge. If you're doing stego or re, you're either down a rabbit hole or there's an easier way._

### Find the flag

1. Extract USB data from the first PCAP file
   - Filter USB traffic from a specific source device and extract the captured data
   - Convert the hex data to binary format

```bash
sudo tshark -r pcap2_1583863710056.pcapng -Y 'usb.src == "1.7.1"' -T fields -e usb.capdata > raw
xxd -r -p raw raw2.bin
```

2. Use binwalk to extract embedded files and identify file signatures

```bash
binwalk -e raw2.bin
```

3. Extract TCP data from the main PCAP file
   - Filter TCP traffic destined for port 4444
   - Extract the data payload and convert to binary

```bash
tshark -r pcap_chal.pcap -Y 'tcp.dstport == 4444' -T fields -e data.data > raw3
xxd -r -p raw3 raw4.bin
```

4. Set up cryptcat listener
   - Use cryptcat with encryption key to listen for incoming connections
   - Save received data to a file for analysis

```bash
cryptcat -vv -k BER5348833 -l -p 1337 > raw5.bin
```

5. Send data to the cryptcat listener
   - Use netcat to send the extracted TCP data to the cryptcat listener
   - This simulates the original encrypted communication

```bash
nc -vv -w 1 localhost 1337 < raw4.bin
```

6. Reverse engineer the binary in Ghidra

<img width="1399" height="634" alt="SCREEN01" src="https://github.com/user-attachments/assets/4dd62ee7-2a89-4700-8bf8-298c3aa6d92c" />

7. Decode the flag using [CyberChef](https://gchq.github.io/CyberChef/). Convert the hex value to bytes, reverse the byte order, convert hex bytes to ASCII, apply ROT13 cipher, reverse the string again to get the final flag

<img width="1086" height="523" alt="SCREEN02" src="https://github.com/user-attachments/assets/34e7c790-6a19-4999-aeab-d825e7ac5392" />

# CCT2019 - re3

## There's some kind of a high security lock blocking the way. Defeat the GUI to claim your key!

### What is the key to re3? (Hey, that rhymes)

1. Solve the mathematical constraint
   - The challenge requires finding 4 numbers that satisfy multiple conditions:
     - Sum equals 711
     - Product equals 711,000,000
     - Numbers are in descending order
   - Use Python to brute force all permutations from the given list

```python
import itertools

num = 711
num2 = 711000000
l = [1, 2, 3, 4, 5, 6, 8, 9, 10, 12, 15, 16, 18, 20, 24, 25, 30, 32, 36, 40, 45, 48, 50, 60, 64, 72, 75, 79, 80, 90, 96, 100, 120, 125, 144, 150, 158, 160, 180, 192, 200, 225, 237, 240, 250, 288, 300, 316, 320, 360, 375, 395, 400, 450, 474, 480, 500, 576, 600, 625, 632, 711]

for combo in itertools.permutations(l, 4):
    if (sum(combo) == num and
        combo[0] > combo[1] > combo[2] > combo[3] and
        combo[0] * combo[1] * combo[2] * combo[3] == num2):
        print(combo)
```

<img width="721" height="34" alt="SCREEN03" src="https://github.com/user-attachments/assets/591faa8a-6642-4450-94f7-73fc80386b0b" />

2. Move the sliders in the application to match the calculated values. The correct combination unlocks the key/flag

<img width="514" height="366" alt="SCREEN04" src="https://github.com/user-attachments/assets/556e3717-0105-404e-be33-2b0e599dcbcd" />

# CCT2019 - for1

## Our former employee Ed is suspected of suspicious activity. We found this image on his work desktop and we believe it is something worth analyzing. Can you assist us in extracting any information of value?

### What is the flag?

1. Use exiftool to extract metadata from the JPEG file
   - Morse code in description translates to `JUSTAWARMUPRIGHT?`

```bash
exiftool for1_8f90d68390b565c308871a52c6572de8_1583875226079.jpeg
```

<img width="721" height="415" alt="SCREEN05" src="https://github.com/user-attachments/assets/e459fa14-1304-4a40-a22b-323bef131b46" />

2. Use stegoveritas to perform multiple steganography detection techniques
   - `/results/keepers/7212.zip/fakeflag.txt` can be opened with `justawarmupright?` password
   - In `fakeflag.txt` there is another password: `Z10N0101`
   - In some images in `/results` there is visible passsword: `0ni0n_0f_0bfu5c@ti0n`

```bash
pip3 install stegoveritas
pip3 install --upgrade capstone
stegoveritas for1_8f90d68390b565c308871a52c6572de8_1583875226079.jpeg
```

3. Use steghide to extract hidden data from image file
   - Password to the `archive.zip` is `0ni0n_0f_0bfu5c@ti0n`
   - In `cipher.txt` there is: `FSXL PXTH EKYT DJXS PYMO JLAY VPRP VO`, which should be replaced with `JHSL PGLW YSQO DQVL PFAO TPCY KPUD TF` as said in instruction
   - In `config.txt` there is: `C G. VI. VII. VIII. AMTU RING AM BY CH DR EL FX GO IV JN KU PS QT WZ`

```bash
steghide extract -sf 1757757330.6277616-a8394f6d2aeb2828bda8d743f16a3b58
# Password: Z10N0101
```

4. Use [online Enigma machine simulator](https://cryptii.com/)

<img width="1639" height="710" alt="SCREEN06" src="https://github.com/user-attachments/assets/5efe5120-c441-4775-97e9-993abd69a747" />

5. Remove spaces from the decoded result and use this as password for `/results/keepers/archive.zip/flag.zip/flag.txt`

<img width="730" height="149" alt="SCREEN07" src="https://github.com/user-attachments/assets/6030a381-4166-453d-8f6f-f6716095601b" />

# CCT2019 - crypto1

## Find ye some flags. There are three parts to this challenge, each with its own flag. Solve crypto1a obtain the crypto1a flag and to unlock crypto1b. Solve crypto1b to obtain the crypto1b flag and unlock crypto1c. Solve crypto1c and you'll have all three flags.

### What is the flag for crypto1a?

1. Put text from the `crypto1a.txt` into [dCode cipher identifier](https://www.dcode.fr/cipher-identifier), and then into [dCode keyboard change cipher](https://www.dcode.fr/keyboard-change-cipher)
   - The password is: `dvorakdvorakdvorak`

<img width="791" height="242" alt="SCREEN08" src="https://github.com/user-attachments/assets/d643cf95-2a26-4855-9053-d5735905bd6c" />

<img width="791" height="437" alt="SCREEN09" src="https://github.com/user-attachments/assets/6c29dde8-0150-43cb-97f8-e99d4c6c97fe" />

---

### What is the flag for crypto1b?

1. Put text from the `crypto1b.txt` into [CyberChef rail fence cipher decoder](https://gchq.github.io/CyberChef/)
   - After some googling the password is: `teerrrriiffiicccccc`

<img width="1673" height="581" alt="SCREEN10" src="https://github.com/user-attachments/assets/f42f736b-4daa-4490-a780-20c7ff39ccc5" />

---

### What is the flag for crypto1c?

1. Run the python code to decode message from the `crypto1c.txt`

```python
from Crypto.Util.number import long_to_bytes

enc = "11122112141311112123131222211121621211124112213221112162112113114163113211421121132221622222411321311331611221121413111121231312222111216322121412123312222111122141624112123212416214122231631132114221112162321321242113531132142162321211424112123212322111121322231221111322216222232122141212112163113211422111216231224113221211211232216231224113113232121151124162312311111311416311321142211113212221112162411321222111216212111214132122211121623212114241121232123221111213222312211113222162411211422111124112214113316121421413132163131211211212321241642112141311112162222241132122211121624112231241121121121323221311416311321142141322122111216121212163131214121322213221111321311331615113114162411213242121632122411311322111211241631312211112123212416221321412132221112162411213222141621142211113212221112162112113222164211214131111132131622222123241122323113161421142111111341211212111151322122111122111111512213221111241122131151232121121135211422111132123221115124112123212311513113211422111111513113211211212111221111511"

result = ''.join(('0' if i % 2 == 0 else '1') * int(c) for i, c in enumerate(enc))
flag = long_to_bytes(int(result, 2)).decode("ASCII")
print(flag)
```

<img width="723" height="93" alt="SCREEN11" src="https://github.com/user-attachments/assets/53db8e2d-c78c-42a0-acfe-d30a822902c7" />
