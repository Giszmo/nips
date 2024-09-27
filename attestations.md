# NIP-XXXX

## Software Attestations

`draft` `optional`

### Abstract

This NIP introduces a method for users to make verifiable attestations about
software in general, specific releases or artifacts.
These attestations can be used to express claims, certifications, or
endorsements in a standardized format.

### Motivation

There is a need for a standardized way to make claims or attestations about
software. These attestations could range from personal endorsements to over
general feature descriptions to professional certifications of specific
artifacts.

### Specification

#### Attestation Event

Attestations are represented by events with `kind 31042`.

The `content` field should contain a human-readable description of the
attestation.

The event MUST include the following tags:

- `d`: Identifier of the software the attestation is about.
- `c`: A concise description of what is being attested.

The event MAY include the following optional tags:

- `v`: Version the attestation applies to.
- `h`: Hash of binary artifact the attestation applies to.
- `evidence`: A URL or reference to supporting evidence for the claim.
- `expiration`: A Unix timestamp indicating when the attestation should be considered expired.

#### Examples

```json
{
  "kind": 31042,
  "content": "I, John Doe, attest that the Bitcoin Core 78.0 binary with sha256 hash a9993fd3 and md5 hash 9e360c can be reproduced from the source code revision 72a149",
  "tags": [
    ["d", "Bitcoin Core"],
    ["c", "reproducible"],
    ["v", "78.0"],
    ["h", "a9993fd3b73968da10036cf6e65d1f3845e3878617f5dcff1338feaa4f274c41"],
    ["h", "9e360ce83a234c96bb1d921bfe62231c"],
    ["git", "72a1495b8afb3e3a4536a81e27e9fecb850e7981"],
    ["evidence", "a:<my pub key>:30023:BC_1345234231"],
    ["evidence", "https://example.com/buildLogBC25.2-beta.html"]
  ]
}
```

```json
{
  "kind": 31042,
  "content": "I, John Doe, failed to build Bitcoin Core 78.0 from the source code revision 72a149",
  "tags": [
    ["d", "Bitcoin Core"],
    ["c", "ftbfs"],
    ["v", "78.0"],
    ["git", "72a1495b8afb3e3a4536a81e27e9fecb850e7981"],
    ["evidence", "a:<my pub key>:30023:BC_1345234231"],
    ["evidence", "https://example.com/buildLogBC25.2-beta.html"]
  ]
}
```

```json
{
  "kind": 31042,
  "content": "The provider of the Android wallet 'Trust: Crypto & Bitcoin Wallet' does not publish the full source code.",
  "tags": [
    ["d", "android:com.wallet.crypto.trustapp"],
    ["c", "nosource"]
  ]
}
```

```json
{
  "kind": 31042,
  "content": "Sparrow supports multi signature, watch-only accounts but no lightning",
  "tags": [
    ["d", "sparrow"],
    ["c", "multisig"],
    ["c", "watchonly"],
    ["c", "-ln"]
  ]
}
```
