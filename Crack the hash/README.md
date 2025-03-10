# Level 1
## Can you complete the level 1 tasks by cracking the hashes?

### Hash 1.1: 48bb6e862e54f2a795ffc4e541caed4d

We'll use [hashcat](https://hashcat.net/hashcat/) for cracking. The hint indicates this is an MD5 hash

1. First, save the hash to a file:
```bash
echo "48bb6e862e54f2a795ffc4e541caed4d" > hash.txt
```

2. Crack the hash using hashcat:
```bash
hashcat -m 0 -a 0 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt
```
Parameters explained:
- `-m 0`: Hash mode, since we are cracking MD5 we use 0 according to [hashcat hash types table](https://hashcat.net/wiki/doku.php?id=example_hashes)
- `-a 0`: We use dictionary attack as it should be enough for common and weak passwords

[SCREEN1]

### Hash 1.2: CBFDAC6008F9CAB4083784CBD1874F76618D2A97

Hint: Sha.. but which version

Based on the length (40 characters) and verification using [Crack Station](https://crackstation.net/), this is a SHA-1 hash.

[SCREEN2]

1. Save the hash to a file:
```bash
echo "CBFDAC6008F9CAB4083784CBD1874F76618D2A97" > hash.txt
```

2. Crack the hash using hashcat:
```bash
hashcat -m 100 -a 0 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt
```
Parameters:
- `-m 100`: Hash mode for SHA-1
- `-a 0`: Dictionary attack mode

[SCREEN3]

### Hash 1.3: 1C8BFE8F801D79745C4631D09FFF36C82AA37FC4CCE4FC946683D7B336B63032

Hint indicates SHA again, after checking in [Crack station](https://crackstation.net/) we know it is SHA-256

[SCREEN4]

1. Save the hash to a file:
```bash
echo "1C8BFE8F801D79745C4631D09FFF36C82AA37FC4CCE4FC946683D7B336B63032" > hash.txt
```

2. Crack the hash using hashcat:
```bash
hashcat -m 1400 -a 0 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt
```
Parameters:
- `-m 1400`: Hash mode for SHA-256
- `-a 0`: Dictionary attack mode

### Hash 1.4: $2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom

The `$2y$` prefix indicates bcrypt

1. Save the hash to a file (we need to add escape characters):
```bash
echo "\$2y\$12\$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom" > hash.txt
```
2. Since bcrypt is very slow to crack, we should optimize our approach - we will filter only four-letter words from `rockyou.txt`
```bash
grep -E '^[a-z]{4}$' /root/Desktop/Tools/wordlists/rockyou.txt > shortwords.txt
```
3. Crack the hash using hashcat:
```bash
hashcat -m 3200 -a 0 hash.txt shortwords.txt
```
- `-m 3200`: Hash mode for bcrypt
- `-a 0`: Dictionary attack mode

[SCREEN6]

### Hash 1.5: 279412f945939ba78ce0758d3fd83daa

Hint says it's MD4

1. Save the hash to a file:
```bash
echo "279412f945939ba78ce0758d3fd83daa" > hash.txt 
```
2. Unfortunately this password is not in the `rockyou.txt` file, but we can crack it with [Crack station](https://crackstation.net/). This will tell us that the password has 2 numbers at the end if we still want to use hashcat:
```bash
hashcat -m 900 -a 6 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt ?d?d
```
Parameters:
- `-m 900`: Hash mode for MD4
- `-a 6`: Smart hybrid attack
- `?d?d`: + 2 digits at the end of the password

[SCREEN7]

# Level 2

## This task increases the difficulty. All of the answers will be in the classic rock you password list. You might have to start using hashcat here and not online tools. It might also be handy to look at some example hashes on hashcats page.


### Hash 2.1: F09EDCB1FCEFC6DFB23DC3505A882655FF77375ED8AA2D1C13F640FCCC2D0C85

Using [Crack station](https://crackstation.net/) we can check that this is SHA-256 hash

1. Save the hash to a file:
```bash
echo "F09EDCB1FCEFC6DFB23DC3505A882655FF77375ED8AA2D1C13F640FCCC2D0C85" > hash.txt
```

2. Crack the hash using hashcat:
```bash
hashcat -m 1400 -a 0 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt
```
- `-m 1400`: Hash mode for SHA-256
- `-a 0`: Dictionary attack mode

[SCREEN8]

### Hash 2.2: 1DFECA0C002AE40B8619ECF94819CC1B

Hint says it is NTLM hash

1. Save the hash to a file:
```bash
echo "1DFECA0C002AE40B8619ECF94819CC1B" > hash.txt
```

2. Crack the hash using hashcat:
```bash
hashcat -m 1000 -a 0 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt
```
- `-m 1000`: Hash mode for NTLM
- `-a 0`: Dictionary attack mode

[SCREEN9]

### Hash 2.3: $6$aReallyHardSalt$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqMhSrHVXgMpdjS6xeKZAs02.

This is SHA-512 with "aReallyHardSalt" salt, we can find it in [hashcat examples](https://hashcat.net/wiki/doku.php?id=example_hashes)

1. Save the hash to a file:
```bash
echo "\$6\$aReallyHardSalt\$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqMhSrHVXgMpdjS6xeKZAs02." > hash.txt
```

2. Crack the hash using hashcat (hashcat will automatically extract the salt):
```bash
hashcat -m 1800 -a 0 -w 4 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt
```
- `-m 1800`: Hash mode for SHA-512
- `-a 0`: Dictionary attack mode
- `-w 4`: Workload Profile, 4 is the highest

[SCREEN10]

### Hash 2.4: e5d8870e5bdd26602cab8dbe07a942c8669e56d6

This is HMAC-SHA1 with "tryhackme" salt

1. Save the hash with salt to a file:
```bash
echo "e5d8870e5bdd26602cab8dbe07a942c8669e56d6:tryhackme" > hash.txt
```
2. Crack the hash using hashcat:
```bash
hashcat -m 160 -a 0 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt
```
- `-m 160`: Hash mode for HMAC-SHA1
- `-a 0`: Dictionary attack mode

[SCREEN11]