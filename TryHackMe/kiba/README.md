# [kiba](https://tryhackme.com/room/kiba)

## Identify the critical security flaw in the data visualization dashboard, that allows execute remote code execution.

# Flags

### What is the vulnerability that is specific to programming languages with prototype-based inheritance?

**Answer:** Prototype pollution

---

### What is the version of visualization dashboard installed in the server?

1. Perform a port scan:

```bash
nmap -sV -p- <TARGET_IP>
```

**Scan Results:**

```
PORT     STATE SERVICE      VERSION
22/tcp   open  ssh          OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http         Apache httpd 2.4.18 ((Ubuntu))
5044/tcp open  lxi-evntsvc?
5601/tcp open  http         Elasticsearch Kibana (serverName: kibana)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Navigate to the Management section at `http://<TARGET_IP>:5601/app/kibana#/management?_g=()`. The version information is displayed in the interface as **6.5.4**.

[SCREEN01]

---

### What is the CVE number for this vulnerability? This will be in the format: CVE-0000-0000

**Answer:** [CVE-2019-7609](https://nvd.nist.gov/vuln/detail/cve-2019-7609)
