# [Sneaky Patch](https://tryhackme.com/room/hfb1sneakypatch)

## Investigate the potential kernel backdoor implanted within the compromised system.

A high-value system has been compromised. Security analysts have detected suspicious activity within the kernel, but the attacker’s presence remains hidden. Traditional detection tools have failed, and the intruder has established deep persistence. Investigate a live system suspected of running a kernel-level backdoor.

### What is the flag?

1. **Enumerate loaded kernel modules**

The first step is to list all currently loaded kernel modules and sort them alphabetically, making
it easier to spot anything unusual among the standard Linux/AWS kernel drivers.

```bash
lsmod | sort
```

**Results:**

```
8021q                  45056  0
Module                  Size  Used by
aesni_intel           356352  0
autofs4                57344  2
binfmt_misc            24576  1
crc32_pclmul           12288  0
crct10dif_pclmul       12288  1
cryptd                 24576  2 crypto_simd,ghash_clmulni_intel
crypto_simd            16384  1 aesni_intel
dm_multipath           45056  0
efi_pstore             12288  0
ena                   151552  0
garp                   20480  1 8021q
ghash_clmulni_intel    16384  0
input_leds             12288  0
ip_tables              32768  0
llc                    16384  2 stp,garp
lp                     32768  0
mrp                    20480  1 8021q
msr                    12288  0
nfnetlink              20480  2
parport                73728  3 parport_pc,lp,ppdev
parport_pc             53248  0
polyval_clmulni        12288  0
polyval_generic        12288  1 polyval_clmulni
ppdev                  24576  0
psmouse               217088  0
sch_fq_codel           24576  3
serio_raw              20480  0
sha1_ssse3             32768  0
sha256_ssse3           32768  0
spatch                 12288  0
stp                    12288  1 garp
x_tables               65536  1 ip_tables
```

Scanning the sorted output, all modules appear to be standard Linux kernel drivers associated with networking, cryptography, hardware, and AWS infrastructure with one glaring exception: `spatch`. This module is completely unrecognised, has a very small footprint, is not used by any other module, and does not correspond to any known upstream Linux driver. This makes it an candidate for further investigation.

2. **Inspect the suspicious module's metadata**

Use `modinfo` to query the kernel's internal metadata for the `spatch` module. This reveals its on-disk location, author, description, license, and build fingerprint without actually unloading or interacting with it.

```bash
sudo modinfo spatch
```

**Results:**

```
filename:       /lib/modules/6.8.0-1016-aws/kernel/drivers/misc/spatch.ko
description:    Cipher is always root
author:         Cipher
license:        GPL
srcversion:     81BE8A2753A1D8A9F28E91E
depends:
retpoline:      Y
name:           spatch
vermagic:       6.8.0-1016-aws SMP mod_unload modversions
```

Several details here are highly suspicious:

- **Author:** `Cipher`.
- **Description:** A provocative message that strongly implies the module is designed to grant or maintain root-level access on behalf of `Cipher`.
- **Location:** The module has been dropped into the `drivers/misc/` directory, a common location for miscellaneous kernel drivers and a subtle place to hide a rogue `.ko` file.

3. **Extract readable strings from the module binary**

Running `strings` against the raw `.ko` file extracts all human-readable ASCII sequences embedded in the binary. This is a quick, non-intrusive way to understand what the module does without reverse-engineering the ELF bytecode directly.

```bash
strings /lib/modules/6.8.0-1016-aws/kernel/drivers/misc/spatch.ko | head -n 25
```

**Results:**

```
Linux
Linux
AUATL
[A\A]]1
AUATL
get_flagH9
[A\A]]1
cipher_bd
/tmp/cipher_output.txt
/bin/sh
%s > %s 2>&1
get_flag
/root/src/spatch.c
HOME=/root
3[CIPHER BACKDOOR] Failed to create /proc entry
6[CIPHER BACKDOOR] Module loaded. Write data to /proc/%s
6[CIPHER BACKDOOR] Module unloaded.
3[CIPHER BACKDOOR] Failed to read output file
6[CIPHER BACKDOOR] Command Output: %s
3[CIPHER BACKDOOR] No output captured.
6[CIPHER BACKDOOR] Executing command: %s
3[CIPHER BACKDOOR] Failed to setup usermode helper.
6[CIPHER BACKDOOR] Format: echo "COMMAND" > /proc/cipher_bd
6[CIPHER BACKDOOR] Try: echo "%s" > /proc/cipher_bd
6[CIPHER BACKDOOR] Here's the secret: 54484d7b73757033725f736e33346b795f643030727d0a
```

This output makes the backdoor's mechanism clear but most critically, the final string contains a hex-encoded secret: `54484d7b73757033725f736e33346b795f643030727d0a`

4. **Decode the hex-encoded flag**

The hex string leaked in the binary strings output is the flag, encoded in plain hexadecimal. Decode it using [CyberChef](https://gchq.github.io/CyberChef/) with the "**From Hex**" operation:

<img width="1010" height="501" alt="SCREEN01" src="https://github.com/user-attachments/assets/f79937ef-cee8-46a5-80eb-5b050d2e52a1" />
