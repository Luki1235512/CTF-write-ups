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

# Advanced String Obfuscation

## This program follows the same logic as the previous task, however it has a custom obfuscation layered on top. You might require a decompiler for this, as well as custom tools. This uses anti virtualisation techniques as well, so be warned.

### Find the correct password

_You will either need to statically reverse engineer the string deobfuscation functions or use some kind of custom made virtualisation tool_

This challenge ships three classes obfuscated with Binscure: `0.class`, `1.class`, and `c.class`.

```bash
javap -v -p -c 0.class
javap -v -p -c 1.class
javap -v -p -c c.class
```

`0.main(String[])` checks the argument count and index using constant-folded XOR pairs that resolve to trivial values:

```
1506594314 ^ 1506594315 = 1   -> args.length must be >= 1
-753045066 ^ -753045066 = 0   -> password candidate is args[0]
```

It then computes `E = 0.c("1".a(1, 95))` and compares `E.equals(args[0])`.

`"1".a(int index, int key)` is a control-flow-flattened string table decoder. The `lookupswitch` on a synthetic state variable dispatches between 9 basic blocks; any other state falls through to a trap that does `checkcast java/lang/YourMum`, an intentionally invalid cast meant to crash decompilers or fuzzers that land on a bad state.

Each character of the decoded string is XORed with a key chosen by `i % 5`:

| `i % 5` | key                                                                    |
| ------- | ---------------------------------------------------------------------- |
| 0       | constant `2`                                                           |
| 1       | the method's second argument                                           |
| 2       | `Thread.currentThread().getStackTrace()[2].getClassName().hashCode()`  |
| 3       | `Thread.currentThread().getStackTrace()[2].getMethodName().hashCode()` |
| 4       | sum of the two hash codes above                                        |

The stack trace lookup is the anti-virtualization layer. The key for 3 out of every 5 characters depends on whoever called the caller of `a()`. Call this method from a different context and those characters silently decode to garbage; there's no exception to catch, so a naive tool would produce a plausible-looking but wrong string.

`0.c(String)` layers a second, simpler cipher on top. Working through its constants the same way:

```
1072331622 ^ 1072331622 = 0        -> loop counter starts at 0
-1108647316 ^ -1108647314 = 2      -> mask M
out[i] = in[i] ^ (i & M)
```

Since `M = 2`, this only touches characters where bit 1 of the index is set. Every other character passes through unchanged.

Because the decode depends on the real call stack at the real call site, the reliable way to recover the password is to let the JVM compute it there rather than reimplement the stack trace logic by hand. Using Krakatau to disassemble `0.class`, insert a `dup; getstatic System.out; swap; invokevirtual println` sequence right after the existing `invokestatic 0.c` call in `main()`, and reassemble, the expected value can be leaked without disturbing the call site the anti-VM check relies on:

```bash
mkdir patched
krak2 dis --out 0.j 0.class
# edit 0.j: after `invokestatic Method "0" c (Ljava/lang/String;)Ljava/lang/String;`, insert:
#   dup
#   getstatic Field java/lang/System out Ljava/io/PrintStream;
#   swap
#   invokevirtual Method java/io/PrintStream println (Ljava/lang/String;)V
krak2 asm --out patched 0.j
```

Running the patched `0.class` alongside the untouched `1.class` and `c.class` prints the expected password to stdout before the `equals` check runs.

```bash
mkdir run
cp patched/0.class run/0.class
cp 1.class c.class run
cd run
java 0 test
```

[SCREEN01]

---

# Extreme Obf

This final jar has nearly every exploit I know packed into it. I dont know of any decompilers that will work for it. You will have to use custom tools and bytecode analysis to pick apart this one.

Same format as the previous tasks, takes one argument as the password.

### What is the correct password?

_Hint: You will need custom tools for this one! Feel free to ask mastercooker#7021 on discord for help._

This challenge ships as a jar with `META-INF/MANIFEST.MF`, `0.class`, `1.class`, and `c.class`, but the archive itself is booby-trapped before you even get to the bytecode. Reading the central directory directly shows why:

```py
import zipfile
z = zipfile.ZipFile("BasicStringObfuscation-obf_1584048757202.zip")
for info in z.infolist():
    print(info.filename, info.file_size, info.header_offset, hex(info.CRC))
```

**Results:**

```
â㬥Ô퐀®©¯ë©®ª. 0 0 0x0
META-INF/MANIFEST.MF 19 79 0x193f84a
0.class 1 163 0xdeadbeef
0.class 0 219 0xdeadbeef
0.class 4278 274 0xdeadbeef
1.class 1 2560 0xdeadbeef
1.class 0 2616 0xdeadbeef
1.class 3029 2671 0xdeadbeef
c.class 339 4409 0xf040a22a
```

The archive contains duplicate entries for `0.class` and `1.class`, a garbled non-UTF8 filename with zero size, and every real class entry has its CRC field set to the literal value `deadbeef` instead of a valid checksum. A strict extractor either errors out on the CRC mismatch or grabs the wrong duplicate. unzip just warns and keeps going, landing on the last entry for each name, which is what a size comparison against the central directory confirms.

With the right files in hand:

```bash
javap -v -p -c 0.class
javap -v -p -c 1.class
javap -v -p -c c.class
```

`0.main(String[])` is functionally the same check as the previous task, but every step is now wrapped in extra layers meant to break decompilers and static analysis:

- **Opaque predicates from `c`'s static fields.** `c`'s clinit overwrites its own `ConstantValue` fields at class-load time, so a naive tool reading only the `ConstantValue` attribute gets the wrong number. The real runtime values make most of the branches resolve one way every time, which is exactly what makes them opaque: a decompiler that can't prove the branch is constant has to represent both paths as reachable.
- **`checkcast` on `null`.** Several "traps" do `aconst_null` followed by `checkcast` to a bogus class name. Per the JVM spec, casting `null` always succeeds regardless of the target type, so these never throw. They only look like landmines.
- **Dead `goto`/`athrow` pairs.** Nearly every call is immediately followed by `goto X; athrow`. Since the `goto` is unconditional, the `athrow` right after it is unreachable, it exists purely to pollute a naive control-flow graph.
- **A decoy `invokedynamic`.** The bytecode references an `invokedynamic` call bound to a method in a class named `a`, which doesn't exist anywhere in the jar. It sits behind a condition that's never true at runtime, so it's never actually resolved or invoked, it's bait for tools that eagerly resolve every constant pool entry.
- **The `args.length` and index checks are still simple once decoded.** Working through the same constant-folding as before:

```
-1699801759 ^ -1699801760 = 1   -> args.length must be >= 1
-248472236 ^ -248472236   = 0   -> password candidate is args[0]
```

None of the maze above needs to be hand-traced to get the password. The same technique from the previous task generalizes cleanly: find where the real, final value is computed right before the `String.equals` comparison, and leak it there instead of reconstructing every opaque predicate by hand.

```bash
mkdir patched
krak2 dis --out 0.j 0.class
# edit 0.j: after `invokestatic Method "0" c (Ljava/lang/String;)Ljava/lang/String;`, insert:
#   dup
#   getstatic Field java/lang/System out Ljava/io/PrintStream;
#   swap
#   invokevirtual Method java/io/PrintStream println (Ljava/lang/String;)V
krak2 asm --out patched 0.j
```

Running the patched `0.class` alongside the untouched `1.class` and `c.class` walks through all the opaque predicates using the real runtime field values, resolves the anti-VM stack trace check from the real call site, and prints the expected password before the redundant `equals` check runs.

```bash
mkdir run
cp patched/0.class run/0.class
cp 1.class c.class run
cd run
java 0 test
```

[SCREEN02]
