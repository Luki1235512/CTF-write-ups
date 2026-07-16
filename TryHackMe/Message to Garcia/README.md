# [Message to Garcia](https://tryhackme.com/room/messagetogarcia)

## Encryption and Key Management Challenge.

# A message worth delivering

It was 1899 when "[A Message to Garcia](https://courses.csail.mit.edu/6.803/pdf/hubbard1899.pdf)" became a powerful metaphor for reliability, initiative, and getting the job done: no excuses, no unnecessary questions, just execution.

"My heart goes out to the man who does his work when the "boss" is away, as well as when he is home. And the man who, when given a letter for Garcia, quietly takes the missive, without asking any idiotic questions, and with no lurking intention of chucking it into the nearest sewer, or of doing aught else but deliver it, never gets “laid off,” nor has to go on strike for higher wages. Civilisation is one long anxious search for just such individuals. Anything such a man asks will be granted; his kind is so rare that no employer can afford to let him go. He is wanted in every city, town, and village - in every office, shop, store, and factory. The world cries out for such; he is needed, and needed badly - the man who can carry a message to Garcia." (Elbert Hubbard, 1899)

The story describes how a soldier, Lt. Rowan, was entrusted with a mission: deliver an important message to General Garcia. Instead of getting bogged down with questions about Garcia's location or the route to take, Rowan simply takes action and accomplishes the mission with the limited information he has. He got it done using his resourcefulness to fill in the gaps. Does that sound familiar? Much like how we researchers and security professionals operate, we often work with limited information and find creative ways to identify vulnerabilities and secure systems.

### What was the name of the Lieutenant who delivered the message to Garcia?

**Answer:** `Rowan`

# Challenge

## Now, you find yourself in a similar situation as Lt. Rowan, except instead of a handwritten note, the message you need to deliver is unknown, via a secure file transfer server (SFTP). Can you find your way in?

### What type of encryption is the service using?

1. The first move, as with any box, is a full port scan to map out the attack surface.

```bash
$ nmap -p- -sVC <TARGET_IP>

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 dc:35:15:1f:f9:66:f4:e1:03:e6:a5:2c:62:9e:06:38 (ECDSA)
|_  256 4a:f8:b4:42:fa:98:8e:b5:c2:84:a1:83:8e:5b:9c:a8 (ED25519)
80/tcp   open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: SFTP | Home
5000/tcp open  http    Werkzeug httpd 3.1.3 (Python 3.12.3)
|_http-server-header: Werkzeug/3.1.3 Python/3.12.3
|_http-title: SFTP | Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. In `http://<TARGET_IP>/fetch` fetch for `file://README.md`:

**Results:**

````md
File content:

# SecureComms Platform

Enterprise-grade secure file transfer and messaging system for Garcia Communications Inc.

## Overview

SecureComms is an internal platform designed for secure document exchange and encrypted messaging between authorized field operatives and command personnel. The system implements military-grade encryption for all classified communications and provides secure backup functionality for mission-critical intelligence files.

## Operational History

- **2019**: Initial deployment for Garcia Communications field operations
- **2020**: Expanded to support multi-theater intelligence coordination
- **2021**: Enhanced encryption protocols following security audit
- **2022**: Integration with SFTP services for automated file transfers
- **2023**: Backup system implementation for operational continuity
- **2024**: Resource fetcher module added for external intelligence validation
- **2025**: Migration to Fernet encryption for improved compatibility

The platform has successfully facilitated over 2,847 classified message exchanges across 23 countries, maintaining a 99.97% delivery success rate with zero security breaches.

## System Architecture

- **Frontend**: Modern web interface with TailwindCSS
- **Backend**: Flask-based API with Python 3.x
- **Security**: Fernet symmetric encryption for all message content
- **Storage**: Local filesystem with backup capabilities
- **Monitoring**: SFTP server integration for file transfers

## Quick Start

### Installation

```bash
# Install dependencies
pip install -r requirements.txt



## Configuration

### Environment Setup
- Python 3.8+ runtime environment
- Flask web framework
- Cryptography library for encryption operations

### Network Configuration
- Web interface: Port 5000 (HTTP)
- SFTP service: Port 2222
- File upload limits: 5MB maximum
- Accepted file types: `.gpg`, `.enc`

## Features

### Message Upload System
Secure message delivery system accepting encrypted files. Messages must be properly encrypted with the correct encryption key and contain valid Garcia Standard format content.

**Technical Note**: Message validation logic is implemented in the core application `functions.py` module. System administrators can reference the validation routines for troubleshooting message format issues.

### Intelligence Backup System
Navigate to `/backup` for secure backup functionality. Field operatives can upload classified documents using custom file paths to maintain operational compartmentalization.

### External Resource Validation
Navigate to `/fetch` for advanced reconnaissance module. Supports multiple protocols for validating external intelligence sources and communication channels.

### SFTP Server Integration
Built-in SFTP server for automated file transfers and batch processing operations. Access status monitoring at `/status` endpoint.

## Message Protocol

All classified communications follow the "Garcia Standard" message format established in operational directive 2847-Alpha. Messages must contain:
- Operational codename acknowledgment
- Geographic coordinates for rendezvous
- Mission cipher confirmation
- Proper authentication markers

## Security

- **Encryption**: Fernet symmetric encryption (AES 128-bit)
- **Key Management**: Centralized key storage

## File Structure

.
├── app.py # Main Flask application
├── functions.py # Core encryption and validation logic
├── sftp_server.py # SFTP server implementation
├── create_message.py # Utility to generate encrypted messages
├── requirements.txt # Python dependencies
├── templates/ # HTML templates
├── static/ # CSS, JavaScript, fonts
└── uploads/ # File upload directory
```

## Development

Run with debug mode:

```bash
python3 app.py
```

For production deployment, set the environment variable:

```bash
export FLASK_ENV=production
python3 app.py
```

## Troubleshooting

### Module Import Errors

```bash
# Reinstall dependencies
pip install -r requirements.txt
```

## Current Operations

- **OPERATION COORDINATE**: Active intelligence gathering in Mediterranean theater
- **OPERATION CIPHER**: Cryptographic key distribution to embedded assets
- **OPERATION GARCIA**: Ongoing secure communication with field operatives

**Security Clearance Required**: All personnel must maintain SECRET level clearance or higher for system access.

---

**Last Updated**: January 2025
**Maintainer**: Garcia IT Security Division
**Classification**: CONFIDENTIAL
````

The `/fetch` SSRF confirms the whole class of encryption in play. It's a single shared secret used for both encrypting and decrypting.

**Answer:** `symmetric`

---

### What implementation of authenticated encryption is the service using?

The README states it directly under **Security**: "_Encryption: Fernet symmetric encryption (AES 128-bit)._" Fernet is a well-known Python authenticated symmetric encryption scheme. It combines AES-128 in CBC mode with a SHA256-based HMAC for integrity/authenticity, plus a timestamp, all packaged into one URL-safe base64 token. That HMAC is what makes it "authenticated" rather than just "encrypted" tampering with a Fernet token invalidates it rather than silently decrypting to garbage.

**Answer:** `Fernet`

---

### What is the encryption key?

With the SSRF confirmed, the next logical step is to pull the two files the README pointed us to: `functions.py` and `create_message.py`. Fetch `file://create_message.py`:

**Results:**

```py
File content:
#!/usr/bin/env python3
"""
Simple script to create the encrypted message for the challenge.
"""
from cryptography.fernet import Fernet

# The encryption key (same as in functions.py)
ENCRYPTION_KEY = b'TUVTU0FHRVRPR0FSQ0lBMjAyNF9LRVkhISEhISEhISE='
cipher = Fernet(ENCRYPTION_KEY)

# The expected message
message = "Garcia, it seems I've cracked the code!! I need you to meet me at coordinates: 40.4168° N, 3.7038° W. The cipher is: TRACK"

print(f"[*] Message to encrypt: {message}")
print(f"[*] Encryption key: {ENCRYPTION_KEY.decode()}")

# Encrypt the message
encrypted = cipher.encrypt(message.encode('utf-8'))

# Save to file
with open("message.enc", "wb") as f:
    f.write(encrypted)

print(f"\n[+] Encrypted message saved to: message.enc")
print(f"[+] File size: {len(encrypted)} bytes")

# Test decryption
decrypted = cipher.decrypt(encrypted)
print(f"\n[*] Testing decryption...")
print(f"[+] Decrypted: {decrypted.decode('utf-8')}")
print(f"[+] Match: {decrypted.decode('utf-8').strip() == message}")

print(f"\n{'='*60}")
print(f"SUCCESS! Upload 'message.enc' to the application!")
print(f"{'='*60}")
```

This confirms `functions.py` and `create_message.py` share the exact same hardcoded `ENCRYPTION_KEY`, which is exactly what's needed. This is the same key the server-side validation logic uses to decrypt uploaded `.enc` files:

**Amswer:** `TUVTU0FHRVRPR0FSQ0lBMjAyNF9LRVkhISEhISEhISE=`

---

### What is the message that needs to be delivered?

The plaintext message is right there in `create_message.py`, assigned to the message variable before it gets encrypted:

**Answer:** `Garcia, it seems I've cracked the code!! I need you to meet me at coordinates: 40.4168° N, 3.7038° W. The cipher`

---

### What's the flag?

1. Copy `create_message.py` locally and run it.

2. Upload the generated `message.enc` through the web UI's message upload form at `http://<TARGET_IP>/`. Since the token is a valid Fernet ciphertext produced with the correct key, and the plaintext matches the "Garcia Standard" format the backend expects, the server successfully decrypts and validates it, and returns the flag.

[SCREEN01]
