# [Too Much Love is Scary](https://spooky.ctfd.io/challenges#Too%20Much%20Love%20is%20Scary-40)

Last time I loved I was left broken and in shambles. That’s when I realized that my heart could only hold so much, I must protect it. So I won’t fall in love again, it’s just too scary…

## 1. Identify the binary

```bash
file philophobia
checksec --file=philophobia
```

It is a dynamically linked x86-64 ELF with PIE, full RELRO, a stack canary and NX. The symbols are intact, so the function names are readable.

## 2. Look at the strings and imports

```bash
strings -n 4 philophobia | head -100
```

Useful findings:

- The imports include `scanf`, `fgets`, `strncmp` and `putchar`.
- The symbol table has `main`, `setup` and `print_flag`.
- The prompts ask for "start end" as two integers, then ask for a second line "based on our difference". The text mentions "a certain famous summer day".

`print_flag` is never reached from a plain run, so the goal is to get `main` to call it.

## 3. Disassemble main

```bash
objdump -d -M intel philophobia --no-show-raw-insn | sed -n '/<main>:/,/^$/p'
```

After the banner `puts` calls, the logic is:

1. `scanf("%d %d", &a, &b)` stores `start` at `[rbp-0x418]` and `end` at `[rbp-0x414]`. Anything other than 2 parsed values exits.
2. `getchar()` eats the leftover newline.
3. `fgets(buf, 0x3e8, stdin)` reads a line into `buf` at `[rbp-0x3f0]`.
4. A 20-byte string is built on the stack from these immediates:
   - `0x2049206c6c616853` is `"Shall I "` (little endian)
   - `0x20657261706d6f63` is `"compare "`
   - `0x65656874` is `"thee"`
5. `strncmp(buf + start, "Shall I compare thee", 0x14)` runs. If it returns 0, `main` calls `print_flag(start, end)`. Otherwise it prints the "NO" message.

The check passes if the phrase from Sonnet 18 sits at offset `start` in the second line. `end` is not checked here. It only feeds the key in the next step.

## 4. Disassemble print_flag

```bash
objdump -d -M intel philophobia --no-show-raw-insn | sed -n '/<print_flag>:/,/^$/p'
```

The function builds 35 bytes of ciphertext on the stack from these constants (the dword fills the tail, and the byte after it is the NUL terminator):

```
0x3721233135352a29
0x34056b056938236e
0x6b056a2e053e6969
0x6929233705692c6a
0x273c6b
```

The key is computed as `key = (end - start) + 0x47`, from `sub eax,[rbp-0x44]` (edi is start) after `mov eax,[rbp-0x48]` (esi is end), then `add eax,0x47`. The loop prints each ciphertext byte XORed with `key` until it hits a NUL byte.

## 5. Recover the key

The flag format is `spooky{`, so the first plaintext byte is `s` (0x73). The first ciphertext byte is `0x29`.

```
key = 0x29 ^ 0x73 = 0x5a
end - start = 0x5a - 0x47 = 0x13 = 19
```

That matches the inclusive index of the last character of the 20-character phrase, so the "difference" is `end - start = 19`.

## 6. Decode offline (optional check)

```python
import struct
d=b''.join(struct.pack('<Q',x) for x in (0x3721233135352a29,0x34056b056938236e,0x6b056a2e053e6969,0x6929233705692c6a))+struct.pack('<I',0x273c6b)
d=d.split(b'\0')[0]
k=19+0x47
print(bytes(b^k for b in d).decode())"
```

## 7. Get it from the binary

Use `start = 0`, `end = 19`, and the phrase at the beginning of the second line:

```bash
printf '0 19\nShall I compare thee\n' | ./philophobia
```

The program prints the flag on its own line after the `puts` message at the top of `print_flag`.

Any other pair works as long as `end = start + 19` and the phrase appears at offset `start`. For example, `6 25` followed by `abcdefShall I compare thee` gives the same key.

<img width="1202" height="427" alt="SCREEN01" src="https://github.com/user-attachments/assets/7f240282-d6df-4e1d-b309-c78b75f6e29b" />
