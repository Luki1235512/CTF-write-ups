# [impossible](https://nnsc.tf/challenges?challenge=crypto_impossible)

**Description:**

In the ashes of a ceremony held long ago, you found a secret that was not hidden nearly well enough

Use it to convince the Groth16 verifier that you are authorized to approve a huge mint!

## 1. Extract and read the provided files

```bash
tar xzvf crypto_impossible_tar.gz
```

This gives a small Rust project:

```
crypto_impossible/
  Cargo.toml
  src/lib.rs
  src/bin/solve.rs
  secret
  vk.bin
```

`vk.bin` is the Groth16 verifying key the remote server checks proofs against. `secret` is a 2-line text file. `src/lib.rs` contains everything you need to understand the proof format: the circuit definition, how the ceremony's secret parameters are derived, encoding/decoding helpers, and the `verify()` function itself.

Read `lib.rs` first, in full, before touching anything else. The important parts:

- `MintCircuit`: the two constraints described above.
- `derive(tau)`: takes a single field element `tau` and deterministically derives six more field elements by hashing `tau` with BLAKE2b under different domain labels.
- `mint_ic_scalars(tau, alpha, beta, gamma)`: computes the two "IC" scalars that go into the verifying key, using Lagrange interpolation over 4th roots of unity. This is how the circuit's fixed constraints get baked into the verifying key.
- `verify(vk, proof)`: standard Groth16 verification, plus a check that `proof.ceremony_id` matches and `proof.claim == CLAIM`.

The names `derive` and `mint_ic_scalars` matter a lot: they tell you the entire verifying key can be reconstructed from one scalar, `tau`. That's the toxic waste of a Groth16 trusted setup ceremony. If `tau` ever leaks, the ceremony is compromised.

## 2. Decode the secret file

```bash
cat secret
PRERZBAL_VQ=3p311q9qso7735r42643s394qp2p10ns
GNH=3894627051107121998319229043008213446770981528672674568925122813412699817
```

The letters are ROT13'd. Decoding letter by letter:

```
PRERZBAL_VQ -> CEREMONY_ID
GNH         -> TAU
```

So the file actually says:

```
CEREMONY_ID=3c311d9dfb7735e42643f394dc2c10af
TAU=3894627051107121998319229043008213446770981528672674568925122813412699817
```

This is the whole vulnerability in one line: whoever ran the trusted setup ceremony didn't destroy `tau`, they just rotated the letters of the label and left the value sitting in a file called `secret`. "Not hidden nearly well enough," as the challenge description says.

## 3. Understand why knowing these scalars breaks the proof system

Groth16 verification checks a single pairing equation:

```
e(A, B) = e(alpha, beta) * e(vk_x, gamma) * e(C, delta)
```

where `A, B, C` are the three proof elements and `vk_x = IC[0] + input * IC[1]`.

Normally `A`, `B`, `C` are computed from a real witness that satisfies the circuit, and you can't reverse-engineer valid ones without solving discrete log. But here we know `alpha`, `beta`, `gamma`, `delta` as actual numbers, not just as opaque curve points. That means we can treat the whole equation as ordinary arithmetic over the scalar field instead of a discrete-log-hard pairing check.

Pick `A` and `B` however we like, for example the standard generators themselves. Then solve for the one remaining unknown, `C`, directly:

```
C = (A*B - alpha*beta - vk_x*gamma) / delta
```

Every term on the right is a scalar we already know: `alpha`, `beta`, `gamma`, `delta` from `derive(tau)`, and `vk_x` from `mint_ic_scalars(tau, ...)` combined with the claimed amount, 1,000,000,000. Solve, then turn `C` back into a curve point.

This produces a proof that satisfies the pairing equation for the false statement "1,000,000,000 <= 100" without ever running the circuit or holding a real witness. That's exactly what leaking the trusted setup's toxic waste allows.

Watch two details from `lib.rs` while doing this:

- All the scalars from `derive()` and `mint_ic_scalars()` were computed with respect to the _scaled_ generators, so when combining them into a single scalar equation, convert everything to be relative to the same generator. Multiplying each ceremony scalar by its corresponding scale factor first keeps everything consistent.
- `BellmanProof::read` rejects points at infinity, so after computing `C` as a field element, check it isn't zero before encoding it.

## 4. Build the forged proof

```rust
use impossible_player::*;
use bellman::groth16::{Proof as BellmanProof, VerifyingKey};
use pairing::bls12_381::{Bls12, Fr};
use pairing::{Field, PrimeField};
use std::fs::File;

fn rot13(s: &str) -> String {
    s.chars().map(|c| {
        if c.is_ascii_lowercase() {
            (((c as u8 - b'a' + 13) % 26) + b'a') as char
        } else if c.is_ascii_uppercase() {
            (((c as u8 - b'A' + 13) % 26) + b'A') as char
        } else {
            c
        }
    }).collect()
}

fn main() {
    let secret = std::fs::read_to_string("secret").unwrap();
    let mut ceremony_id = String::new();
    let mut tau_str = String::new();
    for line in secret.lines() {
        let decoded = rot13(line);
        if let Some(rest) = decoded.strip_prefix("CEREMONY_ID=") {
            ceremony_id = rest.trim().to_string();
        } else if let Some(rest) = decoded.strip_prefix("TAU=") {
            tau_str = rest.trim().to_string();
        }
    }
    let tau = Fr::from_str(&tau_str).unwrap();
    let [alpha, beta, gamma, delta, g1s, g2s] = derive(tau);
    let [ic0, ic1] = mint_ic_scalars(tau, alpha, beta, gamma);

    let mut alpha_g1_dlog = alpha; alpha_g1_dlog.mul_assign(&g1s);
    let mut beta_g2_dlog = beta; beta_g2_dlog.mul_assign(&g2s);
    let mut gamma_g2_dlog = gamma; gamma_g2_dlog.mul_assign(&g2s);
    let mut delta_g2_dlog = delta; delta_g2_dlog.mul_assign(&g2s);
    let mut ic0_dlog = ic0; ic0_dlog.mul_assign(&g1s);
    let mut ic1_dlog = ic1; ic1_dlog.mul_assign(&g1s);

    let claim = fr(CLAIM);
    let mut vkx_dlog = ic1_dlog;
    vkx_dlog.mul_assign(&claim);
    vkx_dlog.add_assign(&ic0_dlog);

    let a_dlog = Fr::one();
    let b_dlog = Fr::one();

    let mut rhs = alpha_g1_dlog;
    rhs.mul_assign(&beta_g2_dlog);
    let mut tmp = vkx_dlog;
    tmp.mul_assign(&gamma_g2_dlog);
    rhs.add_assign(&tmp);

    let mut ab = a_dlog;
    ab.mul_assign(&b_dlog);

    let mut numerator = ab;
    numerator.sub_assign(&rhs);

    let delta_inv = delta_g2_dlog.inverse().expect("delta_g2 must be invertible");
    let mut c_dlog = numerator;
    c_dlog.mul_assign(&delta_inv);

    assert!(!c_dlog.is_zero(), "C must not be zero");

    let inner = BellmanProof::<Bls12> {
        a: g1(a_dlog),
        b: g2(b_dlog),
        c: g1(c_dlog),
    };

    let proof = Proof {
        ceremony_id: ceremony_id.clone(),
        claim: CLAIM,
        inner,
    };

    let f = File::open("vk.bin").unwrap();
    let vk = VerifyingKey::<Bls12>::read(f).unwrap();
    let public = Public { ceremony_id: ceremony_id.clone(), vk };
    let ok = verify(&public, &proof);
    eprintln!("local verify() result = {}", ok);
    assert!(ok, "forged proof failed local verification!");

    let payload = encode_proof(&proof);
    println!("{}", payload);
}
```

### Building and running it

```
cd crypto_impossible          # the folder with Cargo.toml, secret, vk.bin
rm -f Cargo.lock              # regenerate it, the shipped one may be too new for your cargo
cargo build --bin forge       # pulls bellman/pairing/blake2-rfc from crates.io on first run
./target/debug/forge
```

Running the binary prints `local verify() result = true` on stderr and the `ceremony_id:hex` payload on stdout. That stdout line is what you paste after `submit` on the remote service.

## 5. Submit to the remote service

Connect to the challenge:

```bash
ncat --ssl <host> 1337
```

It prompts:

```
authorized balance: 100
requested mint: 1000000000
submit <ceremony_id>:<proof>
>
```

Send the string produced by `encode_proof()`, the same `ceremony_id:hex` line you already verified locally. The server runs the same `verify()` function you tested against, on the same `vk.bin`, with the same `ceremony_id` and `CLAIM`, so a proof that passed locally passes remotely. The server accepts the mint and returns the flag.

<img width="1201" height="191" alt="SCREEN01" src="https://github.com/user-attachments/assets/c283994d-5951-4ce9-a9a8-b9d65447fdf9" />
