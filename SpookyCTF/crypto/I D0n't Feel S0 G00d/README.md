# [I D0n't Feel S0 G00d](https://spooky.ctfd.io/challenges#I%20D0n't%20Feel%20S0%20G00d-47)

You wake up in a mad scientist's lab. You feel queasy, like you took old medication. And the feeling is only getting worse. You feel a stinging sensation in your arm and look down to see "84 53 16 8 28 16 11 6 7" written on your arm. Looking around, you find two lists: one that seems to be a list of chemicals and another that is all gibberish. On top of the two pages is a note that says "Don't forget to move the neon!".

Your antidote will be formatted like spooky{antidote_here}.

### 1. Read the numbers as atomic numbers

The nine numbers are all valid atomic numbers, and the chemicals list contains an out-of-order "Periodic Acid" that hints at the periodic table. This one-liner holds the element symbols for atomic numbers 1 to 84 and prints the symbol for each number:

```python
els="H He Li Be B C N O F Ne Na Mg Al Si P S Cl Ar K Ca Sc Ti V Cr Mn Fe Co Ni Cu Zn Ga Ge As Se Br Kr Rb Sr Y Zr Nb Mo Tc Ru Rh Pd Ag Cd In Sn Sb Te I Xe Cs Ba La Ce Pr Nd Pm Sm Eu Gd Tb Dy Ho Er Tm Yb Lu Hf Ta W Re Os Ir Pt Au Hg Tl Pb Bi Po".split()
print("".join(els[int(n)-1] for n in "84 53 16 8 28 16 11 6 7".split()))'
```

Output:

```
PoISONiSNaCN
```

Split it as Po-I-S-O-N-I-S-Na-C-N and it reads "POISON IS NaCN". The poison is sodium cyanide.

### 2. Find NaCN in the chemicals list

```bash
grep -n "Cyanide" chemicals.txt
```

```
11:Calcium Cyanide
44:Potassium Cyanide
52:Sodium Cyanide
```

Sodium Cyanide is line 52. The other two cyanides are decoys.

### 3. Decode the antidotes file

The note says "Don't forget to move the neon!". Neon is element 10, so the cipher is a Caesar shift of 10. It only touches letters, and the digits are leetspeak that stays as is.

To undo the shift, each ciphertext letter moves back 10 places, so `a` becomes `q`, `b` becomes `r`, and so on. In `tr` that mapping is `a-z` to `q-za-p`.

The file's name pairs it with the chemicals list, so the antidote for the chemical on line 52 should be on line 52 of the antidotes file. Raw line 52 first:

```bash
sed -n 52p antidotes.txt
```

```
z0150x_15_l46_p0b_i0e
```

Then decode it:

```bash
sed -n 52p antidotes.txt | tr 'a-z' 'q-za-p'
```

```
p0150n_15_b46_f0r_y0u
```
