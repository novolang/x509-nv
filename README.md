# x509-nv

An X.509 certificate is the document that binds a public key to a name.
It is what a web server presents, what a firmware image is signed
against, and what a client validates before it trusts either. The
format, its extensions and the algorithm for validating a chain of them
are specified in
[RFC 5280](https://www.rfc-editor.org/rfc/rfc5280), which also defines
the certificate revocation list; the certification request is
[RFC 2986](https://www.rfc-editor.org/rfc/rfc2986), PKCS #10. This
package reads all three and validates a chain, on top of
[asn1-nv](https://novo-lang.org/packages/asn1-nv), which reads the DER
underneath.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

Two gaps are worth knowing before you read further.

**Nothing here verifies a signature.** The arithmetic is supplied by the
caller as a value, so this package pins no set of algorithms. A program
that has an elliptic-curve package verifies elliptic-curve signatures; a
program that has none still parses certificates, checks their dates,
their names and their extensions, and is told which signatures went
unchecked.

**There is no RSA anywhere on this registry.** A certificate signed
`sha256WithRSAEncryption` — which is most of the public web — parses
here and can be verified by nothing, today, in any novo-lang program.
The algorithm is named and reported rather than hidden: a caller is told
that is what it is holding.

## What a certificate is

A certificate has three fields. The **tbsCertificate** — "to be signed"
— holds everything the certificate asserts: a version, a serial number,
the algorithm the issuer signed with, the **issuer**'s name, a
**validity** period of two moments, the **subject**'s name, the
subject's public key, and the **extensions**. The
**signatureAlgorithm** names the algorithm again, outside. The
**signatureValue** is the issuer's signature over the tbsCertificate.

A **distinguished name** is the issuer's or the subject's: a sequence of
attributes, each named by an OBJECT IDENTIFIER, such as
`CN=example.com,O=Acme Ltd,C=GB`. An **OBJECT IDENTIFIER**, or OID, is a
path of numbers in a global tree — `2.5.29.19` — that names a thing with
no ambiguity anywhere in the world; certificates are full of them.

An **extension** is an OID, a `critical` flag, and a value whose
structure the OID decides. Extensions carry everything modern about a
certificate: whether it may sign other certificates, what its key may be
used for, and which hostnames it covers. The `critical` flag says that a
reader which does not understand this extension must refuse the
certificate rather than ignore it.

**Path validation** is what makes a certificate mean anything. On its
own it asserts a name and a key and nothing more. It becomes meaningful
when there is a chain from it to a **trust anchor** — a certificate the
reader was configured in advance to believe. Validation walks that
chain and checks, at every step, the signature, the validity period, the
names, and the extensions.

A **certificate revocation list**, or CRL, is how a certificate authority
says that a certificate it issued is no longer valid before the
certificate's own expiry. It is a signed list of serial numbers with the
date each was revoked.

A **certification request** is how somebody asks for a certificate. It
carries the name and the public key they want certified and is signed
with the matching private key, which proves they hold it.

| Quantity | Value |
| --- | --- |
| Certificate version with extensions | 3 |
| CRL version with extensions | 2 |
| PKCS #10 request version | 0 |
| Conforming serial number, at most | 20 bytes |
| Key usage bits defined | 9 |
| Extensions RFC 5280 defines and this package reads | 13 |
| Default bound on a chain's length | 8 |
| `keyCertSign` bit number | 5 |
| `anyPolicy` | `2.5.29.32.0` |
| `anyExtendedKeyUsage` | `2.5.29.37.0` |

## Install

```
novo pkg add x509-nv
```

## Example

```novo
use std.bytes
use civil
use asn1read
use x509cert
use x509ext
use x509name

fn main() [io]
    // The certificate's DER, as the host read it.  This package opens
    // no file: the bytes are the caller's, and stay the caller's.
    let der = bytes.zeros(0)

    // The moment the dates are checked against.  It is an argument,
    // read from the caller's own clock.
    let at = civil.datetime(civil.date_from_epoch_day(20713), civil.midnight())

    match x509cert.parse(der, asn1read.default_limits())
        Err(e)   => println(e.message())
        Ok(cert) =>
            // Who the certificate is for, written for a person.
            println(x509name.to_rfc4514(x509cert.subject_of(cert)))

            // Whether it is valid at that moment.
            println("${x509cert.is_valid_at(cert, at)}")

            // The hostnames it actually covers.  They are in the
            // subject alternative name extension and not in the
            // common name.
            match x509ext.subject_alt_name(x509cert.extensions_of(cert),
                                           asn1read.default_limits())
                Err(e)          => println(e.message())
                Ok(None)        => println("no subject alternative name")
                Ok(Some(names)) => println("${list.len(x509ext.dns_names(names))}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented:
x509-nv.<module>.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `x509cert` | A certificate read out of the caller's bytes: version, serial, issuer, subject, validity, public key, extensions, both algorithm identifiers, the signature, and the span the signature covers. |
| `x509ext` | The thirteen extensions RFC 5280 defines, each as a typed value, the OIDs that name them, the nine key-usage bits by number, and the call that lists the critical extensions nobody recognised. |
| `x509crl` | A revocation list: this-update, next-update, the revoked entries by serial with their dates and reasons, the list's own extensions, and the lookup by serial. |
| `x509csr` | A PKCS #10 certification request: subject, public key, the extensions it asks for, the challenge password, and the span its signature covers. |
| `x509verify` | The signature check as a value the caller supplies, the three-way outcome it answers, and the names of the signature algorithms a certificate can carry. |
| `x509path` | Trust anchors, the validation policy, and path validation as a function of a chain, a trust set, a verifier, an instant and that policy. |
| `x509name` | Distinguished names: the attribute types by OID, the RFC 4514 text form with its escaping, and the comparison a chain is built with. |
| `x509err` | The refusals, each carrying the byte it was found at or the certificate it is about, and the two predicates that separate a malformed document from a valid one that failed a rule. |

## How to choose an entry point

**`x509path.validate` is the whole job in one call.** A chain, a trust
set, a verifier, an instant and a policy go in; a validated path or a
refusal that names the certificate and the reason comes out. A program
deciding whether to trust a peer calls this.

**`x509cert.parse` and the `x509ext` readers are for a program that
wants to know what a certificate says.** A certificate viewer, an
inventory tool, a linter or a monitor that warns before an expiry needs
no chain and no trust set.

**`x509crl` is a separate question from both.** A certificate can be
perfectly valid and revoked. The list arrives as bytes the caller
fetched, and the lookup is separate so that a program can see whether it
did it.

**`x509verify.no_verifier` is for a program with no cryptography
package at all.** Everything above works with it: the path validates,
and every signature it could not check is reported by index.

## The rules a user needs

1. **A parse answers spans into your bytes, and never a copy.** A
   certificate's signature is computed over the DER bytes of its
   tbsCertificate exactly as they arrived, not over a re-encoding of
   the parsed fields. `X509Cert.tbs` is a span into the buffer you
   supplied, and `x509cert.tbs_bytes` copies it out when you want it.
   **You must keep that buffer for as long as you hold the
   certificate.** The rule is inherited from asn1-nv, whose readers all
   answer spans, and it is why a certificate whose writer was not
   byte-for-byte canonical still verifies. RFC 5280 section 4.1.1.2.

2. **An unknown critical extension is a reason to refuse, not to
   ignore.** RFC 5280 section 4.2 requires a reader that meets a
   critical extension it does not recognise to reject the certificate.
   The flag exists so a certificate authority can add a constraint and
   be sure no older reader will quietly skip it.
   `x509ext.check_unknown_critical` is that refusal, and a program that
   implements an extension of its own names its OID so that it stops
   being unknown.

3. **The moment is always an argument.** Nothing in this package reads a
   clock. `x509cert.is_valid_at`, `x509crl.is_current_at` and
   `x509path.validate` all take the instant you read from yours. A test
   pins an expiry without a fake clock, an audit tool asks whether a
   certificate was valid when it was *presented* rather than only
   whether it is valid now, and replaying a request log gives the
   answers it gave the first time.

4. **The signature check is a value you supply, so no algorithm set is
   pinned here.** An `X509Verifier` holds two functions: whether it
   implements an algorithm, and whether a signature verifies. A caller
   holding an elliptic-curve package builds one over it; a caller
   holding two combines them; a caller holding none passes
   `x509verify.no_verifier`.

5. **"I could not check that" is an answer and not a failure.**
   `X509VerifyOutcome` has three arms, not two. A signature under an
   algorithm your verifier does not implement is
   `X509SignatureUncheckable`, and the index lands in
   `X509Path.unchecked`. `x509verify.outcome_is_good` is false for it.
   A program that reports "valid" without reading `unchecked` is
   claiming something it was told it could not claim.

6. **Names are compared by encoding.** Building a chain means finding
   the certificate whose subject is this certificate's issuer, and the
   comparison is over the DER bytes. `x509name.to_rfc4514` is for a
   person and not for that comparison: two names that print the same can
   have different encodings. RFC 5280 section 7.1 defines a fuller
   matching algorithm, with case folding and string-type equivalence,
   which this package does not implement.

7. **Hostnames are in the subject alternative name, never in the common
   name.** RFC 6125 deprecated matching against the common name in 2011
   and RFC 9525 removed it in 2024. `x509path.matches_hostname` reads
   the extension, and a wildcard there matches one label in the leftmost
   position only.

8. **`basicConstraints` and `keyUsage` mean opposite things when
   absent.** A certificate with no `basicConstraints` is not a
   certificate authority. A certificate with no `keyUsage` has no
   restriction on what its key may be used for. RFC 5280 sections
   4.2.1.9 and 4.2.1.3.

9. **Validation policy is one value, not a set of arguments.**
   `x509path.strict()` checks everything; each narrowing call turns one
   check off or adds one requirement, and shows up in a diff. A service
   builds one policy, names it, and uses the same one everywhere.

10. **A serial number is bytes, and it is unique only under one
    issuer.** Twenty bytes does not fit an integer, and the leading zero
    DER writes on a positive number is part of the value. Looking a
    serial up in another issuer's revocation list finds nothing and
    answers "not revoked"; `x509crl.is_revoked_by` compares the issuer
    names as well.

## Running on a microcontroller

This package makes no device claim and ships no probe.

Reading a certificate is `Bytes`, `Str`, `Result` and lists of parsed
structures, and the embedded runtime with no heap allocator defines none
of them. The shape is wrong rather than the size: a certificate is a
tree of variable-length values whose count is decided by whoever wrote
the file, and there is no version of that which fits in a fixed frame.

A device that needs to check a public key against one it was provisioned
with does not need this package. It walks the DER headers and compares
an OID, and that is
[asn1-nv](https://novo-lang.org/packages/asn1-nv)'s `asn1tag`, which
does carry a device claim.

## Timing behaviour

- **Nothing in this package is timing-sensitive, because nothing in it
  touches a secret.** A certificate, a revocation list and a
  certification request are public documents. A private key never
  reaches any function here.
- **Signature verification, where the timing questions live, is not in
  this package.** It is in the verifier the caller supplies, and its
  timing behaviour is that package's to state.
- **Comparisons here are ordinary.** Name encodings, serial numbers and
  key identifiers are compared byte by byte with early exit. None of
  them is secret, and a constant-time comparison would be stating a
  property the data does not need.

## What is not included

- **RSA.** Nothing on this registry implements it. Certificates signed
  `sha256WithRSAEncryption`, `sha1WithRSAEncryption`, the SHA-384 and
  SHA-512 variants, and RSASSA-PSS all parse here, and every verifier
  anybody can build today answers `X509SignatureUncheckable` for them.
  The OIDs are published and `x509verify.is_rsa_algorithm` names the
  family, so a program can say which of its certificates it will never
  be able to check.
- **Signature arithmetic of any kind.** See rule 4. A parser that also
  verified would be a parser nobody could audit separately from the
  arithmetic, and the arithmetic is where the mistakes are.
- **PEM.** This package takes DER, as asn1-nv does.
  [base64-nv](https://novo-lang.org/packages/base64-nv) decodes the
  body of a `-----BEGIN CERTIFICATE-----` block, and stripping the
  armour around it is a handful of lines in the caller.
- **OCSP.** RFC 6960 is a request, a response and a network exchange,
  which is a package of its own on top of this one.
- **Fetching anything.** A revocation list's address is read from the
  certificate and not resolved; an issuer is never downloaded. Both are
  network work, and this package performs no input or output at all.
- **Issuing or signing certificates.** Turning a request into a
  certificate means choosing a serial, a validity period and an
  extension set, encoding a tbsCertificate and signing it. That is a
  certificate authority: a program with a policy, a key store and an
  audit log.
- **Writing certificates.** This package reads. asn1-nv's `asn1write`
  is where DER is written.
- **The policy processing of RFC 5280 section 6.1** — the valid-policy
  tree, policy mapping, and the inhibit and require counters.
  `certificatePolicies`, `policyMappings`, `policyConstraints` and
  `inhibitAnyPolicy` are read and reported; the tree is not built.
  `X509Path.policies` lists what each certificate asserted.
- **Delta revocation lists, applied.** `x509crl.delta_crl_indicator`
  says a list is a delta and names the base it applies to. Combining
  the two is left to the caller, which holds both.
- **Certificate transparency.** RFC 6962's signed certificate
  timestamps are carried as an extension this package does not
  recognise.
- **X.400 addresses and EDI party names.** Two of the nine general-name
  forms, written by nothing on the public internet. They arrive as
  `X509UnsupportedName` with their tag and their bytes.

## Related packages

- [asn1-nv](https://novo-lang.org/packages/asn1-nv) reads the DER
  underneath this package and parses the key structures — a public key
  file, a PKCS #8 private key, and a certificate's own fields. This
  package is the layer above: every extension RFC 5280 defines,
  revocation lists, certification requests and path validation.
- [base64-nv](https://novo-lang.org/packages/base64-nv) is what turns a
  PEM block into the bytes this package takes.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) supplies the
  civil dates a validity period is made of, and the comparison that
  decides whether an instant falls inside one.
- [jwt-nv](https://novo-lang.org/packages/jwt-nv) solves the same
  problem for a different document: a signed token whose claims cannot
  be read unverified. It takes its current time as an argument for the
  same reason this package does.
- A TLS package is the consumer this one was written for. It needs a
  peer's chain validated at a moment it chooses, against a trust set it
  was configured with, with a hostname matched against the subject
  alternative names.

## Tests

The suites are written against the signatures and are red until the
bodies land. They assert what the implementation will have to satisfy.

| Suite | What it asserts |
| --- | --- |
| `x509_cert_tests.nv` | The tbsCertificate's bytes come back exactly as they arrived; validity is checked against a supplied instant; the serial keeps its leading zero; names are compared by encoding and escaped by RFC 4514; the attribute types RFC 5280 requires. |
| `x509_ext_tests.nv` | The nine key-usage bits numbered as RFC 5280 section 4.2.1.3 numbers them, against the two encodings every certificate carries; an unknown critical extension refuses and a non-critical one does not; the extension and key-purpose OIDs; a name constraint says nothing about a form it does not mention. |
| `x509_path_tests.nv` | A signature nobody can check is an answer and not a failure; an RSA-signed certificate reports that nothing checked it; the verifier is a value and two combine; the policy narrows one check at a time; a chain that breaks names the certificate that broke it; a wildcard matches one label. |
| `x509_revocation_tests.nv` | The revocation lookup is a pure function of a list and a serial; a serial is unique only under one issuer; the reason numbers including the unused 7; a request is self-signed and its extensions are a request. |
| `x509_cover_tests.nv` | Every remaining public function, one case each: an absent extension is `Ok(None)`, a decode cause survives whole, and every signed structure goes through the same verifier. |

Where a byte vector appears it is one taken from RFC 5280's own ASN.1
module or from the encoding rules of ITU-T X.690 — the key usage
`03 02 01 86` a certificate authority carries, the `03 02 05 a0` a TLS
server carries, and the `2.5.29.*` and `1.3.6.1.5.5.7.3.*` identifiers.
Where no vector was certain the assertion is about the structure and the
rule, and the comment names the section it comes from.

## Implementation status

| Item | Implemented |
| --- | --- |
| The version, serial-length, key-usage-bit and chain-length constants | yes (they are constants) |
| `x509cert.X509Cert`, `x509ext.X509BasicConstraints`, `.X509KeyUsage`, `.X509GeneralName`, `.X509AuthorityKeyId`, `.X509GeneralSubtree`, `.X509NameConstraints`, `.X509DistributionPoint`, `.X509PolicyInfo`, `.X509PolicyQualifier` | the types are declared |
| `x509crl.X509Crl`, `.X509RevokedCert`, `.X509CrlReason`, `x509csr.X509Csr`, `.X509CsrAttribute` | the types are declared |
| `x509verify.X509Verifier`, `.X509VerifyOutcome`, `x509path.X509Anchor`, `.X509TrustSet`, `.X509PathPolicy`, `.X509Path`, `x509err.X509Error` | the types are declared |
| `x509cert.parse`, `.parse_at`, `.parse_all`, `.tbs_bytes`, `.tbs_span`, `.der_bytes`, `.signature_bytes`, `.serial_hex` | no |
| `x509cert.is_valid_at`, `.check_validity_at`, `.is_self_issued`, `.algorithms_agree`, `.check_conformance` | no |
| `x509cert.public_key_algorithm`, `.public_key_of`, `.issuer_of`, `.subject_of`, `.extensions_of`, `.has_extension`, `.version_of` | no |
| `x509ext`'s seventeen OID functions and `.known_oids` | no |
| `x509ext.find`, `.is_critical`, `.unknown_critical`, `.check_unknown_critical`, `.extension_value` | no |
| `x509ext.basic_constraints`, `.is_ca`, `.key_usage`, `.key_usage_of_bits`, `.key_usage_bit`, `.key_usage_name` | no |
| `x509ext.extended_key_usage`, `.has_key_purpose`, `.subject_alt_name`, `.issuer_alt_name`, `.dns_names`, `.ip_addresses`, `.general_name_text` | no |
| `x509ext.subject_key_identifier`, `.authority_key_identifier`, `.name_constraints`, `.subtree_contains`, `.constraints_permit` | no |
| `x509ext.crl_distribution_points`, `.distribution_point_uris`, `.certificate_policies`, `.asserts_policy` | no |
| `x509crl.parse`, `.tbs_bytes`, `.signature_bytes`, `.is_revoked`, `.is_revoked_by`, `.entry_for`, `.check_revoked`, `.entries_of`, `.entry_count` | no |
| `x509crl.revocation_reason`, `.reason_number`, `.reason_named`, `.reason_name`, `.invalidity_date`, `.certificate_issuer` | no |
| `x509crl.crl_number`, `.delta_crl_indicator`, `.is_current_at`, `.check_current_at`, `.extensions_of`, `.issuer_of`, and its six OID functions | no |
| `x509csr.parse`, `.info_bytes`, `.info_span`, `.signature_bytes`, `.subject_of`, `.public_key_of`, `.attributes_of`, `.attribute_by_oid` | no |
| `x509csr.requested_extensions`, `.requested_dns_names`, `.challenge_password`, `.check_conformance`, `.oid_extension_request`, `.oid_challenge_password` | no |
| `x509verify.verifier`, `.no_verifier`, `.combined`, `.supports`, `.verifier_name`, `.verify_signature` | no |
| `x509verify.verify_certificate`, `.verify_crl`, `.verify_csr`, `.is_self_signed` | no |
| `x509verify.outcome_is_good`, `.outcome_is_uncheckable`, `.outcome_code`, `.algorithm_name`, `.is_rsa_algorithm`, and its six OID functions | no |
| `x509path.anchor`, `.trust_set`, `.empty_trust_set`, `.anchors_of`, `.anchor_count`, `.with_anchor`, `.find_anchor` | no |
| `x509path.strict`, `.permissive`, `.with_max_chain_length`, `.allowing_unchecked_signatures`, `.knowing_extensions`, `.requiring_key_purpose`, `.without_validity_check`, `.without_name_constraints`, `.check_policy` | no |
| `x509path.validate`, `.validate_der`, `.build_chain`, `.check_chain_order`, `.check_certificate` | no |
| `x509path.leaf_of`, `.anchor_of`, `.path_length`, `.unchecked_of`, `.is_fully_checked`, `.policies_of`, `.matches_hostname`, `.names_permitted` | no |
| `x509name`'s six OID functions, `.attribute_short_name`, `.attributes`, `.attribute_values`, `.common_name`, `.rdn_count`, `.is_empty` | no |
| `x509name.to_rfc4514`, `.escape_rfc4514`, `.name_span`, `.name_der`, `.issuer_matches_subject` | no |
| `x509err.is_malformed`, `.is_rule_failure`, `.is_validity_failure`, `.error_index`, `.error_offset`, `.decode_cause`, `.code`, `X509Error.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
