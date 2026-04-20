NIP-VOCAB
=========

Vocabularies
------------

`draft` `optional`

This NIP defines a system for publishing controlled vocabularies, taxonomies, thesauri, and classification schemes as Nostr events. It is inspired by [SKOS](https://www.w3.org/TR/skos-reference/) (Simple Knowledge Organization System) but designed natively for Nostr.

## Motivation

Nostr has labeling infrastructure ([NIP-32](32.md)) and domain-specific classification patterns ([NIP-AMB](AMB.md), [NIP-35](35.md)), but no general-purpose mechanism for publishing, discovering, and using shared controlled vocabularies on the protocol itself. Vocabulary definitions currently live on external web servers. This NIP brings them onto Nostr, making them decentralized, discoverable, and maintainable by anyone.

## Event Kinds

This NIP uses six addressable event kinds — one per entity type, with a parallel draft kind following NIP-23's pattern:

| Entity         | Published | Draft  |
|----------------|-----------|--------|
| ConceptScheme  | `39737`   | `39736` |
| Concept        | `39738`   | `39735` |
| Collection     | `39739`   | `39734` |

Distinct kinds let relays (which only index single-letter tags per NIP-01) return just the entity type a client asked for. The `["type", ...]` tag is retained for human readability and secondary validation but carries no query semantics.

Clients displaying *published* vocabularies MUST filter by published kinds and MUST NOT surface draft kinds as published.

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
    ["a", "39738:<pubkey>:s10", "<relay>", "hasTopConcept"],
    ["a", "39738:<pubkey>:s20", "<relay>", "hasTopConcept"],
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
  "kind": 39738,
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
    ["a", "39738:<pubkey>:s10", "<relay>", "broader"],
    ["a", "39738:<pubkey>:s101701", "<relay>", "narrower"],
    // associative relation
    ["a", "39738:<pubkey>:s1018", "<relay>", "related"],
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
  "kind": 39739,
  "tags": [
    ["d", "bachelor-programs"],
    ["type", "Collection"],
    ["prefLabel", "Bachelor Programs", "en"],
    ["prefLabel", "Bachelor-Studiengänge", "de"],
    ["a", "39737:<pubkey>:schulfaecher", "<relay>", "inScheme"],
    ["a", "39738:<pubkey>:civil-eng", "<relay>", "member"],
    ["a", "39738:<pubkey>:mech-eng", "<relay>", "member"]
  ],
  "content": ""
}
```

Collections are organizational only. Semantic relations (`broader`, `narrower`, `related`) MUST NOT be used on or target Collection events.

> **Note:** SKOS also defines `OrderedCollection` for collections with meaningful ordering. This is intentionally omitted from this NIP for simplicity and may be specified in a future extension.

## Drafts

Each published kind has a matching draft kind with identical tag structure:

- `kind:39736` — ConceptScheme draft
- `kind:39735` — Concept draft
- `kind:39734` — Collection draft

Drafts use the same `["d", ...]` identifier as their eventual published counterpart. Clients SHOULD treat drafts as private-to-the-author by default (e.g., publish only to the author's outbox relays). When publishing a draft as final, clients SHOULD delete the draft via NIP-09 and publish a new event under the corresponding published kind.

`published_at` remains OPTIONAL metadata on published events (for preserving first-publish time across edits) and is never a draft gate.

### Relay policy

Relays MAY accept or reject draft kinds per their policy. Public discovery relays MAY choose to reject drafts; private or author-scoped relays (e.g., a user's outbox) SHOULD accept them to support cross-device editing.

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
  "kind": 39738,
  "tags": [
    ["d", "video"],
    ["type", "Concept"],
    ["prefLabel", "Video", "de"],
    ["prefLabel", "Video", "en"],
    ["a", "39737:<pubkey>:hcrt", "<relay>", "inScheme"],
    // mapping to another publisher's vocabulary
    ["a", "39738:<other-pubkey>:moving-image", "<relay>", "exactMatch"]
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
["a", "39738:<pubkey>:s1017", "<relay>"]
```

This makes the concept queryable: a relay filter `{"#a": ["39738:<pubkey>:s1017"]}` returns all events that reference that concept, across all event kinds.

## Authority Model

In Nostr, there is no domain authority. Instead, **the pubkey is the namespace**. Multiple pubkeys can publish events for the same vocabulary (same `d` tags), resulting in different versions.

Clients SHOULD use the user's web of trust to select which publisher's version of a vocabulary to display, similar to how [NIP-54](54.md) (Wiki) handles competing article versions.

A vocabulary publisher MAY signal their identity using a [NIP-05](05.md) identifier or by including provenance metadata in the concept scheme's `content` field.

## Querying

### Fetch a Specific Scheme

```json
{"kinds": [39737], "authors": ["<pubkey>"], "#d": ["schulfaecher"]}
```

### Fetch a Specific Concept

```json
{"kinds": [39738], "authors": ["<pubkey>"], "#d": ["s1017"]}
```

### Fetch all ConceptSchemes from a publisher

```json
{ "kinds": [39737], "authors": ["<hex>"] }
```

### Fetch all Concepts belonging to a ConceptScheme

```json
{ "kinds": [39738], "#a": ["39737:<hex>:<d>"] }
```

### Fetch all Collections belonging to a ConceptScheme

```json
{ "kinds": [39739], "#a": ["39737:<hex>:<d>"] }
```

All filters use only single-letter tags (`a`) or the `kinds` field, so they work with any NIP-01-compliant relay. No multi-letter tag indexing is required.

Clients that also want to show the viewing user's drafts can widen the filter by adding the matching draft kind — for example, schemes plus draft schemes:

```json
{ "kinds": [39737, 39736], "authors": ["<hex>"] }
```

### Find All Events Tagged with a Concept

```json
{"#a": ["39738:<pubkey>:s1017"]}
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
    ["a", "39738:abc123...:text", "wss://relay.example.com", "hasTopConcept"],
    ["a", "39738:abc123...:audiovisual", "wss://relay.example.com", "hasTopConcept"],
    ["i", "https://w3id.org/kim/hcrt/scheme"]
  ],
  "content": "A controlled vocabulary of resource types for higher education."
}
```

**Top Concept:**

```json
{
  "kind": 39738,
  "pubkey": "abc123...",
  "tags": [
    ["d", "audiovisual"],
    ["type", "Concept"],
    ["prefLabel", "Audiovisuelles Medium", "de"],
    ["prefLabel", "Audiovisual Medium", "en"],
    ["a", "39737:abc123...:hcrt", "wss://relay.example.com", "inScheme"],
    ["a", "39737:abc123...:hcrt", "wss://relay.example.com", "topConceptOf"],
    ["a", "39738:abc123...:video", "wss://relay.example.com", "narrower"],
    ["a", "39738:abc123...:audio", "wss://relay.example.com", "narrower"],
    ["i", "https://w3id.org/kim/hcrt/audiovisual"]
  ],
  "content": ""
}
```

**Leaf Concept with External Mappings:**

```json
{
  "kind": 39738,
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
    ["a", "39738:abc123...:audiovisual", "wss://relay.example.com", "broader"],
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
  "kind": 39738,
  "pubkey": "abc123...",
  "tags": [
    ["d", "math"],
    ["type", "Concept"],
    ["prefLabel", "Mathematik", "de"],
    ["prefLabel", "Mathematics", "en"],
    ["a", "39737:abc123...:schulfaecher", "wss://relay.example.com", "inScheme"],
    ["a", "39738:def456...:mathematics", "wss://relay2.example.com", "exactMatch"]
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
    ["a", "39738:abc123...:s1017", "wss://relay.example.com"],
    ["a", "39738:abc123...:video", "wss://relay.example.com"],
    ["t", "Pythagoras"],
    ["t", "Geometrie"]
  ],
  "content": "An introductory video explaining the Pythagorean theorem"
}
```

## Migration from single-kind

Events published under the pre-split rule (all as `kind:39737`, disambiguated by the `type` tag) are not automatically valid under this revision. Publishers SHOULD republish Concept and Collection events under `kind:39738` / `kind:39739`. Clients MAY continue to read legacy events for backward compatibility but SHOULD treat any `kind:39737` event whose `type` tag is not `ConceptScheme` as legacy-only.

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
