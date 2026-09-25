# Changelog

All notable changes to totp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves crypto-nv 0.1.3 to 0.1.6 and url-nv 0.1.1 to
  0.1.3.  crypto-nv 0.1.3 and url-nv 0.1.1 write into lists through
  names that are not declared `var`, which novo 0.11 refuses (E2038), so
  this package did not build with novo 0.11 against them.  No
  requirement in the manifest changed.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `totphotp` — HOTP (RFC 4226), and the decision that shapes every
  signature in the package: **a code is a `Str`, not an `Int`.**
  `012345` is a valid six-digit code and an integer holding it is
  12345, so an implementation that answers a number has to zero-pad it
  again at the edge, and the one that forgets rejects one user in ten.
  The four stages of § 5 — `counter_bytes`, `hmac_of_counter`,
  `dynamic_truncate`, `code_of_truncated` — are published separately
  rather than hidden inside `hotp`, because RFC 4226 Appendix D prints
  the intermediate HMAC and the truncated decimal for every counter,
  and a port that produces wrong digits can then be bisected against
  those columns without a debugger.
- `totptime` — TOTP (RFC 6238), and **the time as a parameter.** This
  is `core` and a `core` package may not read a clock, so an instant
  arrives as a `TotpInstant` the caller made, the same shape jwt-nv
  takes. The `[time]` is spent once, in the caller, where it is
  visible. The by-products are worth as much as the rule: a test
  verifies at any instant with no fake clock, a support tool can
  answer "was this code valid when the customer says they typed it",
  and a replay of a request log gives the same answers it gave the
  first time. `counter_at` publishes the arithmetic of § 4.2 so it can
  be driven and asserted rather than inferred from a wrong code.
- `totptime.verify` — **the verification window is a value, and
  verification answers WHICH step matched.** Both halves are one
  decision. A verify that answered only true or false could not stop a
  replay, and RFC 6238 § 5.2 requires a server to refuse a code it has
  already accepted; a code stays valid for the rest of its step, so an
  attacker who reads one off a phishing page has the remainder of that
  window. `TotpMatch` carries the counter, the caller stores it, and
  `is_replay` is the comparison — so the rule is a function rather
  than a paragraph. The store stays the caller's, because the caller
  is the one with a database.
- `totpparams` — the parameters as one checked value, and `TotpWindow`
  as **two numbers rather than one**. A clock that is behind and a
  clock that is ahead are different risks: a step backwards covers a
  user typing slowly, and a step forwards covers a fast client clock
  while widening the interval in which a captured code still works. A
  package that published one `window: Int` would have decided for
  every deployment that the two are the same size. Six digits is the
  default and 6 to 8 is the range, because that is what the Key Uri
  Format admits and what an authenticator will actually show.
- `totpalg` — the three algorithms, with SHA-1 as the default and NOT
  as a deprecation. HMAC-SHA-1 is unaffected by the SHA-1 collision
  attacks (RFC 6194, NIST SP 800-131A), and several deployed
  authenticators ignore the URI's `algorithm` parameter and compute
  SHA-1 whatever it says — so a server that enrols with SHA-256 and
  verifies with SHA-256 rejects every code those users produce. That
  is a fact about whether a program works, so it is in the README's
  rules. `block_bytes` is published because RFC 6238's own test
  vectors are stated in terms of it.
- `totpbase32` — RFC 4648 base32 and the secret as a checked value.
  Encoding writes NO padding, because a `=` in a query string has to
  be percent-encoded and several authenticators reject the URI rather
  than decoding it; decoding accepts padding, accepts its absence, and
  is CASE-INSENSITIVE, because a user copying a secret off a screen
  types lower case and a server that refused that would be refusing
  the correct secret. A bad character is refused with the character
  and its offset, which is what an operator can act on.
- `totpuri` — the `otpauth://` URI both ways, and the two places the
  format is under-specified, each with an assertion rather than a
  paragraph. The issuer is written twice, in the label and in the
  `issuer` parameter, and nothing says what to do when they disagree:
  `format` writes both and `parse` prefers the parameter, treating a
  disagreement as usable rather than locking a user out of an account
  they can otherwise reach. A label containing a colon or a space is
  the other: `format` percent-encodes both parts and joins them with a
  literal `:`, so the FIRST UNENCODED colon is the separator.
- `totperr` — eleven refusals in three classes. A secret that is not
  base32 is a CONFIGURATION mistake somebody has to correct, a code
  that is five characters is a USER event that needs no action at all,
  and a digit count of nine is a PROGRAMMING mistake that a test
  should have caught. `is_config_fault`, `is_user_fault` and
  `is_program_fault` divide them, because a deployment that logs all
  three together cannot alert on the first without paging on the
  second every day.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the four API suites reaches `not implemented:
  totp-nv.<module>.<fn>`.
- **A code that does not match is not an error.** `totptime.verify`
  answers `Ok` with `matched` false, the same way bcrypt-nv answers a
  wrong password. A wrong code is an outcome of the check and not a
  failure of it, and a caller that had to catch an error to learn "no"
  would catch the database's errors with it.
- **Every public struct is a plain struct and none is `@value`.**
  `TotpSecret`, `TotpParams`, `TotpWindow`, `TotpMatch` and `TotpUri`
  are each a `Result` payload, and `TotpInstant` is carried inside
  one; a `@value` struct may not be a `Result` payload (E2015). That
  is a documented v1 limit rather than a defect, and nothing here
  wanted an inline array: the package holds no fixed-size buffer, and
  the digest state that would want one is crypto-nv's.
- **No device claim, and no probe.** Every function speaks `Bytes`,
  `Str` or `Result`, and the embedded runtime defines none of the
  three. The README says so and points a firmware author at the
  arithmetic.
- **The dependency on crypto-nv is for two functions**, the HMAC and
  `digest.ct_eq`. Unlike sha3-nv and bcrypt-nv, which refused the
  dependency to keep SHA-2 out of a device's footprint, this package
  cannot: HOTP IS an HMAC, so the hash is not an optional extra but
  the algorithm itself.
- **Base32 lives here rather than in a package of its own.** The
  registry has none, an authenticator secret is the only thing on the
  grid written in base32, and the alphabet is eight lines. totp-rs and
  pyotp made the same call. If a base32 row is ever published,
  `totpbase32` is what it starts from.
- **No effect row wanted to widen.** Every public function is `[]`.
  The two effects this subject would otherwise reach for — `[time]`
  for the clock and `[rand]` for a generated secret — are both spent
  in the caller, and the signatures say which values they produce.
- **Named as missing, not stubbed**: randomness, QR rendering, a
  replay store, HOTP counter resynchronisation (RFC 4226 § 7.4), OCRA
  (RFC 6287), the Steam and Authy variants, and
  `otpauth-migration://` payloads.
