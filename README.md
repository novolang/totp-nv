# totp-nv

A one-time password is a short number that is valid once, or for half
a minute, and is used beside a password as a second factor. Two
specifications define the ones every authenticator application
implements: [RFC 4226](https://www.rfc-editor.org/rfc/rfc4226) defines
HOTP, which counts, and [RFC 6238](https://www.rfc-editor.org/rfc/rfc6238)
defines TOTP, which is the same function with the count taken from a
clock. This package brings both to novo-lang, over
[crypto-nv](https://novo-lang.org/packages/crypto-nv)'s HMAC.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a one-time password is

A **shared secret** is a string of random bytes that a server and a
user's phone both hold. Enrolment is the act of moving it to the phone,
usually by showing a QR code. Nothing else is ever sent: the two sides
each compute a number from the secret, and the user copies the phone's
number into the server's form.

The number is computed from the secret and a **moving factor**, so that
it is different every time. HOTP's moving factor is a **counter** that
both sides increment. TOTP's is the current time divided by a **time
step**, thirty seconds by default, so the number changes on its own and
both sides get the same one without having to agree on anything but the
clock.

A **message authentication code** (MAC) is a value computed from a
message under a secret key, which only a holder of the key can produce
or check. **HMAC** is the MAC built from a hash function, specified in
[RFC 2104](https://www.rfc-editor.org/rfc/rfc2104). The whole of HOTP is
one HMAC and a truncation:

```
HOTP(K, C) = Truncate(HMAC-SHA-1(K, C)) mod 10^Digit
```

`K` is the secret, `C` is the counter as eight bytes, and `Truncate` is
the **dynamic truncation** of RFC 4226 section 5.3: the low four bits
of the last byte of the digest give an offset, and the four bytes at
that offset, with the top bit cleared, are a 31-bit number. The code is
that number modulo ten to the power of the digit count. TOTP, in RFC
6238 section 4.2, is the same thing with

```
C = (Unix time - T0) / X
```

where `X` is the time step in seconds and `T0` is the epoch the steps
are counted from, the Unix epoch in every deployment that
interoperates.

The **verification window** is how many steps either side of the
current one a server will accept. It exists because a user reads a
number and types it, and the step may end while they are typing. A
window of one step backwards is what RFC 6238 section 5.2 recommends.

The secret is written down in **base32**, the encoding of
[RFC 4648](https://www.rfc-editor.org/rfc/rfc4648) section 6:
twenty-six letters and the digits 2 to 7, leaving out the characters a
reader confuses. A secret is carried to a phone in an **`otpauth://`
URI**, which holds the base32 secret and the parameters:

```
otpauth://totp/Example:alice@example.com?secret=GEZDGNBVGY3TQOJQ&issuer=Example&algorithm=SHA1&digits=6&period=30
```

## Install

```
novo pkg add totp-nv
```

## Example

```novo
use std.bytes
use totpbase32
use totphotp
use totpparams
use totptime
use totpuri

fn main() [io]
    // The shared secret, as the server holds it.  A real one is twenty
    // random bytes; this is RFC 4226's own test secret.
    match totpbase32.secret(bytes.from_str("12345678901234567890"))
        Err(e) => println(e.message())
        Ok(s)  =>
            let p = totpparams.default_params()

            // The URI to draw as a QR code when the user enrols.
            println(totpuri.format(totpuri.uri("Example", "alice", s, p)))

            // The code at a given moment.  The caller supplies the
            // instant: this package never reads a clock.
            match totptime.totp(s, p, totptime.instant(59))
                Ok(c)  => println(c)              // 287082
                Err(e) => println(e.message())

            // Verifying what the user typed.  The answer says WHICH
            // step matched, so the caller can store it and refuse the
            // same code a second time.
            match totptime.verify(s, "287082", p, totptime.instant(89))
                Err(e) => println(e.message())
                Ok(m)  =>
                    println("${m.matched} at step ${m.counter}")
                    // true at step 1 — the previous step, inside the window

            // The counter-based form, for a hardware token.
            match totphotp.hotp(s, 0, p)
                Ok(c)  => println(c)              // 755224
                Err(e) => println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented:
totp-nv.<module>.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `totphotp` | HOTP: the counter as eight bytes, the HMAC, the dynamic truncation of RFC 4226 section 5.3, and the code as a zero-padded string. Each stage is published separately, so a wrong implementation can be bisected. |
| `totptime` | TOTP: the instant as a value the caller supplies, the counter arithmetic of RFC 6238 section 4.2, the code, and verification over a window that answers which step matched. |
| `totpparams` | The algorithm, the digit count, the time step, the epoch and the verification window, as one checked value. |
| `totpalg` | The three hashes RFC 6238 admits, each with its digest length, its HMAC block size and the name the URI writes it under. |
| `totpbase32` | RFC 4648 base32, and the shared secret as a checked value. Decoding is case-insensitive and padding is optional; encoding writes none. |
| `totpuri` | The `otpauth://` enrolment URI, parsed and formatted, including the label's percent-encoding. |
| `totperr` | The eleven refusals, divided into the three classes a deployment reacts to differently. |

## How to choose an entry point

**`totptime` is for a phone or an authenticator application.** That is
what almost every deployment means by two-factor authentication.
`totptime.totp` produces a code at an instant and `totptime.verify`
checks one over the window.

**`totphotp` is for a hardware token that counts.** A key fob with a
button has no clock, so its moving factor is a counter that both sides
keep. Use `totphotp.hotp` and `totphotp.verify_at`, and store the
counter yourself.

**`totphotp`'s individual stages are for porting and for debugging.**
`counter_bytes`, `hmac_of_counter`, `dynamic_truncate` and
`code_of_truncated` are the four stages of RFC 4226 section 5, each
published on its own. An implementation that produces wrong digits is
found by comparing them against RFC 4226 Appendix D one stage at a
time.

**`totpuri` is for enrolment and for reading an exported account.** It
answers the string a QR encoder takes, and reads one back.

## The rules a user needs

1. **A code is a string, not a number.** `012345` is a valid six-digit
   code, and an integer holding it is 12345. Every function here
   answers a `Str` of exactly `digits` characters, and a caller that
   parses one into an integer rejects one user in ten.
2. **Compare codes with `digest.ct_eq`, never with `==`.** A string
   comparison stops at the first character that differs, so the time it
   takes says how many leading characters were right. crypto-nv's
   `digest.ct_eq` compares in constant time, and it is what
   `totphotp.verify_at` and `totptime.verify` use. A caller doing its
   own comparison calls it too.
3. **Verification answers which step matched, and the caller must
   remember it.** RFC 6238 section 5.2 requires that a code accepted
   once is refused the second time. A code stays valid for the rest of
   its step, so an attacker who reads one off a phishing page can still
   use it. `totptime.verify` answers a `TotpMatch` carrying the
   counter: store it against the account and pass it to
   `totptime.is_replay` on the next attempt. This package keeps no
   state and cannot do it for you.
4. **The current time is a parameter.** Nothing here reads a clock. A
   caller writes `totptime.instant(time.unix_seconds())` and hands the
   result in, so the one place a program reads the clock is visible in
   the program.
5. **SHA-1 is the default and is the interoperable choice.** RFC 4226
   section 5.1 specifies HMAC-SHA-1, and several widely deployed
   authenticator applications ignore the URI's `algorithm` parameter
   and compute SHA-1 whatever it says. A server that enrols with
   SHA-256 and verifies with SHA-256 rejects every code those
   applications produce. HMAC-SHA-1 is not affected by the SHA-1
   collision attacks, which are about finding two messages with one
   digest; RFC 6194 and NIST SP 800-131A both continue to admit it.
6. **Six digits by default, and 6 to 8 is the whole range.** The Key
   Uri Format admits 6 and 8. Anything outside 6 to 8 is
   `TotpDigitsOutOfRange` rather than a wider code that no
   authenticator will produce.
7. **The window is two numbers, and the forward one is not free.**
   Accepting a step backwards covers a user typing slowly. Accepting a
   step forwards covers a client whose clock runs fast, and widens the
   interval in which a captured code is still worth something. RFC 6238
   section 5.2 recommends at most one step; the default here is one
   step backwards and none forwards.
8. **T0 is the Unix epoch, and cannot be anything else in practice.**
   The enrolment URI has no parameter for it, so an authenticator
   always counts from 0. A deployment that changes `t0_seconds` can
   verify only codes it produced itself.
9. **The secret is written without base32 padding.** A `=` in a query
   string has to be percent-encoded, and several authenticators reject
   the URI rather than decoding it. `totpbase32.encode` writes none.
   `totpbase32.decode` accepts padding and accepts its absence, and is
   case-insensitive, because a user reading a secret off a screen types
   lower case.
10. **In the `otpauth://` URI the issuer is written twice, and this
    package prefers the parameter.** The label may be `Issuer:account`
    and there is also an `issuer` parameter. The format recommends
    writing both and does not say what to do when they disagree.
    `totpuri.format` writes both. `totpuri.parse` takes the parameter
    when both are present and the label's prefix when there is no
    parameter, and treats a disagreement as usable rather than
    refusing it.
11. **The first unencoded colon in the label is the separator.** A
    colon inside an issuer or an account name is percent-encoded as
    `%3A` and a space as `%20`. `totpuri.parse` also accepts `+` for a
    space in the query, because some implementations write one, and
    never writes one itself.
12. **RFC 6238's Appendix B needs a seed the appendix's prose does not
    describe.** The published SHA-256 and SHA-512 values are computed
    over the ASCII seed repeated to the hash's block size — 32 bytes
    and 64 bytes — not over the twenty bytes the text names. This is
    errata 2866 and 3208. An implementation checked against the
    appendix as written reproduces the SHA-1 column and neither of the
    others.

## Running on a microcontroller

This package makes no device claim and ships no probe.

Every function here speaks `Bytes`, `Str` or `Result`, and the embedded
runtime defines none of the three. The device-shaped part of this work
is the digest state underneath, and that belongs to
[crypto-nv](https://novo-lang.org/packages/crypto-nv), whose hash state
is a value that lives in the caller's own stack frame. A firmware
author who needs a one-time password has the whole algorithm in front
of them: eight bytes, an HMAC, four bytes and a modulo.

## Timing behaviour

- **Code comparison is constant-time**, in `totphotp.verify_at` and
  `totptime.verify`, through crypto-nv's `digest.ct_eq`. See rule 2.
- **Verification takes the time of the whole window, whichever step
  matched.** Every step's code is computed and compared, and the loop
  does not stop early. A verifier that stopped at the first match would
  say through its timing which step matched, which says how far the
  client's clock is out.
- **The HMAC itself is crypto-nv's**, and its timing is that package's
  business. It depends on the message length, which here is always
  eight bytes.
- **Base32 decoding is not constant-time.** It branches per character
  and refuses at the first character outside the alphabet. A secret is
  decoded once from a value the server already holds, not per request.

## What is not included

- **Any randomness.** Generating a shared secret needs `[rand]`, which
  a `core` package's effect budget does not admit. The twenty random
  bytes are the caller's, or a `host` package's on top of this one.
  `totpbase32.secret` takes them.
- **Any clock.** See rule 4.
- **QR code rendering.** What a QR encoder takes is a string, and
  `totpuri.format` answers it.
- **A replay store.** This package answers which step matched. Where
  the last accepted step is written down is the caller's decision,
  because the caller is the one with a database. See rule 3.
- **HOTP counter resynchronisation.** RFC 4226 section 7.4 describes a
  look-ahead window for a token whose button has been pressed more
  often than the server has seen. It is a loop over
  `totphotp.verify_at` and a new counter the caller stores, and this
  package stores nothing.
- **OCRA**, the challenge-response algorithm of
  [RFC 6287](https://www.rfc-editor.org/rfc/rfc6287). It is a different
  suite with its own data-input format.
- **Steam's and Authy's variants.** Steam uses a twenty-six character
  alphabet instead of digits, and Authy uses a seven-digit code on a
  ten second step. Neither is specified anywhere, and a package that
  guessed at them would be wrong quietly.
- **`otpauth-migration://` payloads.** Google Authenticator's export
  format is protocol buffers inside a URI, and it is not specified.
- **A password.** A one-time password is a second factor. The first one
  is a password hash, which is
  [bcrypt-nv](https://novo-lang.org/packages/bcrypt-nv) or a key
  derivation function.

## Related packages

- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is the HMAC
  underneath, and the constant-time comparison of rule 2. A caller that
  needs to hash something else already has it in the closure.
- [url-nv](https://novo-lang.org/packages/url-nv) parses the
  `otpauth://` URI and does its percent-encoding. Take it directly for
  URLs that are not enrolment URIs.
- [jwt-nv](https://novo-lang.org/packages/jwt-nv) is the other half of
  a session: a one-time password is checked once, at login, and a token
  carries the result of that check afterwards. It takes its clock as a
  parameter for the same reason this package does.
- [bcrypt-nv](https://novo-lang.org/packages/bcrypt-nv) is the first
  factor.
- [qrcode-nv](https://novo-lang.org/packages/qrcode-nv) renders the
  string `totpuri.format` answers.

## Test vectors

RFC 4226 Appendix D is the HOTP reference: the ASCII secret
`12345678901234567890` at counters 0 to 9, with the intermediate HMAC,
the truncated decimal and the six-digit code. RFC 6238 Appendix B is
the TOTP reference: eight-digit codes at six instants for all three
algorithms, with the caveat in rule 12. RFC 4648 section 10 supplies
the base32 vectors. The implementations to check a port against are the
Rust `totp-rs` crate and Python's `pyotp`.

```bash
novo test tests/totp_rfc4226_tests.nv    # HOTP, and its three stages
novo test tests/totp_rfc6238_tests.nv    # TOTP, the window and the replay rule
novo test tests/totp_base32_tests.nv     # RFC 4648, case and padding
novo test tests/totp_cover_tests.nv      # the algorithms, the URI, the refusals
```

The suite carries all thirty of Appendix D's published values, all
eighteen of Appendix B's, the seven base32 vectors both padded and
bare, and the counter arithmetic at each of Appendix B's instants. It
asserts that a leading zero survives, that a code from the previous
step is accepted and one from two steps back is not, that a code from a
step ahead is accepted only under a window that says so, that a match
at or below a stored counter is a replay, that a bad base32 character
is named with its offset, and that an enrolment URI round-trips through
a label containing a space and a colon.

The tests compile today and fail at run, each on the `not implemented:
totp-nv.<module>.<fn>` panic that is its body. That is the expected
state of an interface release. They turn green one at a time as bodies
land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `totpbase32.TOTP_BASE32_ALPHABET`, `.TOTP_BASE32_PAD` and the two group constants | yes (they are constants) |
| `totpparams.TOTP_DIGITS_DEFAULT`, `.TOTP_DIGITS_MIN`, `.TOTP_DIGITS_MAX`, `.TOTP_STEP_DEFAULT_SECONDS`, `.TOTP_T0_DEFAULT` | yes (they are constants) |
| `totphotp.TOTP_TRUNCATION_MASK`, `.TOTP_COUNTER_BYTES` | yes (they are constants) |
| `totpuri.TOTP_URI_SCHEME`, `.TOTP_URI_TYPE_TOTP`, `.TOTP_URI_TYPE_HOTP`, `.TOTP_URI_NO_COUNTER` | yes (they are constants) |
| `totperr.TotpError`, `totpalg.TotpAlgorithm`, `totpbase32.TotpSecret`, `totpparams.TotpParams`, `.TotpWindow`, `totptime.TotpInstant`, `.TotpMatch`, `totpuri.TotpUri` | the types are declared |
| `totperr.is_config_fault`, `.is_user_fault`, `.is_program_fault`, `.offset`, `.code`, `TotpError.message` | no |
| `totpbase32.char_value`, `.encoded_len`, `.decoded_len` | no |
| `totpbase32.encode_into`, `.encode`, `.decode_into`, `.decode` | no |
| `totpbase32.secret`, `.secret_of_base32`, `.secret_bytes`, `.secret_base32` | no |
| `totpalg.default_algorithm`, `.digest_bytes`, `.block_bytes`, `.alg_name`, `.alg_named` | no |
| `totpparams.window`, `.default_window`, `.exact_window`, `.window_steps` | no |
| `totpparams.default_params`, `.params`, `.check` | no |
| `totpparams.with_algorithm`, `.with_digits`, `.with_step`, `.with_t0`, `.with_window` | no |
| `totphotp.counter_bytes`, `.hmac_of_counter`, `.dynamic_truncate`, `.code_of_truncated` | no |
| `totphotp.hotp`, `.verify_at`, `.check_code` | no |
| `totptime.instant`, `.unix_seconds`, `.plus`, `.counter_at`, `.step_start`, `.seconds_remaining` | no |
| `totptime.totp`, `.verify`, `.no_match`, `.is_replay` | no |
| `totpuri.uri`, `.hotp_uri`, `.is_hotp`, `.label`, `.format`, `.parse` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
