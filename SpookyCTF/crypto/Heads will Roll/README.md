# [Heads will Roll](https://spooky.ctfd.io/challenges#Heads%20will%20Roll-7)

While chasing Ichabod Crane through Sleepy Hollow, the Headless Horseman got as close as he could, then threw his head! While rolling, rolling, rolling, rolling, through the air, his pumpkin head shouted…what? His perfect aim knocked Ichabod to the ground near the rails, but what did he say?

1. Look at the ciphertext and the prompt. The flag format `spooky{}` is visible in scrambled form, so the characters are intact and only reordered. The prompt mentions "rails", which points to a rail fence cipher.

2. Brute-force the rail count and starting offset, and keep the result that contains `spooky`:

```python
ct="Etu_taip5_w5n_hpmnw_hlgsdso{lyh0_d}jyi_piihef_ne_oy3p0lh4oskt_i:k3l3"
n=len(ct)
def dec(ct,k,off=0):
    pat=[];r,d=0,1
    for i in range(n+off):
        pat.append(r)
        if r==0:d=1
        elif r==k-1:d=-1
        r+=d
    pat=pat[off:]
    idx=sorted(range(n),key=lambda i:(pat[i],i))
    out=['']*n
    for c,i in zip(ct,idx): out[i]=c
    return ''.join(out)
for k in range(2,12):
    for off in range(2*(k-1)):
        p=dec(ct,k,off)
        if 'spooky' in p: print(k,off,p)
```

3. The hit is 4 rails, offset 0:

[SCREEN01]
