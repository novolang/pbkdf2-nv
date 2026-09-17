# Changelog

All notable changes to pbkdf2-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `pbkdf2params` — the load-bearing interface, and the reason this
  package is not a four-argument function. Three of PBKDF2's five
  arguments are security decisions, and a library that supplies
  defaults for them supplies decisions nobody revisits: the `1000` that
  was defensible in 2000 is still in production because nothing ever
  asked about it again. `Pbkdf2Params` is that decision as a VALUE a
  service builds once and names, which is jwt-nv's `JwtPolicy` idea
  applied to a work factor.
- **The iteration FLOOR, and the named constructor that steps over
  it.** A default can be overridden by accident; a refusal cannot.
  `params` refuses below the floor, and
  `allowing_below_floor` is the only way past — a named constructor
  rather than a flag, so that a search finds every place a program
  decided to go below it, and a property of the VALUE rather than of
  one call. The floors are OWASP's published figures per pseudorandom
  function, published as constants so a deployment can compare what
  its toolchain believes against what it has stored.
- **Verifying does not meet the floor.** `pbkdf2phc.verify` derives
  with the parameters the stored string carries, because a deployment
  that could not read what it is holding could never migrate off it.
  `needs_rehash` is the separate question, asked at the one moment the
  password is in hand.
- `pbkdf2kdf` — the derivation, with the output buffer the caller's.
  `block` and `block_seed` are published so that a port can be
  bisected: the four-byte big-endian block index is the single most
  common place a hand-written PBKDF2 goes wrong, and it does so three
  hundred thousand HMACs after the mistake. A test that asserts those
  bytes fails AT it.
- `pbkdf2prf` — the three HMACs crypto-nv publishes, with SHA-1 kept
  and not deprecated, because WPA2, most deployed PKCS #8 and RFC
  6070's own vectors all specify it and its collision weakness does not
  apply inside an HMAC. `recommended` does not choose it.
- `pbkdf2phc` — the PHC string both ways. Its B64 is RFC 4648's
  alphabet with the padding FORBIDDEN rather than optional, so that one
  stored hash has exactly one spelling; a trailing `=` is a refusal.
- `pbkdf2err` — eight refusals, with `is_policy_fault` separating the
  one this package invented from the ones RFC 8018 requires, and
  `is_input_fault` and `is_format_fault` separating a bug in the
  calling code from a bad row in the database.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the three API suites reaches `not implemented:
  pbkdf2-nv.<module>.<fn>`.
- **The salt is a parameter, and that is what makes this `core`.**
  Generating one costs `[rand]`, which this layer's budget does not
  admit. No effect row in this package wanted to widen.
- **`Pbkdf2Params` and `Pbkdf2Phc` are plain structs and not `@value`
  ones.** Both are `Result` payloads, and a `@value` struct cannot be
  one (E2015).
- **No device claim, and no probe.** Every function here speaks `Bytes`
  and `Result`, which the embedded runtime does not define. The
  arithmetic underneath is crypto-nv's, whose `*_core` modules do carry
  the claim; a device that needs PBKDF2 drives those directly.
- **PBKDF1, PBES2, PBMAC1, HMAC-SHA-224, HMAC-SHA-384, Argon2 and
  scrypt are named as missing**, not stubbed. The README says which of
  bcrypt-nv and this package to take, and why.
