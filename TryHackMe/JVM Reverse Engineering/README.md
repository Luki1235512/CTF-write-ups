# [JVM Reverse Engineering](https://tryhackme.com/room/jvmreverseengineering)

## Learn Reverse Engineering for Java Virtual Machine bytecode

# Introduction

## Consider the following bytecode:

```java
LDC 0
LDC 3
SWAP
POP
INEG
```

### Which value is now at the top of the stack?

1. The stack operations proceed sequentially. After loading constants 0 and 3, the SWAP instruction exchanges their positions. The POP instruction removes the top value (0), leaving 3 at the top. Finally, INEG negates the integer value, resulting in **-3**

```java
LDC 0 // Push constant 0 onto stack: [0]
LDC 3 // Push constant 3 onto stack: [0, 3]
SWAP  // Swap top two values: [3, 0]
POP   // Remove top value: [3]
INEG  // Negate top integer: [-3]
```

---

### Which opcode is used to get the XOR of two longs?

1. The opcode used to get the XOR of two longs is **LXOR**

---

### What does the -v flag on javap stand for?

1. The `-v` flag on `javap` stands for **verbose**

---

# Simple Hello World

## Complete the follow challenges.

### Find the name of the file that this class was compiled from

_Javap is a useful tool to find information about compiled classes_

1. The class was compiled from `SecretSourceFile.java` file

```bash
javap -v -p Main_1584042869908.class
```

<img width="721" height="94" alt="SCREEN01" src="https://github.com/user-attachments/assets/ad579478-9b59-4543-b916-95778dce49c6" />

---

### What is the super class of the Main class?

_Adding flags like -c to javap allow you to see the bytecode of methods. The constructor of a method always calls the constructor of its super class_

1. The class is `java/lang/Object`

```bash
javap -v -c Main_1584042869908.class
```

<img width="719" height="125" alt="SCREEN02" src="https://github.com/user-attachments/assets/03d08287-de03-4cde-88d1-54dd4ab09879" />

---

### What is the value of the local variable in slot 1 when the method returns?

1. The value of the local variable in slot 1 when the method returns is `2`

```bash
javap -v -c Main_1584042869908.class
```

<img width="720" height="383" alt="SCREEN03" src="https://github.com/user-attachments/assets/cd713a7b-906d-4825-b571-ebae18f880e0" />

```java
0: iconst_0       // Push constant 0 onto stack
1: istore_1       // Store 0 into local variable slot 1
2: getstatic      // Get System.out
5: ldc            // Load "Hello World" string
7: invokevirtual  // Call println
10: iinc 1, 2     // Increment local variable slot 1 by 2
13: return        // Method returns
```

---

# Cracking a password protected application

## The given class file takes a password as a parameter. You need to find the correct one. Tools like javap will be sufficient.

### What is the correct password

_The -v tag on javap will print the constant pool, containing utf8 encoded strings_

1. The correct password is **yxvF2ho95ANJVCX**

```bash
javap -v PasswordProtectedApplication_1584043806242.class
```

<img width="721" height="109" alt="SCREEN04" src="https://github.com/user-attachments/assets/c963c75c-f09b-4fdb-85de-9c0017e62da7" />

# Basic String Obfuscation

## Like the previous task, this program takes a password as an argument, and outputs whether or not it is correct. This time the string is not directly present in the class file, and you will need to use either a decompiler, bytecode analysis or virtualisation to find it.

### What is the correct password?

_You will need to use either a decompiler, bytecode analysis or virtualisation to find it._

1. The `xor` method applies XOR with key `(index % 3)` to each character. The obfuscated string `aRa2lPT6A6gIqm4RE` must be decoded by applying the same XOR operation to recover the original password. This can be done in [CyberChef](https://gchq.github.io/CyberChef/) by selecting XOR with key: `000102` HEX
   - the password is: **aSc2mRT7C6fKql6RD**

```bash
javap -v -c -p BasicStringObfuscation_1584044464985.class
```

<img width="872" height="527" alt="SCREEN05" src="https://github.com/user-attachments/assets/b8e8908a-b462-4921-9d8d-deb48a7743b0" />

---

### Advanced String Obfuscation

_You will either need to statically reverse engineer the string deobfuscation functions or use some kind of custom made virtualisation tool_

// TODO
