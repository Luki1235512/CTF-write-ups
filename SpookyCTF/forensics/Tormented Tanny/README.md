# [Tormented Tanny](https://spooky.ctfd.io/challenges#Tormented%20Tanny-17)

An image was captured of a vicious beast while guarding his castle. The cameraman may have found just what makes this beast so furious. Unfortunately, the cameraman seems to have gone missing. Maybe the photo contains the answer.

1. Check the file type:

```bash
file TormentedTanny.png
```

It is a normal PNG, 1086 x 1448.

2. Dump the metadata:

```bash
exiftool TormentedTanny.png
```

Three fields hold pieces of the flag:

- `Title`: `spooky{tormented_tanner_`
- `Description` (XMP): `hates_his_`
- `Image Description` (EXIF): `vetsulin}`

3. Join the pieces in that order.

[SCREEN01]
