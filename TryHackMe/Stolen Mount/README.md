# [Stolen Mount](https://tryhackme.com/room/hfb1stolenmount)

## Analyse network traffic related to an unauthenticated file share access attempt, focusing on potential signs of data exfiltration.

An intruder has infiltrated our network and targeted the NFS server where the backup files are stored. A classified secret was accessed and stolen. The only trace left behind is a packet capture (PCAP) file recorded during the incident. Your mission, should you accept it, is to discover the contents of the stolen data.

The packet capture (**challenge.pcapng**) is stored in the **~/Desktop** directory.

### What is the flag?

1. **Open the capture in Wireshark and locate the file access traffic**

Open `challenge.pcapng` in Wireshark and apply the display filter: `nfs.opcode == 68`

NFS opcode 68 corresponds to the `READ` procedure call, so this filter isolates every packet where a client requested data from a file on the NFS share. Since the attacker mounted the share without authentication, stepping through these `READ` calls shows exactly which file was being pulled off the server and in what order the file's contents were transferred.

Scroll through the filtered results. The final packet in this sequence corresponds to the read of a file named `secrets.png`. This is the file the intruder exfiltrated.

<img width="1402" height="837" alt="SCREEN01" src="https://github.com/user-attachments/assets/bd44e7cc-0990-4619-93f3-91656917f9e2" />

2. **Extract the raw file data from the packet**

Right-click the packet containing the last chunk of `secrets.png`, then select: `Copy -> ... as Hex Dump`

This copies the full hex + ASCII representation of the packet's bytes to the clipboard, which will let us reconstruct the transferred file outside of Wireshark.

3. **Reassemble the file with CyberChef**

Head to [CyberChef](https://gchq.github.io/CyberChef/) and paste the hex dump into the input pane. Then build the following recipe: `From Hexdump -> Extract Files`

- **From Hexdump** converts the pasted text back into raw bytes.
- **Extract Files** scans those raw bytes for known file signatures and carves out any embedded files it finds.

4. **Save the extracted archive**

CyberChef should detect and carve out a ZIP archive from the payload, listed as something like `extracted_at_0xB2.zip`. This archive contains the stolen data, but it is password-protected.

5. **Recover the ZIP password from the capture**

The password used to protect the archive was also sent over the network.

Go back to Wireshark and inspect the second packet in the original `nfs.opcode == 68` filtered list. Following the stream from that point reveals the password, which was disclosed as an MD5 hash.

<img width="1403" height="837" alt="SCREEN02" src="https://github.com/user-attachments/assets/bf629e81-f4bf-42af-aede-69985bca5e32" />

**Results:** `90eb7723a657b6597100aafef171d9f2 (md5)`

6. **Crack the MD5 hash**

Submit the hash to [CrackStation](https://crackstation.net/), which checks it against large precomputed dictionaries/rainbow tables of common passwords. Use the plaintext value to unlock the ZIP archive.

7. **Open the archive and inspect the contents**

Extract `extracted_at_0xB2.zip` using the recovered password. Inside is `secrets.png`, which when opened turns out to be an image of a QR code.

8. **Decode the QR code to obtain the flag**

Upload or scan `secrets.png` using any QR code reader/decoder. Decoding the image reveals the flag text.

<img width="1284" height="552" alt="SCREEN03" src="https://github.com/user-attachments/assets/8aafe689-8de0-4eb3-9646-7d320b1da4dd" />
