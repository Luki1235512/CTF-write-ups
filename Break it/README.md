# [Break it](https://tryhackme.com/room/breakit)

## Can you break the code?

# Bases

## This challenge requires the basic knowledge of base decoding/encoding

For this part of the challenge we are gonna use [CyberChef](https://gchq.github.io/CyberChef/)

### [Super easy] MVQXG6K7MJQXGZJTGI======

1. This is Base32

![SCREEN01](https://github.com/user-attachments/assets/ebf28b98-0726-4cc9-a013-2c5b83b1bd57)

### [Easy] TVJYWEtZVE1NVlBXRVlMVE1WWlE9PT09

1. This is Base64 into Base32

![SCREEN02](https://github.com/user-attachments/assets/3c4fe92e-737e-423b-b53f-164b9515bbec)

### [Moderate] GM4HOU3VHBAW6OKNJJFW6SS2IZ3VAMTYORFDMUC2G44EQULIJI3WIVRUMNCWI6KGK5XEKZDTN5YU2RT2MR3E45KKI5TXSOJTKZJTC4KRKFDWKZTZOF3TORJTGZTXGNKCOE======

1. This one is Base32 -> base58 -> Hex -> Base64

![SCREEN03](https://github.com/user-attachments/assets/dbd19bf1-5fc8-4dcf-9e29-398ae32afe0f)

### [Hard] HRBUGQDUHFWDIXKUIBWXIJTHIE3DCY3BIE2FKQSZHNDE6MRUIA2TWWDMHRBV2ZKCHQWFCTLPIE2EEJDBIBZCEW3OHUSTOLRCHNFGMVC6IFJXIQ2AHVMVONSBHVOVIM2MHVPV42J4HQVCQ4REIFHVIJ2WHFWDYQSUHROGILJCIFIU23CXHNCEIXK2HRDVSXKOHV2SQJLC

1. This one is Base32 -> Base85 -> Base64 -> Base58 -> Base85 -> Base64 -> Base64

![SCREEN04](https://github.com/user-attachments/assets/6621f36c-e8cb-411d-9db4-df0b32b23b98)

### [Insane]

1. First in [CyberChef](https://gchq.github.io/CyberChef/) go Base32 -> Base85 -> Decimal -> Hex
2. Then in [dCode](https://www.dcode.fr/base-91-encoding) decrypt the output with Base91
3. Back to [CyberChef](https://gchq.github.io/CyberChef/), and go Base58 -> Hex -> Base64 -> Decimal -> Hex

![SCREEN05](https://github.com/user-attachments/assets/b3b2cad7-64d9-4938-9dc6-a2f1812b12e1)
![SCREEN06](https://github.com/user-attachments/assets/1811c69a-8b23-46fc-8ee9-4c06b71077c2)
![SCREEN07](https://github.com/user-attachments/assets/d43ba7b2-64f7-428d-ae43-4ab69f56273d)

# Base and cipher

## How about mixing both base and cipher?

### [Easy] PJXHQ4S7GEZV6ZTDOZQQ====

1. This is Base32 -> ROT13

![SCREEN08](https://github.com/user-attachments/assets/9ad1be63-a855-4223-ae58-e41ac6b7d054)

### [Moderate] NjZMKVhATl1EcEI2Jio4Q0xuVy1EZSo5ZkFLV0M6QVUtPFpGQ0InIkReYg==

1. Base64 -> Base85 (we receive here a key: tango) -> Vigenère cipher

![SCREEN09](https://github.com/user-attachments/assets/2a7684c7-17db-43d3-bb66-94cdf223be46)

### [Hard] -!r/X,]n/Z-Zs\X,X$,rI<@]#-9Oh,-=A]R-p9\\+

1. Base85 -> ROT47 -> Base64 -> Base32 -> ROR13

![SCREEN10](https://github.com/user-attachments/assets/e66647ae-bd5f-4c65-8e25-8dc3f6e0c7f9)

### [Insane]

1. First [CyberChef](https://gchq.github.io/CyberChef/) Base32 -> Base85
2. [dCode](https://www.dcode.fr/base-91-encoding) decrypt the output with Base91
3. Back to [CyberChef](https://gchq.github.io/CyberChef/) Decimal -> Hex -> Base64, we got key: tangodown
4. Remove the key from input and go Base58 -> Vigenère cipher with key: tangodown

![SCREEN11](https://github.com/user-attachments/assets/f3825c63-8c47-43a9-ab44-f1a3d61b36a2)
![SCREEN12](https://github.com/user-attachments/assets/feebe719-a60b-4b8a-a1a2-000b82236f1c)
![SCREEN13](https://github.com/user-attachments/assets/c242240a-a89a-4cf2-ac67-674f1f74fb86)

# base, cipher, bit shift

## Can you decode the combination? Maybe Hex workshop can help you out

### [Moderate] 2D 37 2B 19 31 99 31 B3 B2 AB A5 18 32 37 20 B3 B2 AC 2D 1A 31 B4 A1 3A A4 A3 9C B4 AD 36 AC 9E

1. Do a 127 bit shift in [dCode](https://www.dcode.fr/circular-bit-shift)
2. In [CyberChef](https://gchq.github.io/CyberChef/) do Base64 -> ROT13

![SCREEN14](https://github.com/user-attachments/assets/16c2a68b-33c6-47ec-83e4-173b0ce350c0)
![SCREEN15](https://github.com/user-attachments/assets/b92c16c7-56fb-464c-934c-bb9c9e402cb4)

### [Hard] 19 1A 1C 99 1C 9C A8 38 27 B2 34 27 B9 27 26 28 3C 3B AA 28 A5 AA 2A 29 3C 29 B2 B2 21 B1 AC A6 1C AC 29 A8 38 99 2C

1. Do a 7 bit shift in [dCode](https://www.dcode.fr/circular-bit-shift)
2. In [CyberChef](https://gchq.github.io/CyberChef/) do Base58 -> ROT13

![SCREEN16](https://github.com/user-attachments/assets/8d9cb144-a250-458d-a0f4-2e8b88fd4694)
![SCREEN17](https://github.com/user-attachments/assets/b4ed8843-90ca-49a3-be96-b5c282a89d4e)

### [Insane] A9 A8 2B EE 2A AA C9 C8 C9 AA A8 0F 2A 8A AA EE A9 8A CA A8 A9 4F 6D 46 E9 A8 A8 0F A9 8A 6D 86 A9 AA A9 26 2A 8A 48 E8

1. First do a 5 bit shift in [dCode](https://www.dcode.fr/circular-bit-shift)
2. In [CyberChef](https://gchq.github.io/CyberChef/) do Base64
3. Back to [dCode](https://www.dcode.fr/circular-bit-shift), and do 126 bits shift
4. Again in [CyberChef](https://gchq.github.io/CyberChef/) do Base85 -> ROT13

![SCREEN18](https://github.com/user-attachments/assets/d2ef7f1e-4af3-458f-88d0-f69cc51d6ff6)
![SCREEN19](https://github.com/user-attachments/assets/fc915019-092f-43de-8b4b-302594cb4696)
![SCREEN20](https://github.com/user-attachments/assets/674dd25d-ea8b-4888-bad0-40b10f7bc9fe)
![SCREEN21](https://github.com/user-attachments/assets/22667998-66b2-4200-b981-3755e587ee42)
