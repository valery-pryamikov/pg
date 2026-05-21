# pg — deterministic password generator

A single-page, self-contained password generator that lives in one HTML file.
You remember one master password; this page produces a strong site-specific
password for every domain you visit. Nothing is stored, nothing is synced,
nothing leaves your browser.

Open `pg.html` directly from disk, or host it on any static-file server.
No build step, no dependencies, no network calls.

## Why deterministic?

Conventional password managers store an encrypted vault and sync it between
your devices. That is a lot of moving parts: an account, a server, a sync
protocol, a recovery procedure for when you lose the device.

A deterministic generator skips all of that. Given the same inputs — your
master password, the site's domain, the username you use there, and an
optional diversificator — it always produces the same output password. You
do not need to back anything up. To get the password again, you type the
inputs again.

The trade-offs:

- **Pro:** no vault, no sync, no server, no recovery dance. The HTML file
  itself is public; you can publish it.
- **Con:** if you ever need to change a site password (forced rotation,
  breach), you have to change one of the inputs — typically by bumping the
  diversificator. The default for empty diversificator is "first password
  for this site"; later passwords for the same site become "v2", "work",
  "2024-rotation", or whatever scheme you pick.
- **Con:** there is no way to import an existing password into this scheme.
  The output is determined entirely by the algorithm. You either let the
  generator pick passwords from the start, or you do not use it.

## Usage

1. Download `pg.html`.
2. Open it in any modern browser (Firefox, Chrome, Safari, Edge).
3. Fill in:
   - **Master** — your secret. Same one every time. Pick something long.
   - **Domain** — e.g. `example.com`. A label for the site.
   - **User** — e.g. your email or username at that site.
   - **Diversificator** — optional. Leave blank for the first password;
     change it ("v2", "2024", whatever) when you need to rotate.
4. Press Enter (or click *Generate*). The password is copied to your
   clipboard. Click the eye icon to reveal it if you need to read it.

The cog icon opens character-set settings (how many capitals, lowercase,
digits, and special characters the password should contain). The question
mark icon opens an in-page help dialog with the same explanation as below.

Pressing Enter in each field validates non-empty and advances to the next;
pressing Enter in the diversificator field submits the form.

## What is safe to share or store

Only the **master password** is secret. Everything else can be written down,
synced through any service, or published, with no loss of security:

- `pg.html` itself — anyone running it gets their own passwords from their
  own master, not yours.
- Your domain / user / diversificator values — they function as per-site
  labels, not as secrets.
- Your character-set settings — they shape the output but do not weaken it.
- The generated passwords themselves — knowing the password for one site
  does not help anyone derive the password for another site, nor recover
  the master password without brute-forcing it.

An attacker who knows everything except your master still has to guess your
master by brute force. The pipeline is intentionally heavy (each candidate
costs hundreds of thousands of small block operations), so each guess costs
the attacker roughly the same work it costs you to generate one password.
**Pick a long master password** — a diceware phrase of five or more words is
a good baseline.

## Algorithm

Given inputs *master*, *domain*, *user*, and optional *div*:

1. `A = MD5(master)` — a 128-bit value, used as the AES-128 key.
2. `B = MD5(domain) ‖ MD5(user) [‖ MD5(div) if non-empty]` — 32 or 48 bytes.
3. Encrypt `B` with **AES-128-CBC** under key `A`, zero IV.
4. `C` = the last ciphertext block. Because of CBC chaining, `C` depends on
   every byte of every input.
5. Initialize a **TinyPRP** cipher (a small AES-style permutation on 16-bit
   values) with key `C`. Combined with cycle walking, TinyPRP gives a keyed
   pseudo-random permutation on any range `[0, n)`.
6. For each character class (capitals, lowercase, digits, specials) whose
   configured count is > 0:
   - Shuffle the whole charset with TinyPRP → `reSeed`.
   - Append the first *count* characters of `reSeed` to the working
     password `D`.
   - Update `C := C ⊕ MD5(reSeed)` and re-key TinyPRP. This means the
     next section (and the final shuffle below) uses a fresh permutation.
     Knowing the permutation used for one section reveals nothing about
     the others.
7. Shuffle `D` one final time with the most recent TinyPRP key. The result
   is the password.

The output has the exact composition you configured (e.g. 3 capitals,
6 lowercase, 3 digits, 0 specials → a 12-character password with exactly
that breakdown), with the characters in a deterministic but unpredictable
order.

## Cryptographic primitives

All four primitives are implemented in pure JavaScript with no dependencies,
inlined into `pg.html`:

- **MD5** (RFC 1321) — used as a fast deterministic compression function.
- **AES-128** (FIPS 197) — single-block primitive, used in CBC mode.
- **TinyPRP** — 16-bit AES-style keyed permutation. JS port of a C++
  reference implementation; structure published in *Progress in Cryptology
  — INDOCRYPT 2006*, LNCS vol. 4329, p. 251.
- **HexUtils** — hex ⇄ bytes conversion and byte-wise XOR.

The primitives are tested against published reference vectors (RFC 1321,
FIPS 197) and against Node's `crypto` module for hundreds of randomized
inputs. AES-128 and MD5 implementations are byte-for-byte compatible with
`node:crypto`.

### Why MD5 is fine here
**The design uses MD5 as a fast deterministic compression function, not as
a security primitive.**

In more details: MD5 is broken for *collision resistance*: an attacker can
construct two different inputs with the same MD5 hash. That attack model
does not apply here, because:

- The generator never compares two MD5 values nor accepts attacker-chosen
  inputs to be hashed against each other. Collision attacks require the
  attacker to choose both messages.
- MD5 is used purely as a deterministic compression function — to turn
  variable-length user inputs into fixed-length 128-bit blocks feeding AES
  and TinyPRP. The cryptographic strength of the password comes from AES
  and TinyPRP, not MD5.
- To recover the master from observed passwords, an attacker faces a
  preimage problem against the AES + TinyPRP stack. MD5's preimage and
  second-preimage resistance, while academically reduced from 128 bits,
  remains computationally infeasible (no practical attack is known).

Replacing MD5 with SHA-256 throughout the pipeline would make no practical
difference to the attacker's task.

### Caveats

- **Master entropy is the bottleneck.** No KDF saves a weak master. Use a
  passphrase.
- **No memory-hard KDF.** The heavy cost comes from many small block
  operations, not from memory pressure. An attacker with GPUs gets the
  usual ~100× speedup over a single CPU core. If your threat model includes
  adversaries spending serious money on hardware specifically for you, use
  a master password long enough to remain infeasible even at that scale
  (say, 7+ diceware words).
- **No phishing protection.** This is just a password generator. It cannot
  tell you that the site you are on is the right site. Don't paste your
  password into the wrong domain.

## Verification

There's no build step, but the cryptographic primitives and the end-to-end
pipeline have been verified. If you fork this and change anything in the
crypto, you'll want to re-run equivalent checks. The approach:

**Primitive-level vector testing.** Each of MD5, AES-128, and TinyPRP is
checked against published reference vectors:

- *MD5*: the seven canonical RFC 1321 §A.5 test vectors (`""`, `"a"`,
  `"abc"`, `"message digest"`, the lowercase alphabet, the mixed
  alphanumeric string, and the 80-digit string).
- *AES-128*: the FIPS 197 Appendix B worked example
  (`2b7e151628aed2a6abf7158809cf4f3c` encrypting
  `3243f6a8885a308d313198a2e0370734` → `3925841d02dc09fbdc118597196a0b32`)
  and the Appendix C.1 example.
- *TinyPRP*: round-trip and permutation properties — for each supported
  key size (4/8/12/16 bytes), encrypt every value in `[0, 65536)` and
  verify that (a) `decrypt(encrypt(x)) === x` for all `x` and (b) no two
  distinct inputs produce the same ciphertext (i.e. it is genuinely a
  permutation, as a keyed pseudo-random *permutation* must be).

**Differential testing against Node's `crypto` module.** For MD5 and
AES-128, 200 random inputs at varying lengths are hashed/encrypted under
both this code and `node:crypto`; outputs are compared byte-for-byte. Both
implementations are confirmed bit-compatible with Node's well-tested
native crypto.

**End-to-end pipeline.** A reference implementation of the full password
generation pipeline (steps 1–7 in [Algorithm](#algorithm) above) is run
for fixed inputs in Node, producing a known reference password
(e.g. for `master="hunter2", domain="example.com", user="alice", div=""`
with the default settings, the result is `"t2p3xewFXbyT"`). The same
inputs are then fed to the in-page algorithm via a headless DOM (jsdom),
which loads `pg.html` and runs the actual script in the page; the
in-page result must match the Node reference byte-for-byte.

**Property checks on the pipeline.** Additional tests confirm:

- Determinism: same inputs always produce the same output.
- Composition: a request for *n* capitals + *m* lowercase + ... produces
  a password with exactly that breakdown.
- Sensitivity: changing any one of master / domain / user / diversificator
  changes the output (each is mixed in before the final key is derived).
- Per-section freshness: after step 6's re-keying, shuffling the same
  charset twice with the successive keys produces structurally different
  permutations — so leaking one section's permutation reveals nothing
  about the others.

To reproduce: run the four crypto modules through Node with the equivalent
vector tests, then drive `pg.html` through jsdom and check the in-page
output. The full test scripts are not part of the published repository
(they would not run in a browser), but the approach is enough to verify
any modification.

## File layout

```
pg.html       # the whole thing — open in a browser
README.md     # this file
LICENSE       # license text
```

That is the complete repository. The four pure-JS crypto modules
(`hexUtils.js`, `md5.js`, `aes128.js`, `tinyprp.js`) are inlined into
`pg.html`. If you would prefer them as separate files, split them out of
the `<script>` block — they are clearly delimited by comments.

## License

Released into the public domain under [The Unlicense](https://unlicense.org).
See `LICENSE` for the full text. You may copy, modify, publish, use, compile,
sell, or distribute this work — in source or binary form, for any purpose,
commercial or non-commercial — with no obligations.
