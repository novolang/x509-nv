# Changelog

All notable changes to x509-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `x509cert` — a certificate read out of the caller's own bytes. The
  decision the package rests on is that `X509Cert.tbs` is a SPAN and
  not a copy: a signature is computed over the DER of the
  tbsCertificate exactly as it arrived, and a parser that handed back
  fields for the caller to re-encode would produce different bytes for
  every certificate whose writer was not byte-for-byte canonical. The
  rule is inherited from asn1-nv and it is why an `X509Cert` is a
  description of a buffer rather than an owner of one — every function
  that needs bytes takes `src` first.
- `x509ext` — the thirteen extensions RFC 5280 defines, each as a
  typed value: basic constraints, the nine key-usage bits by their own
  numbers, extended key usage, subject and issuer alternative names,
  the two key identifiers, name constraints, CRL distribution points,
  certificate policies, and the three policy-processing extensions
  read but not processed. And `unknown_critical`, which is the rule
  section 4.2 requires a conforming implementation to apply. "Unknown"
  means unknown to the PROGRAM: a caller that implements an extension
  of its own hands in its OID and it stops being unknown, rather than
  having to fork the package or ignore the rule.
- `x509crl` — revocation lists. `is_revoked` is a pure lookup of a
  list and a serial, so a replay of a request log answers what it
  answered the first time. `is_revoked_by` compares the issuer names
  as well, because a serial is unique only under one issuer and a
  lookup in the wrong list answers "not revoked" confidently.
- `x509csr` — PKCS #10. The extensions a request carries are named
  `requested_extensions`, because that is what they are: a certificate
  authority that copied them into a certificate unexamined would
  certify whatever anybody asked for.
- `x509verify` — **the interface the package is built around.** The
  signature check is an `X509Verifier`: a value holding two functions
  the caller wrote, one answering whether an algorithm is implemented
  and one doing the check. A struct of function values rather than a
  trait, so that a program can build one at run time from whichever
  packages it happens to have and `combined` can put two together
  without either knowing about the other. `X509VerifyOutcome` has
  three arms rather than two, because "I have no implementation of
  that algorithm" is an answer and not a failure.
- `x509path` — path validation as a PURE FUNCTION. `validate` takes
  the chain, the trust set, the verifier, the instant and the policy,
  and reads no clock, no file and no socket. A test validates a chain
  at any instant with no fake clock; an audit tool asks whether a
  certificate was valid when it was presented; a replay gives the same
  answers. The policy is a VALUE narrowed from `strict()`, on jwt-nv's
  model, and every refusal names which certificate of the chain failed
  and why.
- `x509name` — distinguished names over asn1-nv's `AsnName`: the
  attribute types RFC 5280 requires that asn1-nv does not already
  name, the RFC 4514 text form with its escaping, and
  `issuer_matches_subject`, which compares the ENCODINGS because that
  is what chain building compares.
- `x509err` — the refusals, with `is_malformed` and `is_rule_failure`
  partitioning the enum. A certificate that does not parse is a broken
  file; one that parses and has expired is a certificate that needs
  renewing. They go to different people. `X509NotDer` carries
  asn1-nv's own `AsnDecodeError` whole, so a caller can ask whether
  the document was BER rather than DER.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the five API suites reaches `not implemented:
  x509-nv.<module>.<fn>`.
- **RSA is missing from the registry, and this package says so rather
  than hiding it.** A certificate signed `sha256WithRSAEncryption` —
  most of the public web — parses here and can be verified by nothing
  any novo-lang program can build today. The OIDs are published,
  `x509verify.algorithm_name` names them, and
  `x509verify.is_rsa_algorithm` identifies the family, so a caller is
  told what it is holding. The alternative — a `todo()` that panics —
  would make a missing package look like a broken certificate.
- **`X509Verifier.verify` declares the `[]` effect row**, which is
  what `core` means. A verifier backed by a hardware security module
  performs input and output and therefore cannot be passed here. That
  widening belongs to a `host` package wrapping this one; it is the
  one place in this package where a declared row wanted to be wider.
- **No dependency on a verification package.** Depending on one would
  pin an algorithm set: this package would be republished every time
  one was added, and every caller would carry an implementation it
  might not need. The verifier is the caller's.
- **The policy processing of RFC 5280 section 6.1 is named as missing,
  not stubbed.** The four policy extensions are read and reported and
  `X509Path.policies` lists what was asserted; the valid-policy tree,
  policy mapping and the inhibit counters are not built.
- **No PEM, no OCSP, no issuance, no network.** The armour around a
  DER block is base64-nv's; a revocation list's address is read and
  not fetched, because fetching it performs input and output and this
  package performs none.
- Every public type, enum variant and module name is prefixed `X509`
  or `x509`, so none of them collides with asn1-nv's, calendar-nv's or
  the standard library's.
