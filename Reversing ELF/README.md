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

[SCREEN01]

# Crackme2

## Find the super-secret password! and use it to obtain the flag

### What is the super secret password ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme2
./crackme2
./crackme2 password
```

[SCREEN02]

2. Use `strings` to look for possible password strings in the binary
   - The password is **super_secret_password**

```bash
strings crackme2
```

[SCREEN03]

### What is the flag ?

1. Get the flag with password

```bash
./crackme2 super_secret_password
```

[SCREEN04]

# Crackme3

## Use basic reverse engineering skills to obtain the flag

### What is the flag?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme3
./crackme3
./crackme3 PASSWORD
```

[SCREEN05]

2. Use `strings`, and decode the password in [CyberChef](https://gchq.github.io/CyberChef/)
   - The password is encoded with Base64

```bash
strings crackme3
```

[SCREEN06]

# Crackme4

## Analyze and find the password for the binary?

### What is the password ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme4
./crackme4
```

[SCREEN07]

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

[SCREEN08]

# Crackme5

## What will be the input of the file to get output `Good game` ?

### What is the input ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme5
./crackme5
```

[SCREEN09]

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

[SCREEN10]

# Crackme6

## Analyze the binary for the easy password

### What is the password ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme6
./crackme6
./crackme6 password
```

[SCREEN11]

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

[SCREEN12]

3. Use [CyberChef](https://gchq.github.io/CyberChef/) to decode the numbers on the right from Decimal

[SCREEN13]

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

[SCREEN14]

3. After converting the value `0x7a69` from hex to decimal, we get `31337`

[SCREEN15]

# Crackme8

## Analyze the binary and obtain the flag

### What is the flag ?

1. Change the permissions to make the binary executable

```bash
chmod +x crackme8
./crackme8
```

[SCREEN16]

2. Let's use Radare2 to analyze the binary

```bash
r2 -d crackme8
aaa
afl
pdf @main
```

[SCREEN17]

3. After converting the value `0xcafef00d` from hex to decimal, we get `-889262067`

[SCREEN18]
