# [Cyberphobia: Formatting Nightmare](https://spooky.ctfd.io/challenges#Cyberphobia:%20Formatting%20Nightmare-20)

I trusted the string, it revealed my secrets.

flag can only be fetched from the remote server, wrap answer in spooky{}

## Step 1: Look at the provided files

```bash
ls -la
cat fsvchall2.c
```

The directory has the binary `fsvchall2` and its source `fsvchall2.c`.

## Step 2: Find the vulnerability

`main()` reads `/opt/ctf/chall2/secret.txt` into a 256-byte buffer `secret` and passes it to `vuln()`:

```c
void vuln(const char *secret) {
    char input[256];
    printf("Input: ");

    if (fgets(input, sizeof(input), stdin) == NULL)
        return;

    printf(input, secret);
    printf("\n");
}
```

The bug is `printf(input, secret)`. The user-controlled `input` is used as the format string, so any conversion specifiers we send get interpreted. The `secret` pointer is also passed as an extra argument, so `printf` will happily consume it if the format string asks for an argument.

## Step 3: Work out which specifier prints the secret

On x86-64 (System V ABI), the first argument goes in `rdi` (the format string) and the first variadic argument goes in `rsi`. In `printf(input, secret)`, `rsi` holds the pointer to `secret`.

That makes the first argument-consuming conversion refer to the secret. The `%s` conversion reads a C string from the pointer it is given, so `%s` prints the secret. The positional form `%1$s` selects argument 1 explicitly, which is the same thing and does not depend on any other specifiers in the input.

No stack offset search or memory write is needed. The secret is handed to `printf` directly.

## Step 4: Send the payload to the remote server

The flag only exists on the remote server, so the payload has to go there:

```bash
echo '%1$s' | nc -w5 18.226.103.146 9002
```

`echo` sends `%1$s` plus a newline, which satisfies `fgets`. The single quotes stop the shell from expanding `$s`. The `-w5` flag makes `nc` exit after 5 seconds of inactivity so it does not hang once stdin closes.

## Step 5: Read the output

```
Input: f0rm4tStr1ngVulN3r4b1lityIsSc4rY
```

The server printed the contents of `secret.txt` where `%1$s` was.

[SCREEN01]
