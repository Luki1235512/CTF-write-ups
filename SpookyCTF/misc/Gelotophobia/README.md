# [Gelotophobia](https://spooky.ctfd.io/challenges#Gelotophobia-9)

Trying to make someone laugh? Then check out this website for some awesome jokes!

this challenge contains ... bright, flashing, strobe lights. Those who are seizure-prone / have epilepsy are advised to skip this challenge.

**1. Read the page source.** The button picks a random entry from `textData` in `testindex2.js` and navigates to `/<entry>`.

```bash
curl -s http://18.217.64.238/testindex2.js -o testindex2.js
```

**2. Decode the entries.** Each entry is base64 of a string of `E` and `e`. Read `E` as 0 and `e` as 1, and group the bits into 16-bit units (UTF-16BE). The entries decode to short words such as `pirate`, `oven` and `tool`. Requesting an entry returns a joke for that word, so the server decodes the path and looks the word up.

**3. Build your own path.** Reverse the encoding to request any word:

```python
import sys, base64
w = sys.argv[1]
bits = ''.join(format(b, '08b') for b in w.encode('utf-16-be'))
print(base64.b64encode(''.join('e' if c == '1' else 'E' for c in bits).encode()).decode())
```

Check that `oven` encodes to the same string as the page's second entry.

**4. Request a word the button never sends.** The list has no `flag` entry, so try it:

```bash
curl -s http://18.217.64.238/$(python3 enc.py flag) | grep -A1 '<h3>'
```

The `<h3>` contains the flag:

[SCREEN01]
