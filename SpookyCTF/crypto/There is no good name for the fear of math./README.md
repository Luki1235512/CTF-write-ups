# [There is no good name for the fear of math.](https://spooky.ctfd.io/challenges#There%20is%20no%20good%20name%20for%20the%20fear%20of%20math.-49)

read the participantFiles! and try you best 💚

## 1. Look at the challenge file

The only file provided is `chal.py`.

```bash
cat chal.py
```

It defines a 4-tuple structure `(a, b, c, d)` with a custom multiplication `mul`, an inverse `inv`, and an associator `assoc`. The public parameters are:

- `p = 1099511640127`, the modulus for the first component
- `q`, a 128-bit prime, the modulus for the other three components
- `mu`, a base element mod `q`, used through `mpow(e) = mu^(e mod p) mod q`
- `g`, the public generator
- `Ma` and `Mb`, the two public keys
- `ct`, the hex ciphertext

The key exchange works like this:

```python
Ma, Mb = conj(a, g), conj(b, g)          # conj(x, g) = (x^-1 * g) * x
K = shared_key(a, g, Mb)                 # assoc(a, g, Mb) * mu^(-a0) mod q
ct = flag XOR shake_256(b"loop|" + K.to_bytes(32, "big")).digest(42)
```

`mul` is not associative (that's why `assoc` exists), so this is a Diffie-Hellman variant on a loop. The secret `a` is unknown, and only `Ma = conj(a, g)` is published.

## 2. Work out what `conj` leaks

Write the secret as `a = (a0, b, c, d)` and set `G = mu^g0` and `u = mu^(-a0)`.

The first component of `mul` is plain addition mod `p`, so the first component of `(a^-1 * g) * a` is `-a0 + g0 + a0 = g0`. That matches the public data, where `Ma[0] = Mb[0] = g[0]`.

The second component of `mul(x, y)` is `b1 * mu^(-a2) + b2`, where `a2` is the first component of `y`. `inv(a)` has second component `-b * mu^a0`. Apply `mul` twice:

```
inv(a) * g      : -b * mu^a0 * mu^(-g0) + g_b
(...) * a       : (-b * mu^a0 * mu^(-g0) + g_b) * mu^(-a0) + b
                = b * (1 - G^-1) + g_b * u
```

The third component works the same way. So:

```
Ma_b = b * (1 - G^-1) + g_b * u
Ma_c = c * (1 - G^-1) + g_c * u
```

Rearranging gives Alice's secret `b` and `c` in terms of public values and the single unknown `u`:

```
b = (Ma_b - g_b * u) / (1 - G^-1)
c = (Ma_c - g_c * u) / (1 - G^-1)
```

## 3. Show that the shared key does not depend on `u`

`shared_key(a, g, Mb)` calls `assoc(a, g, Mb)`, which reads only the first three components of each argument. With `x = a`, `y = g` and `z = Mb` (where `Mb[0] = g0`), the three terms of `assoc` become:

```
t1 = mu^(a0+g0) * (mu^g0 - 1) * (b * g_c - g_b * c)    = (1/u) * G * (G - 1) * (b*g_c - g_b*c)
t2 = mu^(g0+g0) * (mu^a0 - 1) * (g_b * Mb_c - Mb_b * g_c) = G^2 * (1/u - 1) * (g_b*Mb_c - Mb_b*g_c)
t3 = mu^(g0+a0) * (mu^g0 - 1) * (Mb_b * c - b * Mb_c)  = (1/u) * G * (G - 1) * (Mb_b*c - b*Mb_c)
K  = (t1 + t2 + t3) * inv_mu * u
```

Substitute the expressions for `b` and `c` from step 2 and simplify with sympy. Create the script:

```python
import sympy as sp
u, G, Gb, Gc, Mbb, Mbc, Mab, Mac = sp.symbols('u G Gb Gc Mbb Mbc Mab Mac')
b = (Mab - Gb*u) / (1 - 1/G)
c = (Mac - Gc*u) / (1 - 1/G)
ma = 1/u
t1 = ma*G*(G - 1)*(b*Gc - Gb*c)
t2 = G*G*(ma - 1)*(Gb*Mbc - Mbb*Gc)
t3 = G*ma*(G - 1)*(Mbb*c - b*Mbc)
print(sp.factor(sp.expand(sp.simplify((t1 + t2 + t3)*u))))
```

Run it (needs `pip install sympy` if it isn't already installed):

```bash
python3 -I ~/solve/derive.py
```

Output:

```
-G**2*(Gb*Mac - Gb*Mbc - Gc*Mab + Gc*Mbb + Mab*Mbc - Mac*Mbb)
```

`u` cancels out completely. The key is built only from public values:

```
K = -G^2 * (g_b*Ma_c - g_b*Mb_c - g_c*Ma_b + g_c*Mb_b + Ma_b*Mb_c - Ma_c*Mb_b) * (mu - 1)^-1  mod q
```

No discrete log is needed. This is the whole break.

## 4. Compute the key and decrypt

Create the solver with the constants copied from `chal.py`:

```python
import hashlib
p  = 1099511640127
q  = 340282370743387819766807672197819448083
mu = 145131588818725534535896278189401302887
inv_mu = pow(mu - 1, -1, q)
g  = (739614761553, 87671554642066775345573935247301052006, 85147831812679793672994902968842668105)
Ma = (739614761553, 312134278234991743315140578529310805510, 126369916264124511285906038626254340804)
Mb = (739614761553, 244699321175961590059239024660541174646, 335110553885376877685776857080059519613)
ct = bytes.fromhex("9915497c4929fceea2ab3cfd06a744ac9ff6d8de290c8911d18b746d885fb3f5508282d2da0adff5fa15")

G = pow(mu, g[0] % p, q)
Gb, Gc = g[1], g[2]
Mab, Mac = Ma[1], Ma[2]
Mbb, Mbc = Mb[1], Mb[2]

K = (-G*G*(Gb*Mac - Gb*Mbc - Gc*Mab + Gc*Mbb + Mab*Mbc - Mac*Mbb)) * inv_mu % q
mask = hashlib.shake_256(b"loop|" + K.to_bytes(32, "big")).digest(len(ct))
print(bytes(x ^ y for x, y in zip(ct, mask)))
```

Run it:

```bash
python3 -I ~/solve/solve.py
```

Output:

```
b'spooky_scary_nonassociative_diffie_hellman'
```

## 5. Format the flag

The decrypted message has no braces. Keep the plaintext exactly as it is and put it inside the flag format.
