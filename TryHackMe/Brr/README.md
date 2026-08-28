# [Brr](https://tryhackme.com/room/brr)

## You've been called in to assess an OT environment.

The cold never lies, but the SCADA panel guarding it just handed you the keys. Chase the chill all the way down.

### What's the flag?

1. Full port scan with service/version detection

Start with a full TCP port sweep against every port combined with service/version detection and default NSE scripts.

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE      VERSION
22/tcp   open  ssh          OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 b1:11:13:0f:5a:ca:16:24:03:20:4d:e6:53:3e:0e:17 (ECDSA)
|_  256 5f:28:77:79:a5:d4:c4:c2:2a:28:d2:f5:85:6c:58:bf (ED25519)
80/tcp   open  http         WebSockify Python/3.12.3
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 405 Method Not Allowed
|     Server: WebSockify Python/3.12.3
|     Date: Thu, 27 Aug 2026 20:18:26 GMT
|     Connection: close
|     Content-Type: text/html;charset=utf-8
|     Content-Length: 355
|     <!DOCTYPE HTML>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 405</p>
|     <p>Message: Method Not Allowed.</p>
|     <p>Error code explanation: 405 - Specified method is invalid for this resource.</p>
|     </body>
|     </html>
|   HTTPOptions:
|     HTTP/1.1 501 Unsupported method ('OPTIONS')
|     Server: WebSockify Python/3.12.3
|     Date: Thu, 27 Aug 2026 20:18:26 GMT
|     Connection: close
|     Content-Type: text/html;charset=utf-8
|     Content-Length: 360
|     <!DOCTYPE HTML>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 501</p>
|     <p>Message: Unsupported method ('OPTIONS').</p>
|     <p>Error code explanation: 501 - Server does not support this operation.</p>
|     </body>
|     </html>
|   RTSPRequest:
|     <!DOCTYPE HTML>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 400</p>
|     <p>Message: Bad request version ('RTSP/1.0').</p>
|     <p>Error code explanation: 400 - Bad request syntax or unsupported method.</p>
|     </body>
|_    </html>
|_http-server-header: WebSockify Python/3.12.3
|_http-title: Error response
5020/tcp open  zenginkyo-1?
5901/tcp open  vnc          VNC (protocol 3.8)
| vnc-info:
|   Protocol version: 3.8
|   Security types:
|     VeNCrypt (19)
|     VNC Authentication (2)
|   VeNCrypt auth subtypes:
|     Unknown security type (2)
|_    VNC auth, Anonymous TLS (258)
8080/tcp open  http         Apache Tomcat/Coyote JSP engine 1.1
|_http-title: ScadaBR CTF
|_http-server-header: Apache-Coyote/1.1
|_http-open-proxy: Proxy might be redirecting requests
| http-methods:
|_  Potentially risky methods: PUT DELETE
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port80-TCP:V=7.99%I=7%D=8/27%Time=6A909B93%P=x86_64-pc-linux-gnu%r(GetR
...
SF:d\x20method\.</p>\n\x20\x20\x20\x20</body>\n</html>\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- **22/tcp** - standard OpenSSH, no immediate foothold without credentials.
- **80/tcp** - a WebSockify proxy. This is almost certainly the bridge that lets a browser reach the VNC service on 5901.
- **5020/tcp** - unidentified by nmap, but this port is the default listener for the Modbus TCP protocol in many ScadaBR/OT lab setups. Worth revisiting once we have a way to talk to it.
- **5901/tcp** - a real VNC server, offering both VNC Authentication and VeNCrypt. This is likely the actual remote framebuffer for whatever HMI/panel we're after, exposed to the browser via the WebSockify proxy on port 80.
- **8080/tcp** - an Apache Tomcat instance titled "ScadaBR CTF". ScadaBR is an open-source SCADA/HMI web application, and it's the obvious entry point since it's a full web app and PUT/DELETE are flagged as available HTTP methods.

2. Logging into ScadaBR with default credentials

Browsing to the login page and trying the well-known ScadaBR default credentials worked immediately. The instance had never been hardened:

```
http://<TARGET_IP>:8080/ScadaBR/login.htm
```

Credentials: `admin:admin`

This grants full administrative access to the ScadaBR dashboard, including data source configuration, data point management, and the ability to view live/point values coming from connected devices.

3. Inspecting the configured data source

With admin access, the next step is to look at what data sources ScadaBR has configured. These represent the industrial protocol connections it's polling. Navigating to the edit page for data source ID 1 shows its configuration:

```
http://<TARGET_IP>:8080/ScadaBR/data_source_edit.shtm?dsid=1
```

This confirms the data source is a Modbus connection pointing at the internal PLC/simulator, and lists the data points/registers being polled. This is effectively the map of "what the cold storage system's sensors and controls look like" from ScadaBR's perspective.

[SCREEN01]

From here, drilling into the associated data points exposes the raw register table below. A set of 16-bit register addresses each holding a 4-digit hex word, read directly off the Modbus device.

4. Decoding the leaked registers

The register values below are almost certainly text, since each one is a small hex value that maps cleanly into the printable ASCII range when read as a 16-bit big-endian word:

```
0 ==> 0054
1 ==> 0048
2 ==> 004d
3 ==> 007b
4 ==> 006d
5 ==> 006f
6 ==> 0064
7 ==> 0062
8 ==> 0075
9 ==> 0073
10 ==> 005f
11 ==> 0068
12 ==> 0069
13 ==> 0064
14 ==> 007d
15 ==> 0000
16 ==> 0000
17 ==> 0000
18 ==> 0000
19 ==> 0000
```

Decoding each register as a single ASCII byte gives:

[SCREEN02]
