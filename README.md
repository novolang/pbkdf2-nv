# pbkdf2-nv

PBKDF2 is a password-based key derivation function, specified in
[RFC 8018](https://www.rfc-editor.org/rfc/rfc8018), which is PKCS #5
version 2.1. It turns a password into a key of whatever length is
asked for, and it is deliberately slow. This package brings it to
novo-lang, over the HMAC in
[crypto-nv](https://novo-lang.org/packages/crypto-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What PBKDF2 is

A **key derivation function** turns something a person can remember
into something a cipher can use. The problem it solves is that a
password has little entropy and a key needs a lot, so an attacker who
has the derived key and can guess passwords will simply try them. The
answer is to make each try expensive.

PBKDF2 does that by repetition. A **pseudorandom function** — here a
**message authentication code (MAC)**, which is a hash under a secret
key — is run over the password many times in a chain, and the number of
times is the **iteration count**. RFC 8018 § 5.2 calls it `c`. A
**salt** is extra input that makes two users with the same password
derive different keys, so one precomputed table cannot attack them
both.

The derived key is built a block at a time, one block per `hLen` bytes
asked for, where `hLen` is the length of the pseudorandom function's
output. Block `i` is

```
F(P, S, c, i) = U_1 XOR U_2 XOR ... XOR U_c
```

where `U_1` is the function of the password over the salt followed by
`i` as four big-endian bytes, and each later `U` is the function of the
password over the one before it. The blocks are joined and the result
is cut to the length asked for.

Two properties follow from that shape, and both matter to a caller.
The chain inside one block is **sequential**, so a block cannot be
computed in parallel — which is the point. Different blocks are
**independent**, so a longer key costs a whole extra chain per block
and gives an attacker no extra work if only the first block matters.

| Pseudorandom function | Block | HMAC block size | PHC identifier |
| --- | --- | --- | --- |
| HMAC-SHA-1 | 20 bytes | 64 bytes | `pbkdf2-sha1` |
| HMAC-SHA-256 | 32 bytes | 64 bytes | `pbkdf2-sha256` |
| HMAC-SHA-512 | 64 bytes | 128 bytes | `pbkdf2-sha512` |

PBKDF2 is **not memory-hard**. It is a chain of MACs over a few hundred
bytes of state, which is what a graphics processor does well. That is
not a flaw in an implementation; it is the design, standardised in 2000
before memory-hardness was a technique. PBKDF2 is the right function
when a specification names it — WPA2's pre-shared key, PKCS #8, PKCS
#12, a hundred file formats — and the wrong one to choose for a new
password store.

A stored PBKDF2 hash is usually written in the **PHC string format**,
which came out of the Password Hashing Competition and carries the
parameters beside the key:

```
$pbkdf2-sha256$i=600000$c2FsdHNhbHQ$aGFzaGhhc2hoYXNoaGFzaA
```

## Install

```
novo pkg add pbkdf2-nv
```

## Example

```novo
use std.bytes
use pbkdf2kdf
use pbkdf2params
use pbkdf2phc

fn main() [io]
    // What a derivation costs, as one value the service names once.
    let work = pbkdf2params.recommended(Pbkdf2HmacSha256)

    // Sixteen bytes of salt. Your program reads them from a random
    // source; this package never does.
    let salt = bytes.zeros(16)

    // A key, into a buffer this call allocates.
    match pbkdf2kdf.derive(work, bytes.from_str("password"), salt)
        Err(e) => println(e.message())
        Ok(key) => println("${bytes.len(key)} bytes of key")

    // Or the string to store in a password column.
    match pbkdf2phc.hash_text(work, bytes.from_str("password"), salt)
        Err(e) => println(e.message())
        Ok(stored) => println(stored)

    // Checking one. The count and the salt come out of the stored
    // string, so a hash written at an older count still verifies.
    match pbkdf2phc.verify("\$pbkdf2-sha256\$i=600000\$c2FsdHNhbHQ\$aGFzaA",
                           bytes.from_str("password"))
        Ok(true)  => println("that is the password")
        Ok(false) => println("that is not the password")
        Err(e)    => println("that row is not a PBKDF2 hash: ${e.message()}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented:
pbkdf2-nv.<module>.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `pbkdf2prf` | The three pseudorandom functions, their block lengths, their PHC identifiers and the derived-key cap each implies. |
| `pbkdf2params` | What a derivation costs and how long its answer is, as one value: the constructors, the iteration floor, and the check a service runs at start-up. |
| `pbkdf2kdf` | The derivation itself: into a buffer you own, into one this package allocates, one block at a time, and the constant-time comparison. |
| `pbkdf2phc` | The PHC string both ways, the B64 the fields are written in, hashing to a string and verifying against one. |
| `pbkdf2err` | The eight refusals, and whether each is this package's policy, a fault in the call, or a fault in the stored data. |

## How to choose an entry point

**`pbkdf2params.recommended` is where to start.** It answers the
parameters a new system should use for a pseudorandom function: that
function's floor and a 32-byte key. Raise the count with
`with_iterations` once you have measured your own endpoint.

**`pbkdf2phc.hash_text` and `pbkdf2phc.verify` are for a password
column.** They write and read the string that carries the parameters,
so a hash written at an older count still verifies.

**`pbkdf2kdf.derive_into` is for a key**, not a password hash —
unwrapping a PKCS #8 file, deriving a WPA2 pre-shared key, turning a
passphrase into a cipher key. It writes into a buffer you already own.

**`pbkdf2kdf.block` is for a caller that needs two keys from one
password.** Drive the blocks yourself rather than deriving a long key
and slicing it, so that each key's provenance is visible.

## The rules a user needs

1. **The salt is a parameter.** This package reads no randomness. Your
   program generates it, or takes it out of the stored hash it is
   verifying against. The one line of your program that reads a random
   source is therefore visible.
2. **The iteration count is refused below a floor.** The floor is this
   package's policy, not RFC 8018's: 1,300,000 for HMAC-SHA-1, 600,000
   for HMAC-SHA-256 and 210,000 for HMAC-SHA-512, which are OWASP's
   published figures. They differ because one iteration costs different
   amounts, and the intent is that a guess costs the same whichever is
   chosen.
3. **The only way below the floor is
   `pbkdf2params.allowing_below_floor`.** It is a named constructor and
   not a flag, so that a search finds every place a program decided to
   go below it. The use it exists for is real: verifying a hash written
   a decade ago, which is the one moment the password is in hand and
   the hash can be replaced.
4. **Verifying a stored hash does not meet the floor.** `verify` uses
   the parameters the string carries. A deployment that could not read
   what it is holding could never migrate off it.
   `pbkdf2phc.needs_rehash` is the separate question.
5. **A longer key costs a whole extra chain per block.** A 64-byte key
   out of HMAC-SHA-256 costs exactly twice a 32-byte one, and an
   attacker who only needs the first block checks it at half what you
   paid. `pbkdf2params.block_count` is that fact as a number.
6. **The block index is four bytes, big-endian.** RFC 8018 § 5.2's
   `INT(i)`, and the single most common place a hand-written PBKDF2
   goes wrong: a little-endian index gives a self-consistent key that
   matches nothing. `pbkdf2kdf.block_seed` is published so a test can
   assert those bytes.
7. **Blocks are numbered from one.** There is no block 0.
8. **A zero byte in the password or the salt is an ordinary byte.**
   Unlike bcrypt, PBKDF2 has no NUL-terminated string anywhere in it.
   RFC 6070's sixth vector exists to catch an implementation that went
   through a C string.
9. **A password longer than the HMAC block size gains nothing.** HMAC
   hashes a key longer than its block size first, so two long passwords
   with the same hash derive the same key. That is 64 bytes for
   HMAC-SHA-1 and HMAC-SHA-256, 128 for HMAC-SHA-512.
10. **Compare derived keys with `pbkdf2kdf.keys_equal`, never with
    `==`.** It is crypto-nv's `digest.ct_eq` under a name that says what
    it is for.
11. **A wrong password is `Ok(false)`; a row that is not a PBKDF2 hash
    is `Err`.** A login endpoint has to tell them apart, or it reports a
    database fault as a bad password.
12. **PHC's B64 forbids padding.** The alphabet is RFC 4648's, but a
    trailing `=` is a malformed string rather than a tolerated one, so
    that one stored hash has exactly one spelling.
13. **`pbkdf2params.check` belongs at start-up.** Left to run time, an
    unusable set of parameters arrives as a stream of failed logins that
    look like users getting their passwords wrong.
14. **SHA-1 is here and is not deprecated.** PBKDF2-HMAC-SHA-1 is what
    WPA2, most deployed PKCS #8 and RFC 6070's own vectors specify.
    SHA-1's collision weakness does not apply inside an HMAC. What a new
    system should not do is choose it, and `recommended` does not.

## Running on a microcontroller

This package makes no device claim and ships no probe.

Every function here speaks `Bytes` and `Result`, which are heap objects
the embedded runtime does not define, and one host-only function
anywhere in a compilation unit is an undefined symbol at link time on a
device. The arithmetic underneath is crypto-nv's, whose `sha256_core`
and its siblings *do* carry the claim. A device that needs PBKDF2
drives those directly.

## Timing behaviour

- **The derivation's time depends on the iteration count, the derived
  key length and the password length, and on nothing secret beyond
  those.** The password's length is not its content; no branch depends
  on a password bit.
- **`pbkdf2kdf.keys_equal` is constant-time.** It reads both buffers
  whole and folds every difference into one accumulator, so the time it
  takes says nothing about how many bytes matched. `==` stops at the
  first differing byte, and how long that took says how many bytes were
  right.
- **HMAC's own key-dependent work is the same table-free arithmetic on
  every input**, which is crypto-nv's claim and not a further one made
  here.
- **`pbkdf2phc.parse` is not constant-time** and does not need to be: a
  stored hash is not a secret.

## What is not included

- **Randomness.** Generating a salt costs `[rand]`, and this package
  declares no effects. See rule 1.
- **PBKDF1.** RFC 8018 § 5.1 keeps it for compatibility with PKCS #5
  version 1.5 and says it should not be used for new applications. It
  cannot produce a key longer than the hash.
- **PBES2 and PBMAC1.** RFC 8018 also defines an encryption scheme and
  a MAC scheme over PBKDF2. Both need a cipher, which is a different
  package; the key they would use is what `pbkdf2kdf.derive` answers.
- **HMAC-SHA-224 and HMAC-SHA-384.** crypto-nv publishes neither, and
  no format on this grid specifies either. A caller that meets one is
  told `Pbkdf2UnknownPrf` with the identifier it read.
- **Argon2 and scrypt.** Both are memory-hard and both are better for a
  new password store. Neither is on the registry yet.
- **A password policy.** Length rules, dictionary checks and breach
  lists are a different subject.
- **Any input or output.** A password arrives as bytes the caller read
  and a key leaves as bytes the caller uses.

## Related packages

- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies the
  three HMACs this package is defined over, and `digest.ct_eq`. This
  package depends on it.
- [bcrypt-nv](https://novo-lang.org/packages/bcrypt-nv) is bcrypt.
  Take it when you are storing a password and nothing has specified
  otherwise: its cost is a single number and its key schedule is harder
  to accelerate. Take this package when a specification names PBKDF2,
  or when the result has to be a key of a length you choose.
- [hkdf-nv](https://novo-lang.org/packages/hkdf-nv) is HKDF, which is
  also a chain of HMACs and is a different job: it expands a key that
  is already strong, cheaply. It is not a password hash and has no
  iteration count.
- [base64-nv](https://novo-lang.org/packages/base64-nv) is RFC 4648
  base64. PHC's B64 forbids padding, so this package carries its own
  encoder; see rule 12.

## Test vectors

RFC 6070 is the canonical set for PBKDF2-HMAC-SHA-1 and prints six
cases, including one whose password and salt both contain a zero byte.
The HMAC-SHA-256 cases are the ones published alongside the scrypt
draft and carried by RustCrypto's `pbkdf2` crate and CPython's
`hashlib` test suite.

```bash
novo test tests/pbkdf2_vectors_tests.nv   # RFC 6070, and the SHA-256 set
novo test tests/pbkdf2_params_tests.nv    # the floor, the override, the cost arithmetic
novo test tests/pbkdf2_phc_tests.nv       # the stored string, both ways
```

Every published vector is at an iteration count far below this
package's floor, so every one of them goes through
`allowing_below_floor`. That is the floor working: a vector from 2011
is exactly the case the override exists for.

The suite asserts RFC 6070's cases 1, 2, 3, 5 and 6, the three
HMAC-SHA-256 cases, that the block seed is the salt followed by four
big-endian bytes, that block 1 of a one-block derivation is the whole
key, that blocks are numbered from one, that a short output buffer is
refused rather than partly written, that a count below the floor is
refused and the override accepts it, that changing the pseudorandom
function does not rescale the count, that a 64-byte key is exactly
twice the work of a 32-byte one, that PHC's B64 refuses a trailing `=`,
that a stored string round-trips, and that a legacy hash verifies
without the override while writing one is refused.

The tests compile today and fail at run, each on the `not implemented:
pbkdf2-nv.<module>.<fn>` panic that is its body. That is the expected
state of an interface release. They turn green one at a time as bodies
land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The three floors, `PBKDF2_DEFAULT_DK_BYTES`, `PBKDF2_MIN_SALT_BYTES`, `PBKDF2_BLOCK_INDEX_BYTES`, `PHC_FIELD_SEPARATOR`, `PHC_ITERATIONS_KEY` | yes (they are constants) |
| `pbkdf2prf.Pbkdf2Prf`, `pbkdf2params.Pbkdf2Params`, `pbkdf2phc.Pbkdf2Phc`, `pbkdf2err.Pbkdf2Error` | the types are declared |
| `pbkdf2prf.block_bytes`, `.hmac_block_bytes`, `.prf_name`, `.prf_named`, `.max_derived_bytes` | no |
| `pbkdf2params.recommended`, `.params`, `.allowing_below_floor` | no |
| `pbkdf2params.with_iterations`, `.with_dk_len`, `.with_prf` | no |
| `pbkdf2params.prf_of`, `.iterations_of`, `.dk_len_of`, `.is_below_floor`, `.floor_for` | no |
| `pbkdf2params.block_count`, `.total_hmac_calls`, `.check` | no |
| `pbkdf2kdf.derive_into`, `.derive`, `.block`, `.block_seed`, `.keys_equal` | no |
| `pbkdf2phc.b64_encode`, `.b64_decode`, `.b64_chars_for` | no |
| `pbkdf2phc.parse`, `.render`, `.phc_value` | no |
| `pbkdf2phc.hash_text`, `.verify`, `.needs_rehash` | no |
| `pbkdf2phc.params_of`, `.salt_of`, `.key_of` | no |
| `pbkdf2err.is_policy_fault`, `.is_input_fault`, `.is_format_fault`, `.format_offset`, `.code`, `Pbkdf2Error.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
