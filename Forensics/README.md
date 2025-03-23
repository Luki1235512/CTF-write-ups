# Forensics

## This is a memory dump of compromised system, do some forensics kung-fu to explore the inside

# Volatility forensics

### What is the Operating System of this Dump file? (OS name)

1. Run the `.raw` through volatility3

```Bash
git clone https://github.com/volatilityfoundation/volatility.git
cd volatility
python vol.py -f ../victim.raw imageinfo
```

![SCREEN01](https://github.com/user-attachments/assets/7dd6ee88-0e99-40cf-b5ff-feed1cdb04be)

### What is the PID of SearchIndexer?

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 pstree | grep -i searchindexer
```

![SCREEN02](https://github.com/user-attachments/assets/a22dbed6-27ce-4a8d-a38c-1cd632bc2bc5)

### What is the last directory accessed by the user?

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 shellbags
```

![SCREEN03](https://github.com/user-attachments/assets/d050878b-d9a1-4c6e-bf8e-c08e87ff88be)

# Task 2

## Dig a little more...

### There are many suspicious open ports; which one is it?

1. It will be the first one UDP:5005

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 netscan
```

![SCREEN04](https://github.com/user-attachments/assets/166edcf8-02bd-4cec-a594-b59b80c0f634)

### Vads tag and execute protection are strong indicators of malicious processes; can you find which they are?

1. The `malfind` plugin is particularly useful as it automatically looks for suspicious memory regions with executable pages that aren't backed by a file on disk - a common characteristic of injected code.

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 malfind
```

![SCREEN05](https://github.com/user-attachments/assets/c8b85a64-bdf7-4a6c-b58c-9a447515faa5)

# IOC SAGA

## In the previous task, you identified malicious processes, so let's dig into them and find some Indicator of Compromise (IOC). You just need to find them and fill in the blanks (You may search for them on VirusTotal to discover more details).

### 'www.go\*\*\*\*.ru' (write full url without any quotation marks)

1. First dump the first pid

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 -p 1820 memdump --dump-dir=../dump
```

2. Search the `.dmp`file for `www\.go....\.ru`, it will be the last one

```Bash
strings 1820.dmp | grep -i "www\.go....\.ru"
```

![SCREEN06](https://github.com/user-attachments/assets/c84e1a48-c3d1-4726-9618-8e604a8fca0f)

### 'www.i\*\*\*\*.com' (write full url without any quotation marks)

1. Again search the dump for `www\.i....\.com`, and it will be the first one

```Bash
strings 1820.dmp | grep -i "www\.i....\.com"
```

![SCREEN07](https://github.com/user-attachments/assets/dedd055e-9565-44b1-af1d-cfb69f1bf7a1)

### 'www.ic\*\*\*\*\*\*.com'

```Bash
strings 1820.dmp | grep -i "www\.ic......\.com"
```

![SCREEN08](https://github.com/user-attachments/assets/f0798709-01ae-429d-bda7-0cf89a3a41a4)

### 202.\*\*\*.233.\*\*\* (Write full IP)

```Bash
strings 1820.dmp | grep -i "202\....\.233\...."
```

![SCREEN09](https://github.com/user-attachments/assets/ec892485-1963-4c87-9823-54b8a31c2677)

### \*\*\*.200.\*\*.164 (Write full IP)

```Bash
strings 1820.dmp | grep -i "...\.200\...\.164"
```

![SCREEN10](https://github.com/user-attachments/assets/622b67e1-d3ab-45c9-8804-4f8d71ec61be)

### 209.190.\*\*\*.\*\*\*

```Bash
strings 1820.dmp | grep -i "209\.190\....\...."
```

![SCREEN11](https://github.com/user-attachments/assets/6815aa5e-d620-4731-ad41-15812cb60d60)

### What is the unique environmental variable of PID 2464?

1. To find the unique environmental variable of PID 2464, you need to examine the environment variables of that specific process and identify any that are unusual or unique to it.

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 envars -p 2464
```

![SCREEN12](https://github.com/user-attachments/assets/b9591f1a-8b43-4f07-a6e4-7f38cbc38688)
