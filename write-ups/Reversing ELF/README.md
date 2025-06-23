# [Reversing ELF](https://tryhackme.com/room/reverselfiles)

## Room for beginner Reverse Engineering CTF players

# Crackme1

## Let's start with a basic warmup, can you run the binary?

### What is the flag?

1. First, change the permissions to make the binary executable

```bash
chmod +x crackme1
```

2. Run the binary directly

```bash
./crackme1
```

![SCREEN01](https://github.com/user-attachments/assets/b7899baa-7c14-4192-beeb-8593920d4fc0)

# Crackme2

## Find the super-secret password! and use it to obtain the flag

### What is the super secret password ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme2
./crackme2
./crackme2 password
```

![SCREEN02](https://github.com/user-attachments/assets/45b23cea-0983-493c-ae88-612f71fc7643)

2. Use `strings` to look for possible password strings in the binary
   - The password is **super_secret_password**

```bash
strings crackme2
```

![SCREEN03](https://github.com/user-attachments/assets/da3be2a6-5219-4172-a01f-1c95c976a634)

### What is the flag ?

1. Get the flag with password

```bash
./crackme2 super_secret_password
```

![SCREEN04](https://github.com/user-attachments/assets/6de4c579-574e-4428-8351-40b6e1fa011e)

# Crackme3

## Use basic reverse engineering skills to obtain the flag

### What is the flag?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme3
./crackme3
./crackme3 PASSWORD
```

![SCREEN05](https://github.com/user-attachments/assets/b319ce9c-6d48-41ff-bbd2-eb4a12558027)

2. Use `strings`, and decode the password in [CyberChef](https://gchq.github.io/CyberChef/)
   - The password is encoded with Base64

```bash
strings crackme3
```

![SCREEN06](https://github.com/user-attachments/assets/73a44d9e-b5fb-4a26-9b2e-3c30a00affa7)

# Crackme4

## Analyze and find the password for the binary?

### What is the password ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme4
./crackme4
```

![SCREEN07](https://github.com/user-attachments/assets/33d49172-38f8-4c6e-bcde-b182da2a5670)

2. Let's run the binary through GDB to analyze it while it's executing

```bash
gdb crackme4
info functions

# Set break on our most likely strcmp function
break strcmp@plt
run test

info registers

# Try registers one by one
x/s 0x7fffffffdee0
```

![SCREEN08](https://github.com/user-attachments/assets/214db6f3-1928-4f16-99ac-4227d8a47363)

# Crackme5

## What will be the input of the file to get output `Good game` ?

### What is the input ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme5
./crackme5
```

![SCREEN09](https://github.com/user-attachments/assets/5fcf8301-abde-433d-b8ad-20e1001e5896)

2. Run the binary through GDB to analyze it while it's executing

```bash
gdb crackme5
info functions

# Set break on most suspicious function
break strlen@plt
run test

info registers

# Try registers one by one
x/s 0x7fffffffdef0
```

![SCREEN10](https://github.com/user-attachments/assets/0b6c49ee-a8f6-4128-acd0-ae561f1b3d18)

# Crackme6

## Analyze the binary for the easy password

### What is the password ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme6
./crackme6
./crackme6 password
```

![SCREEN11](https://github.com/user-attachments/assets/5b6c4dc2-277d-468d-b124-83500d0fe34b)

2. This binary seems more complex. Let's use Radare2 for analysis

```bash
r2 -d crackme6
aaa
# Analyze all functions
afl
# List all functions
pdf @main
pdf @sym.compare_pwd
pdf @sym.my_secure_test
```

![SCREEN12](https://github.com/user-attachments/assets/ab0716ae-33ad-4e20-8121-7f58dd5cf28e)

3. Use [CyberChef](https://gchq.github.io/CyberChef/) to decode the numbers on the right from Decimal

![SCREEN13](https://github.com/user-attachments/assets/6175b9f3-7a46-49b7-8e1f-c3110cc47b4d)

# Crackme7

## Analyze the binary to get the flag

### What is the flag ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme7
./crackme7
```

2. Let's use Radare2 to analyze the binary

```bash
r2 -d crackme7
aaa
# Analyze all functions
afl
# List all functions
pdf @main
```

![SCREEN14](https://github.com/user-attachments/assets/a6f8f730-edb1-4edc-9fce-7eda62d726ab)

3. After converting the value `0x7a69` from hex to decimal, we get `31337`

![SCREEN15](https://github.com/user-attachments/assets/6af9f423-d6f0-4da9-b115-0cf99cb27c00)

# Crackme8

## Analyze the binary and obtain the flag

### What is the flag ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme8
./crackme8
```

![SCREEN16](https://github.com/user-attachments/assets/e79711ca-cd26-4630-904e-a139112eca28)

2. Let's use Radare2 to analyze the binary

```bash
r2 -d crackme8
aaa
afl
pdf @main
```

![SCREEN17](https://github.com/user-attachments/assets/e0a283b7-492a-48e1-b43a-b2f7d1213e2a)

3. After converting the value `0xcafef00d` from hex to decimal, we get `-889262067`

![SCREEN18](https://github.com/user-attachments/assets/fb0f251a-6e06-4640-8fd9-ffec03d61163)
