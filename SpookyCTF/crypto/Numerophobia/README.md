# [Numerophobia](https://spooky.ctfd.io/challenges#Numerophobia-8)

I went exploring the house of the crazy old man on the corner of the street. He disappeared last weekend and nobody has seen him since. Rumor has it that he was last seen walking out at midnight screaming random numbers in tongues never spoken before. I decided to go and investigate, unsurprisingly the front door was left unlocked. On the way to the stairs I almost tripped over a stack of books, one was about dyslexia I think. For some reason every other step was missing and marked with a 13 on the wall. I finally reached the top and headed towards the light at the end of the hallway. Taped on the wall was a piece of paper…

1. **Ignore lines 1 and 3.** They are repeating filler (`50 65 ...` and `63 52 ...`). Only line 2 holds data.

2. **Spot the reversal.** The last values, `136 121 120 111 124 112 128`, look like `{ y k o o p s`. That is `spooky{` backwards, which fits the dyslexia hint.

3. **Spot the offset.** In that tail, every second value is its ASCII code plus 13 (`128` is `s`+13, `124` is `o`+13, `120` is `k`+13, `136` is `{`+13). This matches the 13 on the missing stairs. The shifted values are at the even 0-based indices (the 1st, 3rd, 5th values, and so on).

4. **Undo both steps.** Subtract 13 from the shifted values, convert to characters, then reverse the string.

```python
L=[138,50,65,95,123,51,64,98,108,53,134,52,132,108,65,95,66,52,117,95,127,51,132,53,123,52,108,51,117,116,136,121,120,111,124,112,128]
print(''.join(chr(x-13) if i%2==0 else chr(x) for i,x in enumerate(L))[::-1])
```

[SCREEN01]
