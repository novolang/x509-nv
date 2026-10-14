# Changelog

All notable changes to x509-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-12

**The bodies.** Every `todo()` the interface release published is
written, and the four things this package is measured on are pinned by
tests rather than described: DER canonicality, the forms the reader
enforces and the one it does not; a validity window that is a range
check against a clock the CALLER supplies and is inclusive at both
ends; a name comparison that is a comparison of encodings and not of
text; and a truncated certificate at three nesting depths.
`novo test --isolate` is green on each of the seven suites — 83 tests
— `novo test tests` is green as a directory run, no run reports a
leak, and the merged per-line coverage of `src/` is 1305 of 1305
instrumented lines, so not one line needs a `// cov: skip` marker.

### Added

- `x509cert` — `parse_at` reading the version, the serial as the bytes
  DER wrote (the leading zero a positive number carries included), the
  two times, the two names, the key and the extensions, every one of
  them a span into the caller's buffer and none of them a copy.
  `is_valid_at` is `notBefore <= at <= notAfter` written as two
  `civil.compare_datetime` calls with `<= 0` on each, so the two ends
  are INSIDE the window and the second each one falls on is valid;
  `check_validity_at` answers which end failed and carries the index
  the caller gave.  `check_conformance` checks the six rules RFC 5280
  states about one certificate and then re-checks the tbs and
  document spans against the bytes actually present, so a span that
  lies about its own length is a refusal rather than a read past the
  end of the buffer.
- `x509ext` — the thirteen extensions decoded: basic constraints, the
  nine key-usage bits by the numbers RFC 5280 section 4.2.1.3 gives
  them, extended key usage, the two alternative names, the two key
  identifiers, name constraints with the subtree comparison,
  distribution points, certificate policies with their qualifiers, and
  the three policy-processing extensions read as values.
  `unknown_critical` and `check_unknown_critical` measure an extension
  against `known_oids()` plus the caller's own set — "unknown" is
  unknown to the PROGRAM, which is what lets a caller implement an
  extension of its own without forking.  `extension_value` returns the
  extension's DER whole and unparsed, because an extension's bytes are
  inside the signed region and re-encoding a parsed one would change
  the signature.  A value that is not exactly one DER value with
  nothing after it is refused by name.
- `x509crl` — a CRL read out of the caller's bytes, its revoked
  entries (serial, date, entry extensions), `is_revoked` as a pure
  lookup and `is_revoked_by` comparing the issuer names as well,
  `check_revoked` reporting the entry's reason and invalidity date,
  `crl_number` and `delta_crl_indicator`, `is_current_at` and
  `check_current_at` against the caller's instant, and the six OIDs.
- `x509csr` — a PKCS #10 request: the info span, the signature bytes,
  the attributes, `requested_extensions` naming the OID it read them
  from, and `check_conformance` refusing a repeated attribute type, a
  version above v1, an empty subject and an attribute value that is
  not the structure its OID names.
- `x509verify` — `verifier`, `no_verifier` and `combined` over the two
  function values; `supports` and `verify_signature`; the three
  verification calls and `is_self_signed`; the three outcome
  predicates; and `algorithm_name`, `is_rsa_algorithm` and the six
  ECDSA, RSA and RSASSA-PSS OIDs, so a certificate nothing on this
  registry can check is named rather than refused.
- `x509path` — `validate` as a pure function of chain, anchor,
  verifier, instant and policy.  The CA flag and the `keyCertSign`
  bit are asked of every certificate that SIGNED something, which is
  `i > 0` and not "not the last": a chain's leaf signed nothing, and
  requiring the CA flag of it refuses every ordinary chain of two.
  Name constraints, the key purpose requirement, the maximum chain
  length, `build_chain`, `check_chain_order`, `matches_hostname`
  (subjectAltName only, a wildcard matching one leftmost label) and
  every refusal carrying the index of the certificate that produced
  it.
- `x509name` — the six attribute OIDs RFC 5280 requires that asn1-nv
  does not already name, the nine short names, `attributes`,
  `common_name`, the RFC 4514 text form with its escaping, and
  `issuer_matches_subject`, which is `asn1pkix.names_encode_equal`
  and therefore a comparison of the ENCODINGS — two names that print
  the same can be different bytes, and two that print differently can
  be the same name's two string types.
- `x509err` — `code`, `message` through the `Error` trait,
  `is_malformed`, `is_rule_failure`, `is_validity_failure`, and
  `error_index`, `error_offset` and `decode_cause`.  Each sentence is
  built in one place so a free function and the trait method cannot
  drift apart.
- `tests/x509_api_tests.nv` — 28 tests taken from the README's own
  examples and numbered rules: every `fn main` block in the doc
  comments, both ends of the validity window as an argument rather
  than a clock, a name whose case differs, a certificate truncated at
  three depths, and the wire shapes a caller meets — a CSR attribute
  carrying two DER values, a hand-built `X509Cert` whose tbs and der
  spans are shorter than the document.
- `tests/x509_edge_tests.nv` — 8 tests for the arms the five oracle
  suites leave unvisited: four non-canonical encodings refused under
  `default_limits()` and accepted under `ber_limits()`, the same
  document answered two ways for two limits, a window's two ends and
  the second on each side of them, a whole chain walked to its anchor,
  name constraints with the check on and off, and a refusal that
  renders the decode cause underneath it.
- The toolchain floor moves to 0.20.2, the compiler this release was
  built, tested and measured against.

### Changed

- **Three reads were wrong in the first draft of the bodies and are
  fixed here**, each caught by a fixture rather than by reading.  The
  CRL's `signatureAlgorithm` and `issuer` were read from each other's
  positions — `TBSCertList` is `version, signature, issuer,
  thisUpdate`, and `i` only moves off zero when a version is present,
  so the two are the same read on a v1 list and swapped on every v2
  one.  A CRL entry's `crlEntryExtensions` is UNTAGGED in RFC 5280
  appendix A.1 — a bare SEQUENCE — and was read as though the `[0]`
  the LIST's own `crlExtensions` carries were there, so every reason
  and invalidity date a conforming writer wrote was dropped.  And
  `validate` marked a certificate an issuer when it was not the last
  in the chain (`i < top_index`) rather than when something below it
  was issued by it (`i > 0`), which asked the end-entity certificate
  for the CA flag and so refused every ordinary chain of two.
- **A workaround, with the filing named.**  No local in the suites
  takes a fixture's name; they are `bad_bc_refusal` and its kin, not
  `bad_bc`.  A local binding whose name matches a function, called as
  `name()`, is typed by the front end against the FUNCTION and lowered
  by the backend as a call through the local's own slot, which comes
  out as `E6000 internal compiler error` for a `Str`, as invalid LLVM
  IR for an `Int`, and as a SIGSEGV for an enum carrying a field.  The
  comment at the site names
  `codegen-llvm/a-call-to-a-name-bound-to-a-local-is-emitted-as-an-indirect-call-through-the`
  so a later reader can revert the rename when the filing closes.
- **A workaround for the formatter, with the filing named.**
  `novo fmt` formats seven of the eight modules and refuses
  `src/x509verify.nv` with `internal compiler error: novo fmt produced
  text that does not parse (inconsistent indentation) [E6000]` when a
  struct literal's field is a lambda written with its body on the next
  line.  Filed as
  `tooling/novo-fmt-refuses-a-file-whose-struct-literal-carries-a-multi-line-lambda`,
  with a minimal repro.  The workaround is that `combined` asks a
  named private function, `first_that_supports`, and keeps the
  lambda's body on its `=>` line; the comment above it names the
  filing so the declaration goes back into the literal when that
  closes.  Every file in the package now formats.
- The `attribute_short_name` doc block no longer claims that
  `to_rfc4514` writes RFC 4514 section 2.4's `#`-prefixed hexadecimal
  form: `AsnAttribute.value` is a `Str`, because `asn1pkix.name_at`
  reads every attribute value as a string, so there is no non-string
  value here to write in that form.  `to_rfc4514` writes the short
  name or the dotted OID, and the `### Known` section names the rest.
  The `x509ext.find` doc block likewise no longer claims that
  `check_conformance` refuses a certificate carrying one extension OID
  twice: it does not, and the second extension is simply never
  answered.  No signature and no effect row moved.
- No `redundant-bool` stands in this package: `novo.lint` reports zero
  warnings on all eight modules and all seven suites, and there is no
  `== false` or `!= true` under `src/` or `tests/` to rewrite.

### Fixed

- **A pathLenConstraint of 0 admits the end-entity certificate.** `x509path.validate` counted the end-entity certificate and every self-issued certificate against a CA's pathLenConstraint, so `strict()` refused a leaf issued by a CA with constraint 0, the ordinary shape. RFC 5280 section 4.2.1.9 bounds the non-self-issued intermediate certificates that follow the CA, so the end-entity certificate and self-issued intermediates are no longer counted. `x509_api_tests.nv` pinned the old answer for the leaf and CA(0) chain; it now pins the refusal of a leaf, a non-self-issued intermediate and CA(0). `validate` also no longer verifies each certificate against itself before its real issuer.

### Known

- **The time form RFC 5280 section 4.1.2.5 fixes is not enforced.**  A
  date before 2050 must be written UTCTime and a date from 2050 on as
  GeneralizedTime; this reader takes either form at either end of a
  validity period.  Measured: a certificate whose `notBefore` is
  `20491231000000Z` written as GeneralizedTime parses and is valid at
  that instant.  The rule is a writer's obligation here, and
  `x509name`, `x509cert` and the tests pin what is read rather than
  what is refused.
- **A certificate carrying one extension OID twice is not refused.**
  RFC 5280 section 4.2 forbids it.  `x509ext.find` answers the first
  match, and for a critical OID the second certificate's constraint is
  never seen.  A caller that needs the rule checks
  `list.len(c.extensions)` against the number of distinct OIDs it
  recognises.
- **`X509BadExtensionValue` reports `at: 0`.**  An
  `AsnExtension.value` is a detached copy of the octet string's
  content — asn1-nv's own shape — so there is no offset into the
  caller's buffer to report, and zero is what the arm carries.  The
  `oid` field names the extension, which is what a log line needs.
- **`check_certificate` takes `src` and does not read it.**  Every
  value the check needs travels inside the `X509Cert` it is handed,
  which is why a caller can run the single-certificate rule against a
  struct it built; the parameter is kept so a call reads the same as
  every other call in the package.
- **`permissive()` declares `nameConstraints` known and does not
  enforce it.**  A policy that did neither would refuse every
  constrained chain, which is the one thing a policy that checks "the
  shape and almost nothing else" must not do.  The site comment says
  so, and the policy's own doc lists what it turns off.
- **The `#`-prefixed hexadecimal value form of RFC 4514 section 2.4 is
  never written.**  It is for a value whose type is not a string, and
  `x509name.attribute_short_name` and `x509name.to_rfc4514` see only
  `Str` values, because `asn1pkix.name_at` reads every attribute value
  as a string.  A caller with a non-string attribute has its DER from
  `x509name.name_der`.
- **An unrecognised element inside a CRL entry is ignored, not
  refused.**  RFC 5280's `TBSCertList.revokedCertificates.SEQUENCE OF
  SEQUENCE { userCertificate, revocationDate, crlEntryExtensions
  OPTIONAL }` is what the reader accepts; a third element that is
  neither a SEQUENCE nor a `[0]` is skipped, because the only two
  optional members are told apart by their tags and nothing else is
  expected to be there.
- **A zero-length serial number is unreachable through `parse`.**  A
  DER INTEGER with no content octets is refused by asn1-nv before this
  package sees one, so `check_conformance`'s rule is reachable only
  through a struct the caller built — which is exactly the input that
  function takes, and what the api suite hands it.
- **`known_oids()` lists certificate extensions only.**  A CRL's
  `cRLNumber`, `cRLReason` and the rest are read as extensions by the
  CRL readers and are not in the set, so a critical one on a CRL is
  unknown unless the caller names it in `also_known`.
- **No PEM, no OCSP, no fetching, no issuance** — unchanged from the
  interface release, and the README's "What is not included" lists the
  reasons.

## 0.0.2 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- asn1-nv: `^0.0.1` to `^0.1.0`.
- calendar-nv: `^0.0.2` to `^0.2.0`.

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
