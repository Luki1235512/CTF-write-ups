# [Patch Tuesday](https://nnsc.tf/challenges?challenge=rev_Patch+Tuesday)

**Description:**

New to reverse engineering? This beginner challenge is an introduction to dynamic debugging and patching Windows programs.

The provided `x86-64` `.exe` promises a free flag, but it does not behave the way we want. Open it in a debugger such as x64dbg or IDA Free, run it, and step through the code that decides whether you receive the flag. Watch how the program compares values and follows a conditional jump.

Try changing that jump while debugging, or patch the instruction in the executable and run your modified file again. Your goal is to make the program follow the path that produces the expected behavior.

## 1. Load the binary

Open `free-flag.exe` in x64dbg. It will break at the entry point, `0x140001000`, before any code runs.

<img width="1916" height="1041" alt="SCREEN01" src="https://github.com/user-attachments/assets/cfd43f60-bd1a-42ee-a2e1-2b863499a25b" />

## 2. Get oriented with the strings

In x64dbg, right click in the disassembly pane and choose Search For > All Modules > String References. You'll see three relevant strings:

- `Press ENTER to get a free NNS{ flag: `
- `Correct! Here is your free flag: `
- `Sorry, no free flag for you.`

<img width="1923" height="1039" alt="SCREEN02" src="https://github.com/user-attachments/assets/b839584c-0268-4527-8f71-e8662bef2d96" />

Double click the "Sorry" string reference to jump to the code that prints it. This is the fastest way to locate the failure branch without reading the whole function line by line.

## 3. Trace backwards to find the branch

From the "Sorry" print code, scroll up a bit. You'll land on this sequence, a few instructions above:

```
test eax,eax
je   <address A>     ; taken when eax == 0
jmp  <address B>     ; taken when eax != 0
```

`<address A>` leads to the "Sorry" print. `<address B>` leads to a decode loop and then the "Correct!" print. So `eax` is the pass/fail flag, and the `je` is the only thing standing between you and the flag.

<img width="2017" height="1086" alt="SCREEN03" src="https://github.com/user-attachments/assets/06898c7c-9c02-428d-ba6a-13a879fa20db" />

Line above the `test eax,eax`, you'll see `call 0x140001160`, which is what sets `eax` in the first place. If you follow that address down, you'll land on a small subroutine that does `cmp ecx, 0x1337` and `sete al`. `ecx` holds whatever integer value was read from your input. Since the program only asks you to press ENTER, the parsed value stays 0, `0 != 0x1337`, `eax` ends up 0, and you always take the failure path.

<img width="1928" height="1037" alt="SCREEN04" src="https://github.com/user-attachments/assets/e1fe3c65-6aaf-4af2-b300-92fc52957aa7" />

You don't need to fully reverse that subroutine to solve the challenge. All that matters is: `test eax,eax` followed by `je` is the single decision point.

## 4. Patch the jump

Right click the `je` instruction and choose Asseble. Replace it with:

```
jne
```

This inverts the condition. Now the program takes the success path whenever `eax != 0x1337`'s result is false, which is exactly the case you're already in (since your input never equals 0x1337). It's a one-character edit to the mnemonic, and x64dbg re-encodes the opcode for you.

An equally valid patch is to overwrite the `je` with two NOPs instead, which removes the conditional entirely and always falls through to the success path. Either works, since the underlying comparison never needs to succeed for you to reach the flag: the target of the whole exercise is just to stop the failure branch from being taken.

## 5. Save the patch to disk

In x64dbg, go to File > Patch File. Review the single patched byte pair and save it as a new exe, or overwrite the original.

This step matters because patching only in the debugger's memory view only affects the current run. Saving the patch to the actual file is what lets you close x64dbg and run the modified exe directly.

## 6. Run the patched executable

Run `.\free-flag-patched.exe` from a terminal, then press ENTER when prompted. The program takes the success branch, decodes its embedded flag buffer with a simple XOR, and prints:

<img width="1113" height="626" alt="SCREEN05" src="https://github.com/user-attachments/assets/3751c8c9-96fb-48c7-8597-279c79556336" />
