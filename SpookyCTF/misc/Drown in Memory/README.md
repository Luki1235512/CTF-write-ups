# [Drown in Memory](https://spooky.ctfd.io/challenges#Drown%20in%20Memory-14)

180 meters below the surface, your cage opens into darkness. Follow the flickering wreck lights, scavenge oxygen, and reach the submarine before it finds you.

## 1. Unpack the game

```bash
cd "/home/kali/CTF/Drown in Memory"
```

```bash
mkdir -p game && cd game && unzip -q -o ../Drown-in-Memory_Windows.zip
```

```bash
find . -type f | head -30
```

The interesting file is `Abyss/Binaries/Win64/Abyss-Win64-Shipping.exe`. Its hash matters later, because the client sends it to the server:

```bash
sha256sum Abyss/Binaries/Win64/Abyss-Win64-Shipping.exe
```

```
483a48aebd7e0e37973d9dda52516ea81efab2dc5b6af2b1ac4794c1d95e0e84  Abyss/Binaries/Win64/Abyss-Win64-Shipping.exe
```

## 2. What is already known about the client

These facts come from static reverse engineering of the shipping exe (the pak config and the pawn's networking code):

- The server URL is `https://d2xe63kuz25gcn.cloudfront.net`. It is set in the pak ini under `[Abyss]`, and you can override it with the `AbyssServer=` command line argument.
- `POST /v1/session` takes `build_sha256` (the exe hash) and `protocol`. It returns a `token` and a `cookie`.
- `POST /v1/step` takes `sequence`, `capacity`, `guard`, `start`, `sprint`, `hide`, `flashlight` and `direction`, and is authenticated with `Authorization: Bearer <token>`. When the result is `phase == "escaped"`, the response contains `flag`.
- The calibration lives in a small heap struct pointed to by `AAbyssPawn+0x4E0`. `capacity` is at +0x08 (factory value 1.0, client allows 0.5 to 3.0), `guard` at +0x0C, and the session `cookie` (the key) at +0x10. The guard is:

```
guard = float_bits(capacity) XOR cookie XOR 0xA731CF29
```

- The in-game hint says "FINAL GAP EXCEEDS STOCK SUIT RANGE. Inspect the paired propulsion calibration." Capacity and guard are the "paired" values.

## 3. Confirm the server is reachable and the session format is right

```bash
curl -s -i -X POST https://d2xe63kuz25gcn.cloudfront.net/v1/session \
  -H 'Content-Type: application/json' \
  -d '{"build_sha256":"483a48aebd7e0e37973d9dda52516ea81efab2dc5b6af2b1ac4794c1d95e0e84","protocol":1}'
```

The response is HTTP 200 with JSON like:

```json
{"token":"HKTzd8eESvnCNR9Eksb35LpqNeP1n7pf6rDTMs_uINc","cookie":1102963933,"tick_hz":10,"phase":"cage",
 "position":[0.0,0.0,0.0],"shark":[125.0,0.0,1.0],"oxygen":35.0,"oxygen_max":55.0,"tank_refill":20.0,
 "collected_tanks":[],"shark_grace":10.0, ...}
```

A step with `sequence: 0` is rejected with `"Input should be greater than or equal to 1"`, so sequences start at 1 and count up.

## 4. Find the level coordinates in the binary

Server positions are in meters and the shark starts at x=125. Unreal uses centimeters and, in UE5, double precision vectors, so the level layout should show up as doubles in the exe. Every script below goes in its own directory, outside the extracted game folder.

```bash
mkdir -p "/home/kali/CTF/Drown in Memory/solve" && cd "/home/kali/CTF/Drown in Memory/solve"
```

### 4.1 Disassembler helper

`disasm.py` disassembles a range and annotates RIP-relative loads with the doubles and floats they point to.

```python
import struct,sys
from capstone import *
from capstone.x86 import *
p="/home/kali/CTF/Drown in Memory/game/Abyss/Binaries/Win64/Abyss-Win64-Shipping.exe"
d=open(p,"rb").read();B=0x140000000
def raw(va):
    rva=va-B
    if 0x1000<=rva<0x7502000: return rva-0x1000+0x600
    if 0x7502000<=rva<0x9400000: return rva-0x7502000+0x7500800
md=Cs(CS_ARCH_X86,CS_MODE_64); md.detail=True
s=int(sys.argv[1],16); n=int(sys.argv[2])
for ins in md.disasm(d[raw(s):raw(s)+n],s):
    c=""
    for op in ins.operands:
        if op.type==X86_OP_MEM and op.mem.base==X86_REG_RIP:
            t=ins.address+ins.size+op.mem.disp;r=raw(t)
            if r and r>=0x7500800:
                c=" ; [%x] d=%s f=%s"%(t,struct.unpack_from("<2d",d,r),[round(x,4) for x in struct.unpack_from("<4f",d,r)])
        if op.type==X86_OP_IMM and op.size==4 and ins.mnemonic=="mov":
            c+=" ; f=%g"%struct.unpack("<f",struct.pack("<I",op.imm&0xffffffff))[0]
    print(hex(ins.address),ins.mnemonic,ins.op_str,c)
```

The section offsets in `raw()` come from the PE headers:

```python
import pefile
pe=pefile.PE('/home/kali/CTF/Drown in Memory/game/Abyss/Binaries/Win64/Abyss-Win64-Shipping.exe',fast_load=True)
[print(s.Name,hex(s.VirtualAddress),hex(s.Misc_VirtualSize),hex(s.PointerToRawData)) for s in pe.sections]
```

### 4.2 Cross-reference helper

`xref.py` finds code in a VA range whose RIP-relative displacement lands inside a target range. It's a brute force over every byte, which is fine for this purpose.

```python
import struct,sys
p="/home/kali/CTF/Drown in Memory/game/Abyss/Binaries/Win64/Abyss-Win64-Shipping.exe"
d=open(p,"rb").read()
B=0x140000000
lo,hi=int(sys.argv[1],16),int(sys.argv[2],16)
s0,s1=int(sys.argv[3],16)-B-0x1000+0x600, int(sys.argv[4],16)-B-0x1000+0x600
text=d[s0:s1]
va0=B+0x1000+(s0-0x600)
res={}
for k in range(0,len(text)-4):
    disp=struct.unpack_from("<i",text,k)[0]
    for extra in (4,5):
        t=va0+k+extra+disp
        if lo<=t<hi: res.setdefault(va0+k,[]).append(t)
for a,t in sorted(res.items()): print(hex(a),[hex(x) for x in t])
```

### 4.3 Spot the coordinate pool

Dump the `.rdata` constant pool around the game's own strings (`SharkActor`, `OxygenTanks` and so on sit at file offset `0x8172e00`) as pairs of doubles:

```python
import struct
d=open("/home/kali/CTF/Drown in Memory/game/Abyss/Binaries/Win64/Abyss-Win64-Shipping.exe","rb").read()
for lo,hi in [(0x8175800,0x8175b40),(0x8179380,0x8179b40)]:
    for o in range(lo,hi,16):
        a,b=struct.unpack_from("<dd",d,o)
        print(hex(o+0x1800+0x140000000),(round(a,3),round(b,3)))
```

Among the output there are 16-byte constants that look like centimeter coordinates, for example `0x14817b270 (2400.0, -1100.0)`, `0x14817b1b0 (-100.0, 5200.0)` and `0x14817b290 (14100.0, -1300.0)`. These are loaded with `movdqa` as two doubles at a time.

### 4.4 Follow them to the level builder

Find code in the game module that loads from that pool:

```bash
python3 xref.py 0x14817ab80 0x14817b2a0 0x144600000 0x144690000 | head -80
```

Hits cluster in `0x144638010` to `0x144639b59`, the pawn's level-building function. Disassemble it:

```bash
python3 disasm.py 0x144638500 2048 > d2.txt
```

```bash
sed -n '290,330p' d2.txt
```

At `0x144638a40` the function loops over a global TArray of FVectors:

```
0x144638a0d cmp dword ptr [rip + 0x4ea43c4], r13d      ; count at 0x1494dcdd8
0x144638a40 mov rax, qword ptr [rip + 0x4ea4389]       ; data ptr at 0x1494dcdd0
0x144638a47 movups xmm0, xmmword ptr [rcx + rax]
0x144638a4f movsd xmm1, qword ptr [rcx + rax + 0x10]
```

That array is filled at startup, so find who writes `0x1494dcdd0`:

```bash
python3 xref.py 0x1494dcdd0 0x1494dcde0 0x140001000 0x147500000
```

This reports `0x14119c5fe` and `0x14119c617`, inside a static initializer at `0x14119c540`:

```bash
python3 disasm.py 0x14119c540 368
```

```
0x14119c58d movdqa xmm6,  [14817b270] d=(2400.0, -1100.0)
0x14119c59c movdqa xmm7,  [14817b1b0] d=(-100.0, 5200.0)
0x14119c5a7 movdqa xmm8,  [14817b240] d=(1300.0, -140.0)
0x14119c5b5 movdqa xmm9,  [14817b280] d=(8200.0, -1200.0)
0x14119c5c3 movdqa xmm10, [14817b1c0] d=(-80.0, 11100.0)
0x14119c5d2 movdqa xmm11, [14817b230] d=(1100.0, -120.0)
0x14119c5db movdqa xmm12, [14817b290] d=(14100.0, -1300.0)
0x14119c5e4 movdqa xmm13, [14817b1d0] d=(-100.0, 16800.0)
0x14119c5ed movdqa xmm14, [14817b220] d=(900.0, -80.0)
0x14119c602 mov dword ptr [rsp + 0x28], 6               ; 6 elements
...
0x14119c622 movups xmmword ptr [rax], xmm6               ; stored back to back
```

That's 18 doubles stored in order, which makes 6 FVectors. Six matches the number of oxygen tanks:

| Tank | UE (cm)              | Server (m)      |
| ---- | -------------------- | --------------- |
| 0    | (2400, -1100, -100)  | (24, -11, -1)   |
| 1    | (5200, 1300, -140)   | (52, 13, -1.4)  |
| 2    | (8200, -1200, -80)   | (82, -12, -0.8) |
| 3    | (11100, 1100, -120)  | (111, 11, -1.2) |
| 4    | (14100, -1300, -100) | (141, -13, -1)  |
| 5    | (16800, 900, -80)    | (168, 9, -0.8)  |

### 4.5 Find the hiding spots

If you swim the tank route directly, the shark kills you around x=95 with the message `"The shark found you. Hide in a wreck recess with your light off."` The same level builder also walks a static `.data` table between `0x1494dbf60` and `0x1494dcdd0` (`lea rbx, [rip + 0x4ea36a1]` at `0x1446388ec`, stride `0x38`). Dump it:

```python
import struct
d=open("/home/kali/CTF/Drown in Memory/game/Abyss/Binaries/Win64/Abyss-Win64-Shipping.exe","rb").read()
def raw(va): return va-0x140000000-0x9400000+0x93fe400
s=0x1446388f3+0x4ea36a1; e=0x144638903+0x4ea44cd
base=s-0x34; o=raw(base)
for i in range((e-base)//0x38+1):
    b=d[o+i*0x38:o+i*0x38+0x38]
    print(i,[round(x,2) for x in struct.unpack_from("<3d",b,0)],[round(x,2) for x in struct.unpack_from("<3d",b,0x18)])
```

Each tank has 11 wreck pieces next to it. Entries 7 to 10 for tank 0 form a box with a roof at z=-65, a back wall at y=-1660, and side walls at x=2230 and x=2570:

```
7  [2400.0, -1500.0, -65.0]  [3.8, 3.8, 0.18]
8  [2400.0, -1660.0, -180.0] [3.8, 0.18, 2.4]
9  [2230.0, -1500.0, -180.0] [0.18, 3.5, 2.4]
10 [2570.0, -1500.0, -180.0] [0.18, 3.5, 2.4]
```

The box is open on the side that faces the tank. That's the recess. Converted to server units and placed just inside the back wall, the recesses are:

```
(24,-16.3,-1.8) (52,17.3,-1.8) (82,-17.3,-1.8) (111,16.3,-1.8) (141,-18.3,-1.8) (168,14.3,-1.8)
```

Standing there with `hide=true` and `flashlight=false` makes the server report `near_cover`, `sheltered` and `hidden` as true.

The submarine is past the last tank. The pool has constants at 35880 to 38200 cm, and in practice the airlock triggers at about x=356 when you swim toward (375, 0, -1).

## 5. Write the bot

`lib.py` handles the protocol and the guard. The server simulates in wall-clock time, so every request advances the game by however long it took.

```python
import requests,struct,json,time
S="https://d2xe63kuz25gcn.cloudfront.net"
class G:
    def __init__(s,cap=1.0):
        s.h=requests.Session()
        r=s.h.post(S+"/v1/session",json={"build_sha256":"483a48aebd7e0e37973d9dda52516ea81efab2dc5b6af2b1ac4794c1d95e0e84","protocol":1}).json()
        s.init=r;s.tok=r["token"];s.key=r["cookie"];s.seq=0;s.cap=cap
    def step(s,d=(0,0,0),**kw):
        s.seq+=1
        g=struct.unpack("<I",struct.pack("<f",s.cap))[0]^s.key^0xA731CF29
        b=dict(sequence=s.seq,capacity=s.cap,guard=g,start=False,sprint=False,hide=False,flashlight=False,direction=list(d));b.update(kw)
        x=s.h.post(S+"/v1/step",json=b,headers={"Authorization":"Bearer "+s.tok})
        try: return x.json()
        except: return {"err":x.status_code,"t":x.text}
```

`bot.py` adds steering toward a point, waiting with a condition, and logging. It exits and prints the full JSON as soon as the phase leaves `chase`.

```python
from lib import *
import math,sys,json
K="phase position shark oxygen checkpoint reason near_cover hidden sheltered patrol_clear shark_mode aggression collected_tanks stamina exhausted shark_grace".split()
def fmt(r): return {k:([round(x,1) for x in r[k]] if isinstance(r.get(k),list) and r[k] and isinstance(r[k][0],float) else (round(r[k],2) if isinstance(r.get(k),float) else r.get(k))) for k in K}
class Bot:
    def __init__(s,cap=3.0):
        s.g=G(cap); s.r=s.g.step(start=True); s.log=[]
    def st(s,d=(0,0,0),**kw):
        s.r=s.g.step(d,**kw)
        if "err" in s.r: print(s.r); raise SystemExit
        if s.r["phase"] not in("chase",): print("END",json.dumps(s.r)); raise SystemExit
        return s.r
    def goto(s,t,tol=1.0,every=10,maxn=400,**kw):
        for i in range(maxn):
            p=s.r["position"];v=[t[j]-p[j] for j in range(3)];n=math.sqrt(sum(x*x for x in v))
            if n<tol: return
            s.st(tuple(x/n for x in v),**kw)
            if i%every==0: print("go",t,round(n,1),fmt(s.r))
    def wait(s,cond,maxn=300,every=5,**kw):
        for i in range(maxn):
            s.st((0,0,0),**kw)
            if i%every==0: print("wait",fmt(s.r))
            if cond(s.r): return True
        return False
```

`run2.py` is the route.

- After `start`, the shark waits for `shark_grace` (10 s) and then patrols from x=125 toward the cage at about 4.3 m/s.
- The bot grabs tanks 0 and 1, then hides in recess 1 until the shark is 12 m past it toward the cage.
- After that the diver (about 6 m/s) outruns the shark. The bot collects tanks 2 to 5 and swims straight to the submarine.
- Waiting for `patrol_clear` in recess 0 instead runs you out of oxygen, so hiding at recess 1 is the important timing choice.

```python
from bot import *
import sys
T=[(24,-11,-1),(52,13,-1.4),(82,-12,-0.8),(111,11,-1.2),(141,-13,-1),(168,9,-0.8)]
R=[(24,-16.3,-1.8),(52,17.3,-1.8),(82,-17.3,-1.8),(111,16.3,-1.8),(141,-18.3,-1.8),(168,14.3,-1.8)]
b=Bot(float(sys.argv[1]))
b.goto(T[0],every=50); b.goto(T[1],every=50); print("T1",fmt(b.r))
b.goto(R[1],tol=0.4,every=50)
b.wait(lambda r:r["shark"][0]<b.r["position"][0]-12,maxn=400,every=10,hide=True)
print("PASSED",fmt(b.r))
for i in range(2,6):
    b.goto(T[i],every=10); print("TANK",i,fmt(b.r))
tgt=(float(sys.argv[2]),0,-1)
b.goto(tgt,every=5,maxn=2000,sprint=len(sys.argv)>3)
print("ARRIVED",json.dumps(b.r))
b.wait(lambda r:False,maxn=30,every=1)
```

## 6. Run it

The arguments are the capacity (3.0, the client's maximum, which the server accepts with `calibration_ok: true`) and the target x of the submarine. One run takes about 95 seconds of real time. If you restart too often, the server answers "Too many restarts, wait 60s".

```bash
cd "/home/kali/CTF/Drown in Memory/solve" && python3 run2.py 3.0 375 2>&1 | grep -E '^(T1|PASSED|TANK|END)'
```

Output from the solving run (trimmed):

```
T1     {'position': [51.9, 12.9, -1.4], 'shark': [122.4, 0.0, 1.0], 'oxygen': 54.46, 'collected_tanks': [0, 1], ...}
PASSED {'position': [51.9, 17.9, -1.9], 'shark': [39.6, 0.4, 1.0], 'hidden': True, 'sheltered': True, ...}
TANK 2 {'position': [81.6, -11.7, -0.8], 'oxygen': 40.05, 'collected_tanks': [0, 1, 2], ...}
TANK 3 {'position': [110.7, 10.7, -1.2], 'oxygen': 53.81, 'collected_tanks': [0, 1, 2, 3], ...}
TANK 4 {'position': [140.2, -12.4, -1.0], 'oxygen': 54.6, 'collected_tanks': [0, 1, 2, 3, 4], ...}
TANK 5 {'position': [167.2, 8.4, -0.8], 'oxygen': 54.6, 'collected_tanks': [0, 1, 2, 3, 4, 5], ...}
END {"phase": "escaped", "position": [356.37, 0.80, -0.98], "oxygen": 23.01, "elapsed": 95.25,
     "reason": "The submarine airlock seals behind you.", "collected_tanks": [0, 1, 2, 3, 4, 5],
     "flag": "spooky{!NjEc70R-P0wer-u5eR}"}
```

The server returns the flag directly in the escape response, so you don't have to read the illuminated blocks the client would draw.

(The flag reads as "Injector Power User", a nod to the intended solve of injecting into the client's memory.)

## Note on the intended path

On Windows the same result comes from Cheat Engine or a small WriteProcessMemory script. Scan for the bytes `03 00 01 00 00 00 C8 42 00 00 80 3F` (the start of the calibration struct). Then write capacity and guard together as one 8-byte write at struct+8: `3.0f` and `0x40400000 ^ [struct+0x10] ^ 0xA731CF29`. Do it after every new session, because each session gets a fresh key. If the guard doesn't match, the client resets capacity to 1.0 and shows "Suit calibration inconsistent".
