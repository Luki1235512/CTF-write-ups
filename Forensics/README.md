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

[SCREEN01]

### What is the PID of SearchIndexer?

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 pstree | grep -i searchindexer
```

[SCREEN02]

### What is the last directory accessed by the user?

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 shellbags
```

[SCREEN03]

# Task 2

## Dig a little more...

### There are many suspicious open ports; which one is it?

1. It will be the first one UDP:5005

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 netscan
```

### Vads tag and execute protection are strong indicators of malicious processes; can you find which they are?

1. The `malfind` plugin is particularly useful as it automatically looks for suspicious memory regions with executable pages that aren't backed by a file on disk - a common characteristic of injected code.

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 malfind
```

[SCREEN05]

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

[SCREEN06]

### 'www.i\*\*\*\*.com' (write full url without any quotation marks)

1. Again search the dump for `www\.i....\.com`, and it will be the first one

```Bash
strings 1820.dmp | grep -i "www\.i....\.com"
```

[SCREEN07]

### 'www.ic\*\*\*\*\*\*.com'

```Bash
strings 1820.dmp | grep -i "www\.ic......\.com"
```

[SCREEN08]

### 202.\*\*\*.233.\*\*\* (Write full IP)

```Bash
strings 1820.dmp | grep -i "202\....\.233\...."
```

[SCREEN09]

### \*\*\*.200.\*\*.164 (Write full IP)

```Bash
strings 1820.dmp | grep -i "...\.200\...\.164"
```

[SCREEN10]

### 209.190.\*\*\*.\*\*\*

```Bash
strings 1820.dmp | grep -i "209\.190\....\...."
```

[SCREEN11]

### What is the unique environmental variable of PID 2464?

1. To find the unique environmental variable of PID 2464, you need to examine the environment variables of that specific process and identify any that are unusual or unique to it.

```Bash
python vol.py -f ../victim.raw --profile=Win7SP1x64 envars -p 2464
```

[SCREEN12]
