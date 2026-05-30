# [Checkmate](https://tryhackme.com/room/checkmate)

## Exploit weak password practices across Marco’s internal systems to achieve full compromise.

# Challenge

Marco Bianchi, a systems administrator, recently deployed several internal services, including a firewall console, employee portal, social platform, and SSH access to critical infrastructure. Due to tight deadlines and operational pressure, Marco reused weak, predictable, and pattern-based passwords across multiple systems.

Your objective is to conduct a password security assessment to identify weaknesses in Marco’s authentication practices.

Start by accessing the main application at http://<TARGET_IP>:5000. From there, you will be guided through each stage of the challenge, uncovering and exploiting weaknesses in Marco’s password usage.

### What is the password for Level 1?

Marco deployed a firewall at firewall.thm:5001 but kept default credentials.

1. Navigate to `http://<TARGET_IP>:5001` in your browser. You are presented with a login form for a firewall administration console.

2. The service was deployed without changing the default credentials. Try the most common default combination. Username `admin` and password `12345`. The login succeeds immediately.

**Answer:** `12345`

---

### What is the password for Level 2?

Marco built an internal Employee Login panel on jobs.thm:5002 and used common company keywords as passwords.

1. Navigate to `http://<TARGET_IP>:5002`. The page displays an employee portal with a company description.

2. Since the hint tells us the password is one of the words in the company description, use these keywords and brute-force the login form.

**Answer:** `excellence`

---

### What is the password for Level 3?

Navigate to social.thm:5003 and derive Marco's password from personal info.

1. Log in to the employee portal at `http://<TARGET_IP>:5002` using the credentials found in the previous level, then navigate to Marco's profile page at `http://<TARGET_IP>:5002/profile`. This reveals his personal details:

```
Employee Details
First Name: Marco
Surname: Bianchi
Nickname: marky
Birthdate (DDMMYYYY): 14021995
```

2. Use `CUPP` to generate a targeted wordlist based on Marco's personal information. CUPP builds a dictionary using common password patterns: name variations, birthdate combinations, appended numbers and special characters, etc.

```bash
cupp -i
```

3. Use Hydra to brute-force the login form at `http://<TARGET_IP>:5003` with the generated wordlist:

```bash
hydra -l marco -P marco.txt <TARGET_IP> -s 5003 -t 32 -f -V http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"
```

**Answer:** `Bianchi2495`

---

### What is the password for Level 4?

On social.thm:5003, Marco recently uploaded a new profile picture. For privacy and storage consistency, the platform automatically renames uploaded files to the SHA256 hash of the original filename and saves them in the format (SHA256).png. Your task is to identify the original filename of Marco’s uploaded profile picture. Submit only the filename to proceed.

1. After logging in to h`ttp://<TARGET_IP>:5003`, download Marco's profile. The image filename is the SHA256 hash of the original filename. Save this hash to a file.

2. Use John the Ripper with the `Raw-SHA256` format and the `rockyou.txt` wordlist. Because the hash is of a filename rather than a password, it is likely a short, common English word that will appear in any standard wordlist:

```bash
john hash --format=Raw-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt
```

**Answer:** `family`

---

### What is the password for Level 5?

Marco has revealed his password pattern on social.thm:5003, using predictable rules based on keywords and formatting. Use this information to generate a targeted wordlist and brute-force the SSH service with username marco.

1. After logging in to `http://<TARGET_IP>:5003`, read Marco's post on his profile:

```
My tip for strong passsord: I take a company keyword, capitalize it, then append the year like 2024 or any other number and an exclamation mark.
security excellence innovation digital cloud
```

1. Create a bash script to generate all possible combinations of the five company keywords with years ranging from 2000 to 2030:

```sh
for word in security excellence innovation digital cloud; do
  for year in $(seq 2000 2030); do
    echo "${word^}${year}!"
  done
done > wordlist.txt
```

2. Run the script to produce `wordlist.txt`:

```bash
bash script.sh
```

3. Use Hydra to brute-force the SSH service on port 22 with the generated wordlist:

```bash
hydra -l marco -P wordlist.txt -s 22 ssh://<TARGET_IP>
```

**Answer:** `Security2024!`
