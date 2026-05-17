NIP-??
======

Collaborative Directories
--------------------------

`draft` `optional`

This NIP defines **collaborative directories** on Nostr: shared knowledge bases where many authors publish entries about subjects within a named **namespace**, and each reader resolves a single canonical version through their personal **Web of Trust**.

It introduces two new event kinds (`kind:30890` for directory entries, `kind:30891` for namespace definitions), three indexed tags, and a small set of trust rules added on top of standard NIP-01 addressable events. NIP-57 zap splits and NIP-54 merge requests are adopted unchanged.

---

## Motivation

Wikipedia works because anyone can edit. Its editorial model imposes a single global truth, producing edit wars, gatekeeping, and forks that are expensive to maintain. Nostr can do better: every user already has a personal social graph, so we can determine *whose edits a reader sees* without anyone agreeing on a global canonical version. The same event stream produces different views per reader — no fork required.

Existing Nostr primitives fall short of this:

- [NIP-78](78.md), [NIP-99](99.md): single author per record. No collaboration.
- [NIP-51](51.md): lists of references, not the primary data.
- [NIP-72](72.md): communities with fixed moderators. No reader-personalization.
- [NIP-54](54.md): multi-author addressable wikis, but resolution is "by some means" — undefined and not parameterized by trust.

**This NIP fills the gap.** One author (the namespace owner) defines a namespace and its rules. The rules state how deep into the Web of Trust edits are accepted — broadly for ontologies needing many contributors, narrowly for politically sensitive bodies of knowledge. Individual entries can tighten those rules further.

Use cases: software directories, relay registries, NIP indexes, glossaries, app stores, ontology vocabularies, curated lists.

---

## How It Works (Plain English)

1. **Alice creates a namespace** named `"nips"` via a `kind:30891` event. She declares her trust rules: e.g. `experts:3, follows:2` — meaning "my explicit expert list contributes up to 3 hops deep; my follows up to 2 hops deep."
2. **Anyone publishes entries** in that namespace using `kind:30890` with the `n` tag set to `"nips"` and a namespaced `d` tag like `"nips/nip-01"`. There are no signup barriers.
3. **A reader's client** picks an active namespace definition (default: the one provided by the application; logged-in users may switch to their own), computes who qualifies as a contributor under the active rule, and presents the newest qualifying entry per subject as canonical.
4. **Disagreements are not destructive.** Entries outside the trust threshold remain visible as proposals. Readers can switch perspective at any time.

---

## The Trust Model

### Two sources, two depths

Every namespace defines two parameters:

- `follows:N` — the **follow graph** is traversed up to N hops from the namespace owner. Hop 0 is the owner. Hop 1 is their direct follows (kind-3 contacts). Hop 2 is follows of follows. Etc.
- `experts:M` — an **explicit per-namespace expert list** is traversed up to M hops. Hop 0 is the owner. Hop 1 is the owner's named experts (`p` tags on the active `kind:30891`). Hop 2 is the experts in each of those L1 experts' own `kind:30891` events for this slug. Etc.

**Invariant:** `follows ≤ experts`. The follow graph bootstraps a namespace cheaply but is a noisy approximation of expertise; trusting it deeper than the curated path inverts the quality signal.

If `follows` is omitted from a declaration, it defaults to `0`.

The owner is always at hop 0 of both graphs by definition. A pubkey is a **contributor** if it is reachable from the owner within either limit. Its **hop distance** is the minimum hops via either path.

### Resolution algorithm

Given all `kind:30890` events with the same `(n, d)`:

1. **All contributors within the rule compete as peers.** Newest `created_at` wins. Ties broken by lowest event id lexicographically.
2. **The owner is a peer in normal editing**, but retains two important privileges: setting the namespace rules, and tightening any individual entry with `["follows", "0"]` or `["experts", "0"]` to restrict who can override it. To prevent contributors from squatting embarrassing slugs (e.g. an expert publishing a mocking entry for `<owner-pubkey>`), the owner can set `follows:0, experts:0` on the namespace, restricting contributions to themselves.
3. **Non-contributors** are shown as proposals, not as canonical content.
4. **If no contributor has published**, the newest event by any author is shown with a clear "outside trust" indicator.

Smaller hop distance does not beat larger on its own — once both authors qualify, newest wins. This avoids "who is more trusted" debates among equals.

### Per-entry tightening

Any contributor MAY tighten the rules on their own entry by including:

- `["experts", "0"]` — locks the entry at the expert hops up to the author's own hop. Experts deeper than the author's hop cannot override.
- `["follows", "0"]` — excludes the follow graph entirely for this entry, even if the namespace allows follows.

A per-entry value MUST be smaller than or equal to the namespace value; larger values are ignored. Omitting a per-entry parameter inherits the namespace value.

The effect: any contributor can be stricter than the namespace, never more permissive. `["follows", "0"]` on an entry removes the noisy follow signal from that entry, leaving it to the curated expert path only. Peers at the same hop remain able to override.

### Whose namespace definition is read?

The active `kind:30891` is the one the client references (typically by `a`-tag style `30891:<owner-pubkey>:nips`). Only this event's `experts:N` and `follows:N` parameters control resolution.

Other pubkeys may publish their own `kind:30891` events for the same slug — those are alternate views, used only when the client switches to that owner.

When traversing the **experts graph**, the client follows the chain hop-by-hop: hop 1 is the active owner's named experts; hop 2 is the named experts in each L1 expert's own `kind:30891` for this slug; etc.

**Important:** Only the **expert list** (`p` tags) is consulted from L1/L2 namespace definitions found during graph traversal. The `experts:N` and `follows:N` *depth parameters* on those non-active definitions are **ignored**. Depth is set exclusively by the active owner.

**A pubkey's expert list replaces their own follows at their hop — only at their hop.** If a pubkey at hop k has published a `kind:30891` for this slug containing `p` tags, then for the purpose of contributing hop k+1 candidates, only their listed experts are used; their follows are ignored. This decision is local: it does not propagate to deeper hops. A hop-k+1 pubkey who maintains no list of their own contributes their follows to hop k+2 as usual.

This is not a blacklist mechanism. If Alice maintains an expert list to exclude Eve from hop 1, but Bob (Alice's expert at hop 1) follows Eve, then Eve re-enters as a hop-2 contributor through Bob's follows under the namespace's `follows:N` rule. To reliably exclude a pubkey, the owner must either tighten the namespace to `follows:1, experts:1`, or layer a separate blacklisting mechanism on top (e.g. NIP-51 mute lists).

---

## Event Format

### `kind:30891` — Namespace Definition

```jsonc
{
  "kind": 30891,
  "pubkey": "<owner-pubkey>",
  "created_at": 1714299600,
  "tags": [
    ["d", "nips"],
    ["name", "Nostr Implementation Possibilities"],
    ["description", "Index of NIP proposals with status and discussion."],
    ["experts", "3"],
    ["follows", "2"],
    ["p", "<expert-pubkey>", "<relay-url>"],
    ["p", "<expert-pubkey>", "<relay-url>"]
  ],
  "content": "Long-form description of the namespace in Djot.",
  "id": "...",
  "sig": "..."
}
```

- `d` is the namespace slug.
- `experts` and `follows` declare trust depth. Omission of `follows` defaults to `0`.
- `p` tags list **experts** (hop 1 for the experts graph).
- Replaceable event; the owner can revise rules at any time.

### `kind:30890` — Directory Entry

```jsonc
{
  "kind": 30890,
  "pubkey": "<author-pubkey>",
  "created_at": 1714299600,
  "tags": [
    ["d", "nips/nip-01"],
    ["n", "nips"],
    ["title", "NIP-01: Basic protocol flow"],
    ["web", "https://github.com/nostr-protocol/nips/blob/master/01.md"],
    ["status", "active"],
    ["t", "core"]
  ],
  "content": "NIP-01 defines the basic protocol flow: events, signatures, and relay communication. See the linked document for the full specification.",
  "id": "...",
  "sig": "..."
}
```

- `d` is **namespaced**: `<n>/<subject>`, e.g. `"nips/nip-01"`. This guarantees the NIP-01 addressable ID (`30890:<pubkey>:nips/nip-01`) is globally unique across all namespaces, preventing collisions with unscoped wiki articles or other directories.
- `n` is the namespace slug, **required** for relay filtering. It MUST match the prefix of the `d` tag.
- Optional tightening: `["experts", "0"]`, `["follows", "0"]`.
- `content` holds free-form Djot text.

### Metadata tags

Structured metadata lives in tags, not content. Unknown tags are ignored.

| Tag | Purpose |
|-----|---------|
| `title` | Display name |
| `web` | Primary URL |
| `git` | Source repository |
| `status` | `active`, `beta`, `deprecated`, etc. |
| `icon` | Logo URL |
| `license` | SPDX identifier |
| `version` | Current version |
| `t` | Category (NIP-01 indexed) |

Namespaces MAY define additional tags as convention.

---

## The `n` Tag

`n` is a new single-letter indexed tag carrying the namespace slug.

- Exactly one `n` tag per directory entry.
- MUST match the prefix before `/` in the `d` tag.

```jsonc
{"kinds": [30890], "#n": ["nips"]}
```

> **Note on collision:** NIP-66 and NIP-87 also use `n`, but on different kinds. Relay filtering requires matching `kinds` and `#n` together, making the collision benign in practice.

---

## Examples

### Example 1: Broad, social directory — `cool-nostr-apps`

Alice wants a community-driven list. She casts a wide net:

```jsonc
{
  "kind": 30891,
  "pubkey": "<alice>",
  "tags": [
    ["d", "cool-nostr-apps"],
    ["name", "Cool Nostr Apps"],
    ["description", "Apps the community thinks are worth trying."],
    ["experts", "3"],
    ["follows", "3"]
  ],
  "content": "Curated by Alice. Anyone in my 3-hop graph can contribute.",
  ...
}
```

Bob (follows Alice) publishes an entry for Amethyst:

```jsonc
{
  "kind": 30890,
  "pubkey": "<bob>",
  "tags": [
    ["d", "cool-nostr-apps/amethyst"],
    ["n", "cool-nostr-apps"],
    ["title", "Amethyst"],
    ["web", "https://github.com/vitorpamplona/amethyst"],
    ["t", "client"], ["t", "Android"]
  ],
  "content": "Amethyst is a feature-rich Nostr client for Android.",
  ...
}
```

Bob is hop 1 in Alice's follow graph. His entry is canonical for readers whose active view is Alice's.

### Example 2: Tight, political directory — `nips`

Alice wants `nips` to be authoritative. She disables follows and keeps a short expert list:

```jsonc
{
  "kind": 30891,
  "pubkey": "<alice>",
  "tags": [
    ["d", "nips"],
    ["name", "Nostr Implementation Possibilities"],
    ["experts", "2"],
    ["p", "<expert-vitor>", "wss://..."],
    ["p", "<expert-fiatjaf>", "wss://..."]
  ],
  "content": "Only my named experts and their experts may contribute.",
  ...
}
```

Vitor (hop 1 expert) publishes an entry for NIP-01. A random follower of Alice has no contribution rights here, even though they could edit `cool-nostr-apps`.

### Example 3: Per-entry tightening

In `cool-nostr-apps`, Alice publishes a definitive entry for noStrudel and locks it down:

```jsonc
{
  "kind": 30890,
  "pubkey": "<alice>",
  "tags": [
    ["d", "cool-nostr-apps/nostrudel"],
    ["n", "cool-nostr-apps"],
    ["title", "noStrudel"],
    ["follows", "0"],
    ["experts", "0"]
  ],
  "content": "Alice's official entry for noStrudel.",
  ...
}
```

Even though the namespace allows `follows:3, experts:3`, this entry accepts no overrides from anyone past hop 0. Because Alice is at hop 0, only she can publish a canonical version; anyone else's entry for this `d` becomes a proposal.

If instead **Bob** (at hop 1 in Alice's follows) publishes this entry with `["experts", "0"], ["follows", "0"]`, the lock applies at his hop level: only Bob and peers at hop 1 or closer can override. Alice (at hop 0) can still override Bob's lock by publishing her own newer entry.

### Example 4: Zap splits

```jsonc
{
  "kind": 30890,
  "pubkey": "<bob>",
  "tags": [
    ["d", "cool-nostr-apps/amethyst"],
    ["n", "cool-nostr-apps"],
    ["title", "Amethyst"],
    ["zap", "<vitor>", "wss://...", "4"],
    ["zap", "<bob>", "wss://...", "2"],
    ["zap", "<alice>", "wss://...", "1"]
  ],
  "content": "...",
  ...
}
```

Zaps split: 57% to the project, 29% to the contributor (Bob), 14% to the curator (Alice).

### Example 5: Resolution walkthrough

Active view: Alice's namespace `cool-nostr-apps` with `experts:3, follows:2`. Subject: `amethyst` at `d: cool-nostr-apps/amethyst`.

| Author | Hop in follows | Hop in experts | created_at | Result |
|--------|---------------|----------------|------------|--------|
| Alice  | 0             | 0              | 1000       | Alternative (peer) |
| Bob    | 1             | —              | 1200       | Alternative (peer) |
| Carol  | 2             | —              | 1500       | **Canonical** (newest qualifying) |
| Dave   | 3             | —              | 1600       | Proposal (beyond `follows:2`) |
| Eve    | —             | 2              | 1400       | Alternative (peer) |
| Frank  | —             | 3              | 1300       | Alternative (peer, within `experts:3`) |

Carol wins — newest among all who qualify. The owner's role is rule-setting, not editorial priority: Alice competes on the same terms as Bob, Carol, Eve, and Frank.

If Alice wants her entry to be the only canonical version for this slug, she sets `["follows", "0"], ["experts", "0"]` on her own entry. If she wants the whole namespace to be hers alone, she sets `follows:0, experts:0` on the namespace definition.

---

## Merge Requests

NIP-54 merge requests (`kind:818`) work unchanged for this directory kind. They are a courtesy notification, not a gate: WoT resolution determines canonical content on its own.

Clients MAY skip merge request support entirely.

---

## Value Attribution (Zap Splits)

Entries SHOULD use NIP-57 Appendix G zap splits:

```jsonc
["zap", "<project-pubkey>", "<relay-url>", "<weight>"]
```

Recommended pattern: most weight to the project, some to the contributor maintaining the entry, some to the namespace owner for ongoing curation.

Clients SHOULD display the split visibly before zapping.

---

## Relay Queries

All entries in a namespace:
```jsonc
{"kinds": [30890], "#n": ["nips"]}
```

One subject across all authors:
```jsonc
{"kinds": [30890], "#n": ["nips"], "#d": ["nips/nip-01"]}
```

Owner's own entries:
```jsonc
{"kinds": [30890], "authors": ["<owner>"]}
```

Namespace definitions:
```jsonc
{"kinds": [30891], "#d": ["nips"]}
```

---

## Security Considerations

**Timestamp manipulation.** Clients SHOULD reject events with `created_at` significantly in the future, and MAY surface competing versions rather than picking one silently.

**Follow-graph capture.** The follow graph is the cheap path. Curators of sensitive namespaces SHOULD set `follows:0` and rely on explicit experts.

**Different owners, same name.** Alice's `"nips"` and Bob's `"nips"` are different namespaces. Clients MUST display the active owner.

**Replaceable events erase history.** Archival relays are RECOMMENDED for namespaces that need provenance.

---

## Rationale

**Why two new kinds instead of extending NIP-54?** `kind:30818` (NIP-54 wiki) and `kind:30819` (NIP-54 wiki headings) treat every event as a standalone article in an unscoped global slug space. Collaborative directories introduce namespace-scoped subject slugs, trust-parameterized resolution, and per-entry tightening — concepts that would distort the wiki semantics if shoehorned onto the same kind. A separate kind keeps both models clean: wiki clients ignore directory entries, and directory clients need only subscribe to `kind:30890`.

**Why namespaced d-tags?** NIP-01 addressable events are keyed by `(kind, pubkey, d)`. Without namespacing, a subject slug like `bitcoin` is global across every namespace and across unscoped wiki events. Prefixing the `d` tag with the namespace (`"nips/nip-01"`) guarantees globally unique addressable IDs without collision.

**Why two separate depth parameters?** The follow graph is large and cheap — good for casting a wide net. The expert graph is small and curated — good for political legitimacy. Separating them lets a namespace be simultaneously open (high `follows`) and authoritative (deep `experts`).

**Why `follows ≤ experts`?** The follow graph is a noisy approximation of expertise; trusting it deeper than the curated path inverts the quality signal.

**Why no NIP-05 verification for namespaces?** Domain ownership doesn't prevent squatting. Namespaces aren't DNS names. Reputation emerges from WoT, not certificates.

**Why is merge-request display optional?** WoT resolution determines canonical content. Merge requests are social signals, not gates.
