# [Hemophobia](https://spooky.ctfd.io/challenges#Hemophobia-24)

You are trapped in this endless nightmare with music and blood seeping in through the ceiling, crawling down the walls. Once the blood reaches the bottom, your feet become rooted to the ground as the blood consumes you, and you relive it all over again. To escape you must find the key hidden within these walls, and only then can you escape this nightmare. Goodluck...

1. Open the page source. The HTML comment points to `nightmare.txt`:

```html
<!-- There's a song that plays in this nightmare ... I think it's in nightmare.txt ... The flag looks like spooky{band} -->
```

2. Fetch the file:

```bash
curl -s http://77.112.83.127/nightmare.txt
```

It gives the song title, "Roots Bloody Roots", and asks "Who made it?".

3. "Roots Bloody Roots" is by Sepultura, so the flag follows the `spooky{band}` format.

4. Optional check: `script.js` compares a client-side hash of the lowercased input to `FLAG_HASH = '6423651715153126'`. Running the page's own `hash()` function on `spooky{sepultura}` gives that value.
