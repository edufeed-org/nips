NIP-VOCAB
=========

Vocabularies
------------

`draft` `optional`

This NIP defines a system for publishing controlled vocabularies, taxonomies, thesauri, and classification schemes as Nostr events. It is inspired by [SKOS](https://www.w3.org/TR/skos-reference/) (Simple Knowledge Organization System) but designed natively for Nostr.

## Motivation

Nostr has labeling infrastructure ([NIP-32](32.md)) and domain-specific classification patterns ([NIP-AMB](AMB.md), [NIP-35](35.md)), but no general-purpose mechanism for publishing, discovering, and using shared controlled vocabularies on the protocol itself. Vocabulary definitions currently live on external web servers. This NIP brings them onto Nostr, making them decentralized, discoverable, and maintainable by anyone.

## Event Kind

This NIP uses `kind:39737` for all published vocabulary-related events (concept schemes, concepts, and collections). These are addressable events, identified by `kind:pubkey:d-tag`.

`kind:39736` is reserved for work-in-progress drafts of these events, with identical structure. See [Drafts](#drafts).

A `type` tag distinguishes between the three event types:

| Type | Description |
|------|-------------|
| `ConceptScheme` | A vocabulary — an aggregation of concepts |
| `Concept` | A unit of thought — the fundamental building block |
| `Collection` | A labeled group of concepts within a scheme |

## Concept Scheme

A concept scheme represents a vocabulary, taxonomy, or classification system.

The Nostr event itself provides standard metadata: `created_at` serves as the publication/modification timestamp and `pubkey` identifies the publisher. Publishers MAY add a `version` tag for explicit versioning.

```jsonc
{
  "kind": 39737,
  "tags": [
    ["d", "schulfaecher"],
    ["type", "ConceptScheme"],
    ["prefLabel", "Schulfächerliste", "de"],
    ["prefLabel", "School Subject List", "en"],
    ["description", "Classification of subjects taught in German schools", "en"],
    ["a", "39737:<pubkey>:s10", "<relay>", "hasTopConcept"],
    ["a", "39737:<pubkey>:s20", "<relay>", "hasTopConcept"],
    // optional: bridge to external URI
    ["i", "http://w3id.org/kim/schulfaecher/"]
  ],
  "content": ""
}
```

### Concept Scheme Tags

| Tag | Value | Required | Description |
|-----|-------|----------|-------------|
| `d` | identifier | yes | Stable identifier for the scheme |
| `type` | `ConceptScheme` | yes | Distinguishes from Concept and Collection events |
| `prefLabel` | label, language | yes | Preferred display label. Third element is a BCP47 language tag. There MUST be at most one `prefLabel` per language. Repeat for multiple languages |
| `description` | text, language | no | Description of the scheme. Third element is a BCP47 language tag |
| `a` | coordinates, relay, `hasTopConcept` | yes* | References to top-level concepts. Required if the scheme has a hierarchy |
| `i` | external URI | no | Bridge to an external URI identifier for this scheme |

## Concept

A concept represents a single term, category, or idea within a vocabulary.

```jsonc
{
  "kind": 39737,
  "tags": [
    ["d", "s1017"],
    ["type", "Concept"],
    ["prefLabel", "Mathematik", "de"],
    ["prefLabel", "Mathematics", "en"],
    ["altLabel", "Mathe", "de"],
    ["altLabel", "Maths", "en"],
    ["notation", "s1017"],
    // scheme membership
    ["a", "39737:<pubkey>:schulfaecher", "<relay>", "inScheme"],
    // hierarchical relations (non-top concept — has broader)
    ["a", "39737:<pubkey>:s10", "<relay>", "broader"],
    ["a", "39737:<pubkey>:s101701", "<relay>", "narrower"],
    // associative relation
    ["a", "39737:<pubkey>:s1018", "<relay>", "related"],
    // optional: bridge to external URI
    ["i", "http://w3id.org/kim/schulfaecher/s1017"]
  ],
  "content": "Mathematics as taught in German school curricula"
}
```

### Concept Tags

| Tag | Value | Required | Description |
|-----|-------|----------|-------------|
| `d` | identifier | yes | Stable identifier for the concept |
| `type` | `Concept` | yes | Distinguishes from ConceptScheme and Collection events |
| `prefLabel` | label, language | yes | Preferred display label. Third element is a BCP47 language tag. There MUST be at most one `prefLabel` per language |
| `altLabel` | label, language | no | Alternative label (synonym, abbreviation). Third element is a BCP47 language tag |
| `hiddenLabel` | label, language | no | Hidden label for search indexing (catches misspellings, legacy terms). Not displayed to users |
| `notation` | code | no | Classification code or notation within the scheme |
| `a` | coordinates, relay, `inScheme` | yes | Every concept MUST reference its containing ConceptScheme |
| `a` | coordinates, relay, `broader` | yes* | Required for non-top concepts. Top concepts omit this and are referenced via `hasTopConcept` on the scheme |
| `a` | coordinates, relay, `topConceptOf` | no | Top concepts SHOULD include this marker pointing back to their scheme |
| `a` | coordinates, relay, `narrower` | yes* | Required if the concept has children. The child MUST have a corresponding `broader` tag |
| `a` | coordinates, relay, marker | no | Other relations: `related`, mapping markers (see [Relations](#relations)) |
| `r` | external URI, marker | no | Mappings to external vocabularies (see [External Mappings](#external-mappings)) |
| `i` | external URI | no | Bridge to an external URI identifier for this concept |
| `definition` | text, language | no | Formal definition. Third element is a BCP47 language tag. The `content` field serves as the primary definition; `definition` tags provide additional translations |
| `scopeNote` | text, language | no | Intended usage scope. Third element is a BCP47 language tag |
| `example` | text, language | no | Usage example. Third element is a BCP47 language tag |
| `note` | text, language | no | Generic documentation note. Third element is a BCP47 language tag |

A concept MAY belong to multiple concept schemes by including multiple `inScheme` tags.

The `content` field SHOULD contain the concept's definition or scope note.

> **Note:** SKOS also defines `historyNote`, `changeNote`, and `editorialNote`. These are intentionally omitted — Nostr's addressable event replacement provides implicit version history.

### Label Integrity

The following constraints MUST be observed:

- At most one `prefLabel` per language per concept
- `prefLabel`, `altLabel`, and `hiddenLabel` are pairwise disjoint: the same literal MUST NOT appear as more than one label type on the same event

## Collection

A collection is a meaningful grouping of concepts within a scheme, used for organizational purposes.

```jsonc
{
  "kind": 39737,
  "tags": [
    ["d", "bachelor-programs"],
    ["type", "Collection"],
    ["prefLabel", "Bachelor Programs", "en"],
    ["prefLabel", "Bachelor-Studiengänge", "de"],
    ["a", "39737:<pubkey>:schulfaecher", "<relay>", "inScheme"],
    ["a", "39737:<pubkey>:civil-eng", "<relay>", "member"],
    ["a", "39737:<pubkey>:mech-eng", "<relay>", "member"]
  ],
  "content": ""
}
```

Collections are organizational only. Semantic relations (`broader`, `narrower`, `related`) MUST NOT be used on or target Collection events.

> **Note:** SKOS also defines `OrderedCollection` for collections with meaningful ordering. This is intentionally omitted from this NIP for simplicity and may be specified in a future extension.

## Drafts

Publishers MAY save work-in-progress vocabulary events using `kind:39736`, which has the same structure as `kind:39737`. This parallels [NIP-23](23.md)'s use of a separate kind (`kind:30024`) for long-form drafts.

Drafts are addressable events identified by `kind:39736:pubkey:d-tag`. The `d` tag SHOULD match the `d` tag that the publisher intends to use when publishing to `kind:39737`, so draft-to-published transitions are traceable.

Drafts carry the full `ConceptScheme`, `Concept`, or `Collection` payload (see above) including the `type` tag. A draft of a concept scheme uses `["type","ConceptScheme"]` and so on.

### Client behavior

- Clients that display **published** vocabularies (explore views, search, referencing) MUST filter by `kind:39737` and MUST NOT surface `kind:39736` events as published vocabularies.
- Clients that offer **draft management** (the author's own editor, collaborative workspaces) SHOULD load `kind:39736` events alongside `kind:39737` events authored by the viewing user.
- Relations between vocabulary events (`a` tags with markers) SHOULD point to the author's intended published coordinates — `39737:<pubkey>:<d>` — even when the target currently exists only as a draft, so that relations resolve automatically once the target is published. Clients MAY also resolve drafts locally while editing.
- Clients publishing a draft's final version as `kind:39737` SHOULD delete the corresponding draft via [NIP-09](09.md) once publication is confirmed.

### Relay policy

Relays MAY accept or reject `kind:39736` per their policy. Public discovery relays MAY choose to reject drafts; private or author-scoped relays (e.g., a user's outbox) SHOULD accept them to support cross-device editing.

### `published_at` tag (optional)

On `kind:39737` events, publishers MAY include a `published_at` tag carrying the unix timestamp (stringified) of the first publication. This parallels [NIP-23](23.md)'s optional `published_at` and is useful because a replaceable event's `created_at` is updated on every edit, losing the original publish time. The `published_at` tag is **strictly optional** and is **not** used to distinguish drafts from published vocabularies — that distinction is carried by the event kind.

## Relations

All relations between vocabulary events use `a` tags with a marker (fourth element) indicating the relation type. This reuses the established Nostr pattern for typed references to addressable events (see [NIP-AMB](AMB.md), [NIP-53](53.md)).

### Hierarchical Relations

| Marker | Inverse | Description |
|--------|---------|-------------|
| `broader` | `narrower` | Direct parent in the hierarchy |
| `narrower` | `broader` | Direct child in the hierarchy |

Publishers MUST assert **both directions**: when a concept has a `broader` tag, the broader concept MUST include a corresponding `narrower` tag pointing back. This enables traversal in both directions without requiring additional queries.

Only assert `broader`/`narrower` for **direct** parent-child relationships. Transitive ancestry is inferred by clients traversing the chain.

### Associative Relations

| Marker | Symmetric | Description |
|--------|-----------|-------------|
| `related` | yes | Non-hierarchical association between concepts |

Publishers SHOULD assert both directions: if concept A is `related` to concept B, then B SHOULD include a corresponding `related` tag pointing to A.

A concept MUST NOT be both hierarchically related (directly or transitively) and associatively related to the same concept.

### Scheme and Collection Relations

| Marker | Used on | Description |
|--------|---------|-------------|
| `inScheme` | Concept, Collection | Links to the containing ConceptScheme |
| `hasTopConcept` | ConceptScheme | Links to top-level concepts |
| `topConceptOf` | Concept | Links a top concept back to its ConceptScheme |
| `member` | Collection | Links to a concept or nested collection belonging to this collection |

### Mapping Relations (Nostr-to-Nostr)

For linking concepts **across different concept schemes** published on Nostr:

| Marker | Symmetric | Description |
|--------|-----------|-------------|
| `exactMatch` | yes | Concepts are interchangeable in all contexts |
| `closeMatch` | yes | Concepts are similar but not always interchangeable |
| `broadMatch` | no | This concept is narrower than the target (cross-scheme) |
| `narrowMatch` | no | This concept is broader than the target (cross-scheme) |
| `relatedMatch` | yes | Non-hierarchical association across schemes |

`exactMatch` MUST NOT be combined with `broadMatch` or `relatedMatch` for the same concept pair. `exactMatch` implies `closeMatch` — every exact match is inherently a close match. Clients querying for close matches SHOULD also include exact matches in their results.

`exactMatch` is transitive: if concept A is an exact match of B, and B is an exact match of C, then A is also an exact match of C. Clients resolving mappings SHOULD account for transitive chains.

For symmetric mapping relations (`exactMatch`, `closeMatch`, `relatedMatch`), publishers SHOULD assert both directions. `narrowMatch` is the inverse of `broadMatch` — publishers SHOULD assert both directions: when a concept has a `broadMatch` tag, the target concept SHOULD include a corresponding `narrowMatch` tag pointing back, and vice versa.

`broadMatch`, `narrowMatch`, and `relatedMatch` are cross-scheme counterparts of `broader`, `narrower`, and `related` respectively. Clients aggregating semantic relations SHOULD include mapping relations in their results.

By convention, mapping markers link concepts in **different** schemes.

Example — mapping between two Nostr-native vocabularies:

```jsonc
{
  "kind": 39737,
  "tags": [
    ["d", "video"],
    ["type", "Concept"],
    ["prefLabel", "Video", "de"],
    ["prefLabel", "Video", "en"],
    ["a", "39737:<pubkey>:hcrt", "<relay>", "inScheme"],
    // mapping to another publisher's vocabulary
    ["a", "39737:<other-pubkey>:moving-image", "<relay>", "exactMatch"]
  ],
  "content": ""
}
```

## External Mappings

For mappings to concepts in external URI-based vocabularies (that are not published on Nostr), use `r` tags with a mapping marker:

```jsonc
["r", "https://w3id.org/kim/hcrt/video", "exactMatch"],
["r", "http://purl.org/dcx/lrmi-vocabs/mediaType/Video", "closeMatch"],
["r", "http://purl.org/dc/dcmitype/MovingImage", "broadMatch"]
```

The same mapping markers apply: `exactMatch`, `closeMatch`, `broadMatch`, `narrowMatch`, `relatedMatch`.

This is distinct from the `i` tag, which asserts identity ("this concept IS that external URI"), while `r` tags with mapping markers assert a relationship ("this concept CORRESPONDS TO that external concept").

## External Identity Bridge

The `i` tag (per [NIP-73](73.md)) provides a bridge to external URI-based identifier systems:

```jsonc
["i", "https://w3id.org/kim/schulfaecher/s1017"]
```

This declares: "this Nostr concept is a re-publication of the concept at that external URI." Systems that know the concept by its HTTP URI can use this to cross-reference.

## Referencing Vocabulary Concepts from Other Events

Other Nostr events can reference concepts defined by this NIP using standard `a` tags:

```jsonc
// In a kind:30142 AMB event, a kind:1 note, or any other event:
["a", "39737:<pubkey>:s1017", "<relay>"]
```

This makes the concept queryable: a relay filter `{"#a": ["39737:<pubkey>:s1017"]}` returns all events that reference that concept, across all event kinds.

## Authority Model

In Nostr, there is no domain authority. Instead, **the pubkey is the namespace**. Multiple pubkeys can publish events for the same vocabulary (same `d` tags), resulting in different versions.

Clients SHOULD use the user's web of trust to select which publisher's version of a vocabulary to display, similar to how [NIP-54](54.md) (Wiki) handles competing article versions.

A vocabulary publisher MAY signal their identity using a [NIP-05](05.md) identifier or by including provenance metadata in the concept scheme's `content` field.

## Querying

### Fetch a Specific Concept or Scheme

```json
{"kinds": [39737], "authors": ["<pubkey>"], "#d": ["s1017"]}
```

### Fetch All Concepts in a Scheme

Since concepts reference their scheme via `a` tags with the `inScheme` marker, and relays index `a` tags:

```json
{"kinds": [39737], "#a": ["39737:<pubkey>:schulfaecher"]}
```

This returns all concepts and collections that reference the scheme, plus any other events that reference it.

### Fetch All Concept Schemes by a Publisher

```json
{"kinds": [39737], "authors": ["<pubkey>"], "#type": ["ConceptScheme"]}
```

Note: this requires relay support for filtering on the `type` tag. Relays that do not index multi-letter tags may require client-side filtering.

This filter returns only published schemes. Clients that also want to show the viewing user's drafts can widen the filter:

```json
{"kinds": [39737, 39736], "authors": ["<pubkey>"], "#type": ["ConceptScheme"]}
```

### Find All Events Tagged with a Concept

```json
{"#a": ["39737:<pubkey>:s1017"]}
```

This works across all event kinds — AMB events, notes, labels, etc.

### Traversing Hierarchies

Clients can walk `broader`/`narrower` chains to compute transitive ancestry or descendant sets. To collect all narrower concepts under a given concept:

1. Fetch the concept and read its `narrower` tags
2. For each narrower concept, fetch it and read its `narrower` tags
3. Repeat recursively until no further `narrower` tags are found

SKOS defines `broaderTransitive` and `narrowerTransitive` as inferred properties. In this model, clients compute transitive closures by traversal rather than storing them as explicit tags.

## Examples

### Example 1: A Complete Small Vocabulary

**Concept Scheme:**

```json
{
  "kind": 39737,
  "pubkey": "abc123...",
  "tags": [
    ["d", "hcrt"],
    ["type", "ConceptScheme"],
    ["prefLabel", "Hochschulcampus Ressourcentypen", "de"],
    ["prefLabel", "Higher Education Resource Types", "en"],
    ["a", "39737:abc123...:text", "wss://relay.example.com", "hasTopConcept"],
    ["a", "39737:abc123...:audiovisual", "wss://relay.example.com", "hasTopConcept"],
    ["i", "https://w3id.org/kim/hcrt/scheme"]
  ],
  "content": "A controlled vocabulary of resource types for higher education."
}
```

**Top Concept:**

```json
{
  "kind": 39737,
  "pubkey": "abc123...",
  "tags": [
    ["d", "audiovisual"],
    ["type", "Concept"],
    ["prefLabel", "Audiovisuelles Medium", "de"],
    ["prefLabel", "Audiovisual Medium", "en"],
    ["a", "39737:abc123...:hcrt", "wss://relay.example.com", "inScheme"],
    ["a", "39737:abc123...:hcrt", "wss://relay.example.com", "topConceptOf"],
    ["a", "39737:abc123...:video", "wss://relay.example.com", "narrower"],
    ["a", "39737:abc123...:audio", "wss://relay.example.com", "narrower"],
    ["i", "https://w3id.org/kim/hcrt/audiovisual"]
  ],
  "content": ""
}
```

**Leaf Concept with External Mappings:**

```json
{
  "kind": 39737,
  "pubkey": "abc123...",
  "tags": [
    ["d", "video"],
    ["type", "Concept"],
    ["prefLabel", "Video", "de"],
    ["prefLabel", "Video", "en"],
    ["altLabel", "Film", "de"],
    ["altLabel", "Moving Image", "en"],
    ["notation", "video"],
    ["a", "39737:abc123...:hcrt", "wss://relay.example.com", "inScheme"],
    ["a", "39737:abc123...:audiovisual", "wss://relay.example.com", "broader"],
    ["r", "https://w3id.org/kim/hcrt/video", "exactMatch"],
    ["r", "http://purl.org/dc/dcmitype/MovingImage", "closeMatch"],
    ["i", "https://w3id.org/kim/hcrt/video"]
  ],
  "content": "A recording of moving visual images."
}
```

### Example 2: Cross-Scheme Mapping

A concept in one Nostr-native vocabulary mapped to a concept in another:

```json
{
  "kind": 39737,
  "pubkey": "abc123...",
  "tags": [
    ["d", "math"],
    ["type", "Concept"],
    ["prefLabel", "Mathematik", "de"],
    ["prefLabel", "Mathematics", "en"],
    ["a", "39737:abc123...:schulfaecher", "wss://relay.example.com", "inScheme"],
    ["a", "39737:def456...:mathematics", "wss://relay2.example.com", "exactMatch"]
  ],
  "content": ""
}
```

### Example 3: Using a Vocabulary Concept in an AMB Event

An educational resource referencing a Nostr-native vocabulary concept:

```json
{
  "kind": 30142,
  "tags": [
    ["d", "pythagorean-theorem-video"],
    ["name", "Pythagorean Theorem Explained"],
    ["a", "39737:abc123...:s1017", "wss://relay.example.com"],
    ["a", "39737:abc123...:video", "wss://relay.example.com"],
    ["t", "Pythagoras"],
    ["t", "Geometrie"]
  ],
  "content": "An introductory video explaining the Pythagorean theorem"
}
```

## References

- [SKOS Reference (W3C)](https://www.w3.org/TR/skos-reference/) — the vocabulary model this NIP is inspired by
- [SkoHub](https://skohub.io/) — SKOS vocabulary publishing infrastructure
- [NIP-01](01.md) — Basic protocol, addressable events
- [NIP-09](09.md) — Event deletion (draft-to-published cleanup)
- [NIP-23](23.md) — Long-form content (precedent for draft kind and `published_at` tag)
- [NIP-32](32.md) — Labeling
- [NIP-73](73.md) — External Content IDs (`i` tag)
- [NIP-54](54.md) — Wiki (web-of-trust authority model precedent)
- [NIP-AMB](AMB.md) — AMB metadata events (precedent for `a` tag markers)
