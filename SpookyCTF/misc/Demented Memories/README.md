# [Demented Memories](https://spooky.ctfd.io/challenges#Demented%20Memories-39)

my memories are leaving me, i missed the days of the past, help me recovered my fragmented parts of the past.

## 1. Extract the archive

## 2. List everything, including hidden files

```bash
ls -la
```

This shows 71 `fogN.fog` files and `fog.fog`, plus six dotfiles: `.fog11.fog`, `.fog21.fog`, `.fog31.fog`, `.fog41.fog`, `.fog51.fog` and `.fog61.fog`. There is also a `.git` directory, so the challenge is a git repo.

## 3. Part 3: the stray file in `.git`

A normal git directory has no `errored_space.txt`, so it's worth opening.

```bash
cat .git/errored_space.txt
```

Output:

```
61, the year the memories cuts through the fog... th3_void} (3)
```

This gives the end of the flag, `th3_void}`. It's labelled part 3, and the `61` points at `.fog61.fog`.

## 4. Part 2: the hex-encoded hidden file

```bash
cat .fog61.fog
```

Output:

```
7468655f6c3073745f406e6420283229
```

That is 32 hex characters, the same length as an MD5 hash. That is why the other `fogN.fog` files look like hashes. Decode it:

```bash
python3 -I -c "import binascii; print(binascii.unhexlify('7468655f6c3073745f406e6420283229'))"
```

Output:

```
b'the_l0st_@nd (2)'
```

This is part 2, `the_l0st_@nd`. The other hidden dotfiles (`.fog11`, `.fog21`, `.fog31`, `.fog41`, `.fog51`) decode to random bytes. The numbered `fogN.fog` files and `fog.fog` are the same kind of hash filler. None of them are needed.

## 5. Part 1: the deleted file in git history

```bash
git log --all --oneline
```

Output:

```
0f00424 MEMORY WIPE INITIALIZATION
4987681 the weight of the fog
f7c00b3 Create retrospective.c
0926e96 added core memories
deab4f3 Initial commit
```

Commit `0f00424` is the "wipe". Its stat shows what it removed:

```bash
git log --all --stat --format='--- %h %s' | head -5
```

Output:

```
--- 0f00424 MEMORY WIPE INITIALIZATION

 retrospective.c | 55 -------------------------------------------------------
 1 file changed, 55 deletions(-)
```

It deleted `retrospective.c`. That file was added in `f7c00b3`, so print it from there:

```bash
git show f7c00b3:retrospective.c
```

The last line of the file is a comment containing part 1:

```
// miss the days when people cared about optimizing there code ...spooky{the_g00d_ (1)
```

This is part 1, `spooky{the_g00d_`.

## 6. Assemble

| Part | Source                             | Value              |
| ---- | ---------------------------------- | ------------------ |
| 1    | `git show f7c00b3:retrospective.c` | `spooky{the_g00d_` |
| 2    | `.fog61.fog` (hex)                 | `the_l0st_@nd`     |
| 3    | `.git/errored_space.txt`           | `th3_void}`        |

Part 1 already ends in an underscore. The files don't give a separator between parts 2 and 3. I joined them with `_`, which reads as "the good, the lost and the void" and was confirmed correct.
