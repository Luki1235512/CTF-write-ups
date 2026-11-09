# [I Feel Like Someone's Watching Me](https://spooky.ctfd.io/challenges#I%20Feel%20Like%20Someone's%20Watching%20Me-31)

1. The spaces, punctuation, and capitalization are preserved, so it's a monoalphabetic substitution cipher. Word shapes give easy cribs:

- `Ypejpof Erpveaypj,` is a greeting, so "Dearest Alexander," gives Y->D, p->E, e->A, j->R, o->S, f->T, r->L, v->X, a->N.
- `D` is the one-letter word "I", and `Fcp` is "The", so D->I, F->T, c->H.
- The sign-off `Fcp Lap Zlw Rptf Ipcday` is "The One You Left Behind", which fills in more letters.

2. Extend the mapping until the whole text reads as English. The full cipher-to-plain table is:

```
p->e e->a j->r y->d o->s f->t a->n v->x r->l c->h l->o w->u z->y
t->f d->i g->c u->g h->m x->w n->k q->p b->j m->v i->b k->q
```

3. Apply it with a short script (the only cipher letter that is left over is `s`):

```python
m = {'p':'e','e':'a','j':'r','y':'d','o':'s','f':'t','a':'n','v':'x','r':'l',
    'c':'h','l':'o','w':'u','z':'y','t':'f','d':'i','i':'b','g':'c','u':'g',
    'h':'m','x':'w','n':'k','q':'p','b':'j','m':'v','k':'q'}
t = open('graveyard_letter.txt').read()
print(''.join((m[c.lower()].upper() if c.isupper() else m[c]) if c.lower() in m else
    ('['+c+']' if c.isalpha() else c) for c in t))
```

4. The final paragraph decrypts to the clue:

> eye[s] are watching me. Just remember underscores instead of spaces and to wrap it correctly.

The letter also says "I'll put a clue that should help you find me", so this sentence is the flag text, with underscores for spaces and wrapped in `spooky{}`.

5. The `[s]` is the one letter left unmapped. Cipher `s` appears only once in the whole letter. Every plaintext letter except `z` already has a cipher partner, so `s` must map to `z`. The phrase is therefore "eyez are watching me".

[SCREEN01]
