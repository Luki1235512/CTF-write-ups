# [Trypophobia](https://spooky.ctfd.io/challenges#Trypophobia-45)

Classic reverse engineering, right?

## 1. Identify the file

```bash
file binary
```

```
binary: compiled Java class data, version 52.0 (Java 1.8)
```

It is a compiled Java class, so there is no native code to deal with. `javap` can disassemble it fully.

## 2. Look at the strings

```bash
strings -n 6 binary | head -80
```

The useful lines are:

```
binary
(Ljava/lang/String;)Z
Usage: java binary <flag>
Correct!
Incorrect.
```

The class is named `binary`. It takes the flag as a command line argument and has a method that takes a `String` and returns a `boolean`.

## 3. Copy the class under its real name

The JVM looks for `binary.class` when you run `java binary`, so the file needs that name.

```bash
cp binary binary.class
```

## 4. Disassemble

```bash
javap -c -p -constants binary.class
```

`main` calls `check(args[0])` and prints `Correct!` or `Incorrect.`. In `check`, the first test is:

```
aload_0
invokevirtual String.length
bipush 34
if_icmpne -> return false
```

So the flag is exactly 34 characters long. After that there are 34 blocks, one per index. Each loads `charAt(i)`, runs a few stack operations on it, and jumps to `return false` if the result differs from a constant. Block 0 looks like this:

```
aload_0
iconst_0
invokevirtual String.charAt
bipush 31
swap
ixor
dup
pop
bipush 7
iadd
bipush 115
if_icmpne -> return false
```

Tracing the stack, with `c` as the character:

- `[c]`, then push 31 gives `[c, 31]`.
- `swap` gives `[31, c]`, and `ixor` gives `[31 ^ c]`.
- `dup; pop` does nothing.
- Push 7 and `iadd` give `(31 ^ c) + 7`.
- That must equal 115.

So `31 ^ c = 108`, and `c = 108 ^ 31 = 0x73 = 's'`. That matches the `s` in `spooky{`.

Block 1 is a second pattern:

```
bipush 73
swap
isub
bipush 11
ixor
bipush -46
if_icmpne
```

This computes `(73 - c) ^ 11 == -46`. Since `-46 ^ 11 = -39`, we get `73 - c = -39`, so `c = 112 = 'p'`.

The 34 blocks use a handful of patterns, all built from `swap`, `dup`, `pop`, `ixor`, `iadd` and `isub`. Each block depends only on its own character, with no state shared between positions. That means every position can be solved on its own.

## 5. Automate it

Rather than decode all 34 blocks by hand, I wrote a small stack machine simulator. It parses the `javap` output and splits `check` into one segment per character. For each segment it tries every printable ASCII value as the starting character, runs the ops, and keeps the value for which the final compare passes.

Save this as `solve.py`:

```python
import sys, re

t = sys.stdin.read().split("private static boolean check")[1].split("public static void main")[0]

ins = []
for l in t.splitlines():
    m = re.match(r"\s+\d+: (\w+)\s*(-?\d+)?", l)
    if m:
        ins.append((m.group(1), m.group(2)))

segs = []
cur = None
for op, a in ins:
    if op == "aload_0":
        cur = []
        segs.append(cur)
        continue
    cur.append((op, a))

res = []
for seg in segs[1:]:
    k = [j for j, (op, _) in enumerate(seg) if op == "invokevirtual"][0]
    body = seg[k + 1:]
    found = "?"
    for c in range(32, 127):
        s = [c]
        ok = None
        for op, a in body:
            if op.startswith("iconst_"):
                s.append(int(op[-1]))
            elif op in ("bipush", "sipush"):
                s.append(int(a))
            elif op == "dup":
                s.append(s[-1])
            elif op == "pop":
                s.pop()
            elif op == "swap":
                s[-1], s[-2] = s[-2], s[-1]
            elif op == "ixor":
                b = s.pop(); x = s.pop(); s.append(x ^ b)
            elif op == "iadd":
                b = s.pop(); x = s.pop(); s.append(x + b)
            elif op == "isub":
                b = s.pop(); x = s.pop(); s.append(x - b)
            elif op == "if_icmpne":
                b = s.pop(); x = s.pop()
                ok = (x == b)
                break
        if ok:
            found = chr(c)
            break
    res.append(found)

print("".join(res))
```

Two details matter here. The `iconst_N` handler has to cover every `iconst_0` to `iconst_5`, because some blocks use `iconst_3` as a constant (index 7, for example) and not just as an array index. Also, segment 0 has to be skipped, since it only contains the length check and has no character on the stack.

## 6. Run the solver

```bash
javap -c -p binary.class 2>/dev/null | python3 -I solve.py | tee sol.txt
```

That is 34 characters, which matches the length check. `-I` runs Python in isolated mode so it won't import anything from the current directory.

## 7. Verify against the real binary

```bash
java -cp . binary "$(cat sol.txt)"
```

```
Correct!
```
