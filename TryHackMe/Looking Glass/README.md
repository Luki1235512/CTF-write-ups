# [Looking Glass](https://tryhackme.com/room/lookingglass)

## Step through the looking glass. A sequel to the Wonderland challenge room.

# Looking Glass

## Climb through the Looking Glass and capture the flags.

### Get the user flag.

_O(log n) A looking glass is a mirror._

1. Start with a full port scan to identify all open ports on the target

```bash
nmap -p- <TARGET_IP>
```

This reveals numerous SSH ports in the range 9000-13999.

2. When connecting to various ports, the SSH service responds with either `Higher` or `Lower`, indicating which direction to search. This is a binary search hint - the challenge expects an O(log n) approach.

```bash
ssh -o HostkeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa <TARGET_IP> -p 13243
```

3. Implement a binary search algorithm to find the correct port efficiently:

```python
import subprocess
import os

TARGET_IP = "<TARGET_IP>"
MIN_PORT = 9000
MAX_PORT = 13999

def check_port(port):
    try:
        cmd = [
            'ssh',
            '-o', 'StrictHostKeyChecking=no',
            '-o', 'UserKnownHostsFile=/dev/null',
            '-o', 'HostkeyAlgorithms=+ssh-rsa',
            '-o', 'PubkeyAcceptedAlgorithms=+ssh-rsa',
            '-o', 'ConnectTimeout=5',
            '-o', 'LogLevel=ERROR',
            '-p', str(port),
            TARGET_IP
        ]

        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            timeout=10,
            stdin=subprocess.DEVNULL
        )

        output = result.stdout + result.stderr

        if "Higher" in output:
            return "Higher"
        elif "Lower" in output:
            return "Lower"
        elif output.strip() and "Connection" not in output:
            return output.strip()

        return None

    except subprocess.TimeoutExpired:
        return None
    except Exception as e:
        return None

def binary_search():
    low = MIN_PORT
    high = MAX_PORT
    attempts = 0

    while low <= high:
        mid = (low + high) // 2
        attempts += 1

        print(f"[*] Attempt {attempts}: Trying port {mid}...")

        response = check_port(mid)

        print(f"[+] Port {mid}: {response}")

        if response == "Lower":
            low = mid + 1
        elif response == "Higher":
            high = mid - 1
        else:
            return mid

    return None

if __name__ == "__main__":
    print(f"[*] Searching for correct SSH port on {TARGET_IP}")
    print(f"[*] Range: {MIN_PORT}-{MAX_PORT}")
    print()

    binary_search()
```

[SCREEN01]

```
You've found the real service.
Solve the challenge to get access to the box
Jabberwocky
'Mdes mgplmmz, cvs alv lsmtsn aowil
Fqs ncix hrd rxtbmi bp bwl arul;
Elw bpmtc pgzt alv uvvordcet,
Egf bwl qffl vaewz ovxztiql.

'Fvphve ewl Jbfugzlvgb, ff woy!
Ioe kepu bwhx sbai, tst jlbal vppa grmjl!
Bplhrf xag Rjinlu imro, pud tlnp
Bwl jintmofh Iaohxtachxta!'

Oi tzdr hjw oqzehp jpvvd tc oaoh:
Eqvv amdx ale xpuxpqx hwt oi jhbkhe--
Hv rfwmgl wl fp moi Tfbaun xkgm,
Puh jmvsd lloimi bp bwvyxaa.

Eno pz io yyhqho xyhbkhe wl sushf,
Bwl Nruiirhdjk, xmmj mnlw fy mpaxt,
Jani pjqumpzgn xhcdbgi xag bjskvr dsoo,
Pud cykdttk ej ba gaxt!

Vnf, xpq! Wcl, xnh! Hrd ewyovka cvs alihbkh
Ewl vpvict qseux dine huidoxt-achgb!
Al peqi pt eitf, ick azmo mtd wlae
Lx ymca krebqpsxug cevm.

'Ick lrla xhzj zlbmg vpt Qesulvwzrr?
Cpqx vw bf eifz, qy mthmjwa dwn!
V jitinofh kaz! Gtntdvl! Ttspaj!'
Wl ciskvttk me apw jzn.

'Awbw utqasmx, tuh tst zljxaa bdcij
Wph gjgl aoh zkuqsi zg ale hpie;
Bpe oqbzc nxyi tst iosszqdtz,
Eew ale xdte semja dbxxkhfe.
Jdbr tivtmi pw sxderpIoeKeudmgdstd
Enter Secret:   Incorrect secret.
```

4. This is a Vigenère cipher. Use [Vigenère Cipher Autosolver](https://www.boxentriq.com/code-breaking/vigenere-cipher-autosolver) or similar tools to decrypt.

The key is `THEALPHABETCIPHER`, and the plaintext reveals:

```
'Twas brillig, and the slithy toves
Did gyre and gimble in the wabe;
All mimsy were the borogoves,
And the mome raths outgrabe.

'Beware the Jabberwock, my son!
The jaws that bite, the claws that catch!
Beware the Jubjub bird, and shun
The frumious Bandersnatch!'

He took his vorpal sword in hand:
Long time the manxome foe he sought--
So rested he by the Tumtum tree,
And stood awhile in thought.

And as in uffish thought he stood,
The Jabberwock, with eyes of flame,
Came whiffling through the tulgey wood,
And burbled as it came!

One, two! One, two! And through and through
The vorpal blade went snicker-snack!
He left it dead, and with its head
He went galumphing back.

'And hast thou slain the Jabberwock?
Come to my arms, my beamish boy!
O frabjous day! Callooh! Callay!'
He chortled in his joy.

'Twas brillig, and the slithy toves
Did gyre and gimble in the wabe;
All mimsy were the borogoves,
And the mome raths outgrabe.
Your secret is bewareTheJabberwock
```

The secret password is **`bewareTheJabberwock`**.

[SCREEN02]

5. Connect to the discovered SSH port and enter the secret:

```bash
ssh -o HostkeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa <TARGET_IP> -p 13243
```

When prompted, enter: `bewareTheJabberwock`

[SCREEN03]

6. Use these credentials to connect as the `jabberwock` user:

```bash
ssh jabberwock@10.10.45.13 -p 22
cat user.txt
```

7. To get the user flag, you need to reverse the string.

[SCREEN04]
