# [Elves can be Spooky Too](https://spooky.ctfd.io/challenges#Elves%20can%20be%20Spooky%20Too-28)

the elves left you a very small doorway. you are given a host, a port, and a byte budget. send an x86-64 linux elf that: is at most 112 bytes (the service prints the live limit on connect), writes its own exact bytes to stdout, exits with status 0. do that and the service hands you the flag. host: port: `31903` protocol: connect, read the banner, send `SIZE ` then a newline, then exactly raw bytes. reply is PASS/FLAG or FAIL.

## Step 1: Work out which header fields the kernel actually checks

For a static `ET_EXEC` binary, the Linux loader checks only these:

- `e_ident[0..3]` is `\x7fELF`, and `EI_CLASS` is 2 (64-bit).
- `e_type` is 2 (`ET_EXEC`).
- `e_machine` is 0x3e (x86-64).
- `e_phentsize` is 56.
- `e_phoff` and `e_phnum` point at a valid program header table.
- `e_entry` is the entry point.

It ignores `e_ident[8..15]`, `e_version`, `e_shoff`, `e_flags`, `e_ehsize`, `e_shentsize`, `e_shnum` and `e_shstrndx`. That leaves free space for code at:

- offsets 8..15 (8 bytes)
- offsets 20..23 (4 bytes)
- offsets 40..53 (14 bytes)

## Step 2: Overlap the program header with the ELF header

I set `e_phoff = 56`, so the program header starts at file offset 56, inside the last 8 bytes of the ELF header.

`p_type` occupies offsets 56..59, which are the same bytes as `e_phnum` (56..57) and `e_shentsize` (58..59). Setting `e_phnum = 1` and `e_shentsize = 0` makes those four bytes `01 00 00 00`, which is `PT_LOAD`. One set of bytes serves both structures.

`p_flags` sits at offsets 60..63, which overlap `e_shnum` and `e_shstrndx`. Both are ignored, so I wrote 5 (read plus execute) there.

The rest of the program header runs from 64 to 111, which makes the whole file exactly 112 bytes. The kernel reads the full 56-byte program header, so the file cannot be cut shorter than 112 with this layout.

Layout:

| Offset | Size | Field                                    | Value                     |
| ------ | ---- | ---------------------------------------- | ------------------------- |
| 0      | 8    | magic, class, data, version, osabi       | `7f 45 4c 46 02 01 01 00` |
| 8      | 8    | ignored, code part 1                     | `ff c0 ff c7 b2 70 eb 18` |
| 16     | 2    | e_type                                   | 2                         |
| 18     | 2    | e_machine                                | 0x3e                      |
| 24     | 8    | e_entry                                  | 0x400008                  |
| 32     | 8    | e_phoff                                  | 56                        |
| 40     | 13   | ignored, code part 2                     | see below                 |
| 54     | 2    | e_phentsize                              | 56                        |
| 56     | 4    | e_phnum=1, e_shentsize=0 (also p_type=1) | `01 00 00 00`             |
| 60     | 4    | p_flags                                  | 5                         |
| 64     | 8    | p_offset                                 | 0                         |
| 72     | 8    | p_vaddr                                  | 0x400000                  |
| 88     | 8    | p_filesz                                 | 112                       |
| 96     | 8    | p_memsz                                  | 112                       |

`p_offset = 0` and `p_vaddr = 0x400000` map the entire file at 0x400000. That puts the file's own bytes in memory at a known address, which is what the quine needs. The entry point is 0x400008, the start of the code in the ignored ident padding.

## Step 3: Write the code

At process start on x86-64 Linux, the general-purpose registers other than `rsp` are zero. This lets the code skip loading zeros.

Part 1, at offsets 8..15:

```
ff c0        inc eax          ; eax = 1 (SYS_write)
ff c7        inc edi          ; edi = 1 (stdout)
b2 70        mov dl, 112      ; edx = 112 (count)
eb 18        jmp +0x18        ; next ip is 16, target is 40
```

Part 2, at offsets 40..52:

```
be 00 00 40 00   mov esi, 0x400000   ; buffer = start of our own mapping
0f 05            syscall             ; write(1, 0x400000, 112)
ff cf            dec edi             ; edi = 0
b0 3c            mov al, 60          ; eax = 60 (SYS_exit), upper bytes already 0
0f 05            syscall             ; exit(0)
```

After the `write` syscall returns, `eax` holds 112. Loading 60 into `al` leaves the upper bytes at zero, so `eax` is exactly 60.

## Step 4: Generate the file

The generator script, saved as `mk.py`:

```python
import struct,sys
b=bytearray(112)
b[0:8]=b'\x7fELF\x02\x01\x01\x00'
b[8:16]=bytes([0xff,0xc0, 0xff,0xc7, 0xb2,0x70, 0xeb,0x18])   # inc eax; inc edi; mov dl,112; jmp 40
struct.pack_into('<HH',b,16,2,0x3e)
struct.pack_into('<Q',b,24,0x400008)   # entry
struct.pack_into('<Q',b,32,56)         # phoff
b[40:53]=bytes([0xbe,0,0,0x40,0, 0x0f,0x05, 0xff,0xcf, 0xb0,0x3c, 0x0f,0x05])
struct.pack_into('<H',b,54,56)         # phentsize
struct.pack_into('<H',b,56,1)          # phnum ; also p_type=1 (shentsize=0)
struct.pack_into('<I',b,60,5)          # p_flags
struct.pack_into('<Q',b,64,0)          # offset
struct.pack_into('<Q',b,72,0x400000)   # vaddr
struct.pack_into('<Q',b,88,112)        # filesz
struct.pack_into('<Q',b,96,112)        # memsz
open('q.elf','wb').write(b)
```

Commands used to build and test it locally:

```bash
python3 -I mk.py
chmod +x q.elf
./q.elf > out.bin; echo "exit=$?"
cmp q.elf out.bin && echo QUINE_OK
wc -c q.elf
```

Output:

```
exit=0
QUINE_OK
112 q.elf
```

I dumped the bytes with Python:

```bash
python3 -I -c "d=open('q.elf','rb').read();print(len(d));print(d.hex())"
```

This printed the 112 bytes as hex:

```
7f454c4602010100ffc0ffc7b270eb1802003e000000000008004000000000003800000000000000be000040000f05ffcfb03c0f050038000100000005000000000000000000000004000000000000000000000000000700000000000000070000000000000000000000000000000
```

## Step 5: Submit to the service

The challenge protocol is: connect, read the banner, send `SIZE <n>` and a newline, then exactly n raw bytes.

```python
import socket
d=bytes.fromhex("7f454c4602010100ffc0ffc7b270eb1802003e000000000008004000000000003800000000000000be000040000f05ffcfb03c0f050038000100000005000000000000000000000000004000000000000000000000000000700000000000000070000000000000000000000000000000")
s=socket.create_connection(("3.129.162.36",31903))
print(s.recv(4096).decode())
s.sendall(b"SIZE %d\n"%len(d)+d)
print(s.recv(4096).decode())
```

The banner confirmed the limit of 112, and the server replied:

<img width="563" height="234" alt="SCREEN01" src="https://github.com/user-attachments/assets/27aa6a55-107f-4314-898f-b616f485a650" />
