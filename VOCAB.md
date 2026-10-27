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

Standalone mappings between concepts (see [Standalone Mappings](#standalone-mappings)) use three further addressable kinds, without draft counterparts:

| Entity          | Kind    |
|-----------------|---------|
| Mapping         | `39740` |
| Concordance     | `39741` |
| Mapping review  | `39742` |

*Published kinds increment (39737 → 39738 → 39739); draft kinds decrement (39736 → 39735 → 39734) so the `39737` / `39736` pair from an earlier revision of this NIP is preserved.*

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
| `i` | external URI | no | Bridge to an external URI identifier for this concept (see [The `i` tag](#the-i-tag)) |
| `definition` | text, language | no | Formal definition. Third element is a BCP47 language tag. The `content` field serves as the primary definition; `definition` tags provide additional translations |
| `scopeNote` | text, language | no | Intended usage scope. Third element is a BCP47 language tag |
| `example` | text, language | no | Usage example. Third element is a BCP47 language tag |
| `note` | text, language | no | Generic documentation note. Third element is a BCP47 language tag |

A concept MAY belong to multiple concept schemes by including multiple `inScheme` tags.

### The `i` tag

The `i` tag of a scheme or concept states the entity's **own** external identifier (e.g. `http://w3id.org/kim/schulfaecher/s1017`): the Nostr event is a republication of that concept, and the URI is its identifier everywhere else in this NIP. This is an identity statement about the event itself, not a reference to external content as in [NIP-73](73.md); the value is a plain URI (not a NIP-73 typed id such as `isbn:…`), and no `k` tag accompanies it. The single-letter tag is used deliberately so relays index it and `{"#i": ["<uri>"]}` finds the concept.

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

`a` tag references between vocabulary events SHOULD use the **published** kind prefix of the target even when the target currently exists only as a draft — i.e., a draft Concept whose `inScheme` points at a still-unpublished ConceptScheme SHOULD reference it as `39737:<pubkey>:<d>`, not `39736:<pubkey>:<d>`. This way references resolve automatically the moment the target is published. A client rendering a reference whose target has not yet been published MAY show it as an unresolved reference.

`published_at` remains OPTIONAL metadata on published events (for preserving first-publish time across edits) and is never a draft gate. Its value, when present, is a stringified unix timestamp in seconds — mirroring NIP-23's use of the same tag.

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

`exactMatch` MUST NOT be combined with `broadMatch`, `narrowMatch`, or `relatedMatch` for the same concept pair. `exactMatch` implies `closeMatch` — every exact match is inherently a close match. Clients querying for close matches SHOULD also include exact matches in their results.

`exactMatch` is transitive: if concept A is an exact match of B, and B is an exact match of C, then A is also an exact match of C. Clients resolving mappings SHOULD account for transitive chains.

For symmetric mapping relations (`exactMatch`, `closeMatch`, `relatedMatch`) embedded in Concept events, publishers SHOULD assert both directions. `narrowMatch` is the inverse of `broadMatch` — publishers SHOULD assert both directions: when a concept has a `broadMatch` tag, the target concept SHOULD include a corresponding `narrowMatch` tag pointing back, and vice versa. This rule applies to embedded tags only; [standalone mappings](#standalone-mappings) are stored in one direction and clients infer the inverse.

Embedded mapping tags can only be added by the concept's author. To link concepts of different publishers, or to publish a crosswalk as a third party, use [standalone mappings](#standalone-mappings).

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

Other Nostr events reference a concept by its [concept identifier](#concept-identifiers) and, when the concept is published on Nostr, SHOULD add a companion `a` tag carrying the address and a relay hint:

```jsonc
// In a kind:30142 AMB event (NIP-AMB), a NIP-DIDACTIC record, or any other event
// that carries flattened concept facets:
["about:id", "http://w3id.org/kim/schulfaecher/s1017"],   // concept with an i tag: its external URI
["about:prefLabel:de", "Mathematik"],
["a", "39738:<pubkey>:s1017", "<relay>", "about"],

["teaches:id", "nostr:naddr1…"],                           // concept without i tag: canonical Nostr URI
["teaches:prefLabel:de", "Wendepunkte in Jesus-Geschichten wahrnehmen"],
["a", "39738:<pubkey>:wendepunkte-wahrnehmen", "<relay>", "teaches"],

// In a kind:1 note or any event without flattened facets:
["a", "39738:<pubkey>:s1017", "<relay>"]
```

The fourth element of the companion `a` tag is the flattened tag prefix the concept appears under (`about`, `teaches`, `ext:<ns>:<facet>`, …) — the same convention [NIP-AMB](AMB.md) uses for `isPartOf`/`hasPart` and [NIP-DIDACTIC](DIDACTIC.md) uses for its facets. Events without flattened facets omit it. The identifier is authoritative; the `a` tag is an index and a relay hint, and for a `nostr:naddr…` identifier it is fully derivable from the identifier. Consumers without a vocab-aware index can ignore the `a` tag.

This makes the concept queryable: a relay filter `{"#a": ["39738:<pubkey>:s1017"]}` returns all events that reference that concept, across all event kinds.

## Standalone Mappings

Mapping relations embedded in Concept events (see [Mapping Relations (Nostr-to-Nostr)](#mapping-relations-nostr-to-nostr)) can only be asserted by the concept's own author, and every added mapping rewrites the concept event. This section defines three further addressable kinds — Mapping, Concordance, and Mapping review — for mappings that link concepts across publishers or that a third party (a curator or institution) publishes as a crosswalk, without touching the concept events themselves. The model follows [JSKOS](https://gbv.github.io/jskos/), the data format used by Cocoda / coli-conc, and the [TS4NFDI Mapping Sameness Identifier](https://ts4nfdi.github.io/mapping-sameness-identifier/) for content-addressing mappings.

### Concept Identifiers

Every mapping refers to concepts and schemes by a **concept identifier** string, which is always a URI:

- the concept's external URI (the value of its `i` tag), if it has one;
- otherwise the concept's **canonical Nostr URI** — `nostr:` followed by the [NIP-19](19.md) `naddr` encoding of its address, per [NIP-21](21.md), e.g. `nostr:naddr1qqpn2tfkqgsdy6y79aqa40a6j576yej449xw92jwq2wrs0hfy8r2fh404wv6vysrqsqqpxe6vja26r` (see [Canonical Nostr URI](#canonical-nostr-uri)).

The same rule applies to schemes (the scheme's `i` tag, else its canonical Nostr URI). If an event carries several `i` tags, the identifier is the value of the **first** one (matching the reference library, whose `vocabIdentifier` uses the first `i` tag it finds). The same identifiers are used wherever another event refers to a concept, e.g. in the flattened `<facet>:id` tags of [NIP-AMB](AMB.md) and [NIP-DIDACTIC](DIDACTIC.md) events (see [Referencing Vocabulary Concepts from Other Events](#referencing-vocabulary-concepts-from-other-events)).

#### Canonical Nostr URI

The canonical Nostr URI of a vocabulary event is `nostr:<naddr>`, where `<naddr>` is the NIP-19 `naddr` encoding of exactly these TLV entries, in this order:

- type `0` — the event's `d` tag value, byte for byte as it appears in the event. It is **not** normalised: addressability is by the exact `d` value ([NIP-01](01.md)), and the naddr MUST decode back to the address the relay knows;
- type `2` — the author pubkey;
- type `3` — the **published** kind (39738 Concept, 39737 ConceptScheme, 39739 Collection), even when the event being identified is itself a draft, and even when an `a` tag being converted points at a draft-kind address (e.g. `39735:<pubkey>:<d>` identifies as the naddr of `39738:<pubkey>:<d>`).

No type `1` (relay) entries are included, and the bech32 string is lowercase. Identifiers therefore never carry relay hints, event ids, labels, or timestamps, and label, definition, notation, and hierarchy edits never change an identifier. Relay hints belong in the companion `a` tags of a reference, not in the identifier.

NIP-19 does not mandate TLV order, so a `nostr:naddr…` string received from elsewhere may differ from the canonical string while naming the same event. Consumers MUST NOT compare Nostr URIs as received strings: they MUST decode them and compare the triple (published kind, pubkey, exact `d`), ignoring relay entries — or re-encode both sides canonically first. The canonical string matters in exactly one place: the [sameness identifier](#sameness-identifier) hashes identifier strings, so producers MUST write canonical Nostr URIs into mapping events. External-URI identifiers MUST be compared as exact strings after NFC normalisation.

**Why a bech32 URI inside an event.** [NIP-19](19.md) describes bech32 entities as display and sharing formats rather than core-protocol values, and protocol-level references in this NIP accordingly stay in hex: every reference to a vocabulary event that a relay has to index is an `a` tag with a `<kind>:<pubkey>:<d>` coordinate (and the relay hint). The canonical Nostr URI is an *application-level identifier value*: AMB requires concept ids to be URIs, [NIP-21](21.md) defines the only `nostr:` URI form, and [NIP-27](27.md) already establishes `nostr:` URIs inside events (in `content`). It therefore appears only where a URI is required — in `<facet>:id` tags of [NIP-AMB](AMB.md)/[NIP-DIDACTIC](DIDACTIC.md) events, in mapping `from`/`to`/`fromScheme`/`toScheme` tags, and in AMB JSON — always next to the companion `a` tag that carries the protocol-level reference.

Publishers SHOULD choose NFC-normalised `d` values when creating concepts and schemes, and MUST keep them at most 255 bytes of UTF-8: a NIP-19 TLV length is a single byte, so a longer `d` has no naddr and therefore no canonical Nostr URI (consumers encountering such an event MUST NOT throw; they SHOULD treat the raw address as unresolvable). Implementations MUST use a bech32 codec without the 90-character BIP-173 limit (NIP-19 allows up to 5000 characters).

**Identifiers are immutable.** Once a concept or scheme is published:

- its `i` tag MUST NOT be added, changed, or removed — a concept published without `i` keeps its `nostr:` identifier forever;
- its `d` tag and author are fixed by addressability.

A concept that needs a different identifier MUST be published as a new concept; the old concept MAY be linked to it with `exactMatch`. Publishing tools MUST refuse edits that would add, change, or remove the `i` tag of a published concept, and SHOULD warn when a scheme is published without URIs if it is expected to be referenced from other events.

> **Legacy form.** Earlier revisions of this NIP identified concepts without an `i` tag as `nostr:<kind>:<pubkey-hex>:<d>`. That form is not a NIP-21 URI and is not even always a valid URI (a `d` value may contain characters that are illegal in URIs). Producers MUST NOT emit it. Consumers SHOULD accept it on input and canonicalise it: parse the address, map a draft kind to its published kind, and encode the canonical Nostr URI. Mapping events published with legacy identifiers stay valid under the `d` rule (the hash is computed over the tag values as written) but do not aggregate with canonical ones; their publishers SHOULD republish them.

### Mapping — `kind:39740` (addressable)

```jsonc
{
  "kind": 39740,
  "content": "optional free-text note",
  "tags": [
    ["d", "mapping:<sha256 hex>"],
    ["relation", "exactMatch"],
    ["from", "nostr:naddr1…"],               // canonical Nostr URI of 39738:<pubkey>:5-6 (klassenstufen concepts have no i-tag URI)
    ["to", "https://w3id.org/kim/educationalLevel/level_2"],
    ["fromScheme", "nostr:naddr1…"],         // canonical Nostr URI of 39737:<pubkey>:klassenstufen
    ["toScheme", "https://w3id.org/kim/educationalLevel/"],

    // index tags (for relay filtering only; carry no semantics)
    ["a", "39738:<pubkey>:5-6", "wss://relay.edufeed.org"],
    ["a", "39738:<pubkey>:level_2", "wss://relay.edufeed.org"],
    ["a", "39737:<pubkey>:klassenstufen"],
    ["a", "39737:<pubkey>:educational-level"],
    ["i", "https://w3id.org/kim/educationalLevel/level_2"],

    // optional
    ["a", "39741:<author>:klassenstufen--educational-level", "", "partOf"],
    ["relevance", "0.8"]
  ]
}
```

#### Mapping Tags

| Tag | Card. | Meaning |
|---|---|---|
| `d` | 1 | Sameness identifier (see [Sameness Identifier](#sameness-identifier)) |
| `relation` | 1 | One of `exactMatch`, `closeMatch`, `broadMatch`, `narrowMatch`, `relatedMatch` |
| `from` | exactly 1 | Source concept identifier |
| `to` | 1..n | Target concept identifiers; several = AND combination (JSKOS `memberSet`) |
| `fromScheme`, `toScheme` | 1 each | Scheme identifiers |
| `a` (no marker) | 0..n | Index: address of every referenced concept/scheme that exists as a Nostr event. SHOULD be present for each such concept/scheme |
| `i` | 0..n | Index: every referenced concept identifier that is an external URI (not a `nostr:` URI; Nostr-published concepts are indexed by `a`). SHOULD be present |
| `a` … `partOf` | 0..1 | Concordance membership (see [Concordance](#concordance)) |
| `relevance` | 0..1 | Decimal 0..1, JSKOS `mappingRelevance` |
| content | — | Free-text note (JSKOS `note`) |

Rules:

- `to` MUST be non-empty. Null mappings ("no equivalent in toScheme") are reserved for a future revision.
- OR alternatives are expressed as separate mappings (JSKOS `memberChoice` is not supported).
- `from` and `to` concepts SHOULD be in different schemes.
- Relays can find all mappings touching a concept with `{"kinds":[39740],"#a":["39738:<pubkey>:<d>"]}` or `{"kinds":[39740],"#i":["<uri>"]}`.
- Clients MUST ignore mapping events whose `d` does not equal the sameness identifier computed from their `from`, `relation` and `to` tags.

#### Sameness Identifier

`d` is the [TS4NFDI Mapping Sameness Identifier](https://ts4nfdi.github.io/mapping-sameness-identifier/), affirmative form:

1. subjects = [`from`], objects = `to` values, each NFC-normalised (Nostr-published concepts enter as their canonical Nostr URI, see above); duplicate values MUST be removed before sorting (subjects and objects are sets);
2. sort subjects and objects by Unicode code point, join each with `|`;
3. predicate = the full SKOS IRI, `http://www.w3.org/2004/02/skos/core#<relation>`;
4. string = `<subjects> <predicate> <objects>` (single spaces), UTF-8;
5. SHA-256, lowercase hex; `d` = `mapping:<hex>`.

The following are test vectors using canonical Nostr URIs. The two Nostr-published concepts are (pubkey `d2689e2f41dabfba953da26655a94ce2aa4e029c383ee921c6a4deafab99a612`):

| address | canonical Nostr URI |
|---|---|
| `39738:d2689e2f…a612:5-6` | `nostr:naddr1qqpn2tfkqgsdy6y79aqa40a6j576yej449xw92jwq2wrs0hfy8r2fh404wv6vysrqsqqpxe6vja26r` |
| `39738:d2689e2f…a612:ubung-lernkontrolle` | `nostr:naddr1qqfh2cn4denj6mr9wfhxkmmww3ex7mrvv5pzp5ngnch5rk4lh22nmgnx2k55ec42fcpfcwp7aysudfx7474enfsjqvzqqqym8g03gpqh` |

| from | relation | to | `d` |
|---|---|---|---|
| `nostr:naddr1qqpn2tfkqgsdy6y79aqa40a6j576yej449xw92jwq2wrs0hfy8r2fh404wv6vysrqsqqpxe6vja26r` | `exactMatch` | `https://w3id.org/kim/educationalLevel/level_2` | `mapping:f5d7fbaec398fa339b403145f100af6c30cb5c6d45bf97f93024b26a4578679c` |
| `https://edufeed.org/ns/ekw#lrt/arbeitsblatt` | `exactMatch` | `https://w3id.org/kim/hcrt/worksheet` | `mapping:3ee871bd4f4a105564793c8ff1476f8e72685eb5d8c6b6e3188e3ec95752c75c` |
| `nostr:naddr1qqfh2cn4denj6mr9wfhxkmmww3ex7mrvv5pzp5ngnch5rk4lh22nmgnx2k55ec42fcpfcwp7aysudfx7474enfsjqvzqqqym8g03gpqh` | `narrowMatch` | `https://w3id.org/kim/hcrt/assessment`, `https://w3id.org/kim/hcrt/drill_and_practice` (any order; duplicates don't matter) | `mapping:42154157f823a212feb8995c776f0157abfce74d42e57b973022910a4ea8c7d1` |

Both naddr strings MUST be reproduced byte for byte by a conforming encoder (they were cross-checked with go-nostr and nostr-tools).

Implementations MUST reproduce the two published [TS4NFDI](https://ts4nfdi.github.io/mapping-sameness-identifier/) test vectors (the negative one by implementing the `~` suffix internally, even though negative mappings are not published in this revision), plus the test vectors above.

Consequences:

- The same assertion by different authors produces the same `d`, so agreement and reviews aggregate across authors.
- An author republishing the same content (unchanged `from`/`relation`/`to`) replaces their previous event, e.g. to edit the note, `relevance`, or `partOf`.
- Changing the relation or the members changes `d`, producing a new event; the author SHOULD delete the old one via NIP-09.

#### Direction

Only one direction is stored. Clients resolving mappings MUST infer the inverse: `exactMatch`↔`exactMatch`, `closeMatch`↔`closeMatch`, `relatedMatch`↔`relatedMatch`, `broadMatch`↔`narrowMatch`. Inverses are only inferable for 1:1 mappings; a 1:n mapping is only readable in its stored direction.

The rule "publishers SHOULD assert both directions" (see [Mapping Relations (Nostr-to-Nostr)](#mapping-relations-nostr-to-nostr)) applies to **embedded** mapping tags only; for standalone mappings, publishers SHOULD NOT publish the inverse as a separate event.

### Concordance — `kind:39741` (addressable)

```jsonc
{
  "kind": 39741,
  "content": "Crosswalk der EKW-Klassenstufen auf KIM-Bildungsstufen.",
  "tags": [
    ["d", "klassenstufen--educational-level"],
    ["title", "Klassenstufen ↔ Bildungsstufe", "de"],
    ["fromScheme", "nostr:naddr1…"],         // canonical Nostr URI of 39737:<pubkey>:klassenstufen
    ["toScheme", "https://w3id.org/kim/educationalLevel/"],
    ["a", "39737:<pubkey>:klassenstufen"],
    ["a", "39737:<pubkey>:educational-level"],
    ["p", "<pubkey>", "", "maintainer"],
    ["license", "CC0-1.0"]
  ]
}
```

#### Concordance Tags

| Tag | Card. | Meaning |
|---|---|---|
| `d` | 1 | Author-chosen slug |
| `title` | 1..n | Title, BCP47 language as 3rd element, at most one per language |
| `fromScheme`, `toScheme` | 1 each | Scheme identifiers |
| `a` (no marker) | 0..2 | Index: scheme addresses, where schemes exist on Nostr |
| `p` … `maintainer` | 0..n | Additional pubkeys allowed to add members |
| `license` | 0..1 | SPDX identifier |
| content | — | Description |

Rules:

- A concordance does not list its mappings. Its members are mappings with a `partOf` tag pointing at it **whose author is the concordance author or a listed maintainer**. `partOf` from anyone else MUST be ignored. Clients fetch members with `{"kinds":[39740],"#a":["39741:<owner>:<d>"],"authors":["<owner>", "<maintainer>", …]}`.
- A member's `fromScheme`/`toScheme` MUST equal the concordance's; clients MUST ignore non-matching members.
- To remove a member, the maintainer MUST delete the mapping (NIP-09) or republish it without `partOf`.
- A mapping without `partOf` is a valid personal assertion; it MAY appear when browsing a concept but MUST NOT appear in trust-restricted consumers.
- Concordances are the unit of trust for consumers (see [Resolving Mappings](#resolving-mappings)).

### Mapping Review — `kind:39742` (addressable)

```jsonc
{
  "kind": 39742,
  "content": "Klasse 5-6 umfasst in manchen Ländern noch die Primarstufe.",
  "tags": [
    ["d", "mapping:<sha256 hex>"],
    ["vote", "-1"],
    ["mismatch", "https://uri.gbv.de/terminology/mismatch/inexact"],
    ["a", "39740:<mapping-author>:mapping:<sha256 hex>"],
    ["a", "39741:<owner>:klassenstufen--educational-level"]
  ]
}
```

#### Mapping Review Tags

| Tag | Card. | Meaning |
|---|---|---|
| `d` | 1 | Sameness identifier of the reviewed mapping content |
| `vote` | 1 | `+1` or `-1` |
| `mismatch` | 0..n | Only with `-1`: reason URI from `https://uri.gbv.de/terminology/mismatch/` |
| `a` | 0..n | Optional context: the specific mapping event seen, the concordance |
| content | — | Optional comment |

Rules:

- There MUST be at most one current vote per reviewer per mapping content; a reviewer republishing a review for the same `d` replaces their previous vote.
- All reviews of a mapping content can be fetched with `{"kinds":[39742],"#d":["mapping:<hex>"]}`.
- There is no separate "confirmed" event. A mapping is **confirmed within a concordance** when the concordance owner or a maintainer has published a mapping with the same `d` and `partOf` pointing at that concordance. Adopting someone else's mapping means republishing the same content under one's own key.
- Web-of-trust weighting and aggregating votes into `relevance` are out of scope for this revision; clients SHOULD show the raw vote sum and who voted.

### Resolving Mappings

Client resolution SHOULD proceed as follows:

1. **Normalise** both sources into one model `{id, from, to[], relation, fromScheme, toScheme, author, partOf?, origin}`: embedded `a` tags with mapping markers and embedded `r` tags with mapping markers on Concept events (author = concept author, `origin: "embedded"`), and standalone `kind:39740` events (`origin: "standalone"`). Converting an embedded `a`-tag target into a concept identifier (see [Concept Identifiers](#concept-identifiers)) needs the target concept event, for its `i` tag; clients SHOULD fall back to the canonical Nostr URI, which is computable from the address alone, when the target is not available. Embedded mappings' `id` SHOULD be computed with the same sameness algorithm as standalone mappings; entries with an equal `id` SHOULD be merged into one mapping with multiple sources.
2. **Inverses** SHOULD be computed per [Direction](#direction), so a lookup from either side finds a mapping.
3. The **trust set** is a parameter of resolution: `{concordances: ParsedConcordance[], authors?: pubkey[], includeEmbedded: boolean}`, or `'all'`. Clients SHOULD pass parsed concordance objects (carrying author, maintainers, and schemes), not just concordance addresses, since membership requires them. Editors SHOULD pass "everything" and group results by source; search consumers SHOULD pass their configured concordances.
4. **Transitivity**: clients SHOULD chain only `exactMatch`, only within the trust set, at most 3 hops. `closeMatch`, `broadMatch`, `narrowMatch`, and `relatedMatch` MUST NOT be chained in this revision.
5. **Conflicts**: an `exactMatch` together with a `broadMatch`, `narrowMatch`, or `relatedMatch` for the same concept pair — including pairs where the `exactMatch` is reached only transitively — SHOULD be reported as a conflict. A conflicting target inside the trust set MUST get no `exactMatch` result, direct or transitive, and exact chains MUST NOT be expanded through it.

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

An educational resource referencing two Nostr-published vocabulary concepts — one that carries an external URI in its `i` tag, one that is identified by its canonical Nostr URI:

```json
{
  "kind": 30142,
  "tags": [
    ["d", "pythagorean-theorem-video"],
    ["name", "Pythagorean Theorem Explained"],
    ["about:id", "http://w3id.org/kim/schulfaecher/s1017"],
    ["about:prefLabel:de", "Mathematik"],
    ["a", "39738:abc123...:s1017", "wss://relay.example.com", "about"],
    ["learningResourceType:id", "nostr:naddr1..."],
    ["learningResourceType:prefLabel:de", "Erklärvideo"],
    ["a", "39738:abc123...:erklaervideo", "wss://relay.example.com", "learningResourceType"],
    ["t", "Pythagoras"],
    ["t", "Geometrie"]
  ],
  "content": "An introductory video explaining the Pythagorean theorem"
}
```

The `a` tags make both concepts queryable by `#a` on the relay; the `:id` tags carry the identifiers that AMB consumers read.

## Migration from single-kind

Events published under the pre-split rule (all as `kind:39737`, disambiguated by the `type` tag) are not automatically valid under this revision. Publishers SHOULD republish Concept and Collection events under `kind:39738` / `kind:39739`. Clients MAY continue to read legacy events for backward compatibility but SHOULD treat any `kind:39737` event whose `type` tag is not `ConceptScheme` as legacy-only.

### Impact on existing references

Any event in the wider Nostr ecosystem (for example, NIP-AMB learning-resource events) that references a concept via an `a` tag with the legacy `39737:<pubkey>:<d>` coordinate becomes stale once that concept is republished under `kind:39738`: the reference's kind prefix no longer matches the target's kind. Clients resolving such references SHOULD attempt a fallback lookup using `kind:39738` with the same `pubkey:d` suffix before treating the reference as unresolved.

Publishers migrating their own vocabularies SHOULD:

1. Republish each Concept event under `kind:39738` and each Collection event under `kind:39739`, preserving the original `d` identifier.
2. NIP-09-delete the legacy `kind:39737` Concept / Collection events so relays can garbage-collect them.
3. Optionally announce the migration (e.g., via a kind:1 note) so downstream consumers know to update their cached references.

During a transition window, clients MAY dual-read both the legacy and the new kinds to minimize broken references.

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
- [JSKOS data format](https://gbv.github.io/jskos/) — mapping and concordance model
- [TS4NFDI Mapping Sameness Identifier](https://ts4nfdi.github.io/mapping-sameness-identifier/)
