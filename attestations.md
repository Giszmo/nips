# NIP-XXXX

## Attestations

`draft` `optional`

### Abstract

This NIP introduces a method for users to make verifiable attestations about
entities, concepts, or events, which may or may not have a presence on the Nostr
network.
These attestations can be used to express claims, certifications, or
endorsements in a standardized format.

### Motivation

There is a need for a standardized way to make claims or attestations about
various subjects within the Nostr ecosystem. These attestations could range from
personal endorsements to professional certifications, and even to statements
about entities outside the Nostr network.

### Specification

#### Attestation Event

Attestations are represented by events with `kind 31000`.

The `content` field should contain a human-readable description of the
attestation.

The event MUST include the following tags:

- `d`: The entity, concept, or event being attested about. This could be a name,
       an identifier, or a description.
- `t`: MUST be set to `"attestation"`.
- `c`: A concise description of what is being attested.

The event MAY include the following optional tags:

- `evidence`: A URL or reference to supporting evidence for the claim.
- `expiration`: A Unix timestamp indicating when the attestation should be considered expired.

#### Examples

```json
{
  "kind": 31000,
  "content": "I, John Doe, attest that Company X uses organic tomatoes in their products based on my visit to their farm on June 1, 2024.",
  "tags": [
    ["d", "Company X"],
    ["t", "attestation"],
    ["c", "use organic tomatoes in their products"],
    ["evidence", "https://example.com/farm-visit-report"],
    ["expiration", "1735689600"]
  ]
}
```

```json
{
  "kind": 31000,
  "content": "I, John Doe, attest that the Bitcoin Core binary with sha256 hash 1345234231 can be reproduced from the source code revision 345213451",
  "tags": [
    ["d", "Bitcoin Core artifact 1345234231"],
    ["t", "attestation"],
    ["c", "reproducible"],
    ["git", "345213451"],
    ["version", "25.2-beta"],
    ["evidence", "a:<my pub key>:30023:BC_1345234231"],
    ["evidence", "https://example.com/buildLogBC25.2-beta.html"]
  ]
}
```

### Implementation Considerations

For complex or highly structured attestations, implementations might consider
using a JSON structure in the `content` field and including a schema reference.
