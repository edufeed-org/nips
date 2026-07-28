# NIP-AMB

## Abstract

This NIP defines how to handle the metadata profile ["Allgemeines Metadatenprofil für Bildungsressourcen" (AMB)](https://dini-ag-kim.github.io/amb/latest/) in nostr:

- How to convert AMB metadata to an AMB nostr event
- How to convert an AMB nostr-event to AMB metadata
- How to query for AMB nostr-events in supporting relays

## Event Kind

This NIP defines `kind:30142` as an AMB Metadata Event.
This means this is an addressable event, that can be addressed using `kind:pubkey:d-tag`.

## How to convert AMB metadata *to* an AMB nostr event

The transformation uses JSON-flattening with `:` as the delimiter to convert nested AMB metadata structures into flat Nostr tags. Additionally, Nostr-native tag conventions are used where applicable for better interoperability and query efficiency.

### Nostr-Native Conventions

This NIP follows Nostr conventions where they align with AMB requirements:

- **`d` tag**: Used as the unique identifier for the AMB resource (maps to AMB `id`)
- **`t` tags**: Used for keywords/topics (instead of flattened `keywords` tags)
- **`p` tags**: Used for creator/contributor references when the person has a Nostr identity (pubkey). Format: `["p", <pubkey-hex>, <relay-hint>, <role>]` where `<role>` is `"creator"` or `"contributor"`. The relay hint is a single suggestion for discovery (per NIP-01 convention); clients SHOULD use NIP-65 for full relay resolution. When a `p` tag is used for a person, no `creator:*`/`contributor:*` flattened tags are emitted for that person — the pubkey IS their identity. Persons without a Nostr identity use the flattened `creator:*`/`contributor:*` tag structure instead.
- **`a` tags**: Used for references to other addressable events on Nostr (including other AMB events), with fallback to flattened URIs for external resources. Format: `["a", "30142:<pubkey>:<d-value>", <relay-hint>, <relationship>]`
- **`r` tags**: Used for external URL references (original source, DOI, related web resources)
- **`content` field**: SHOULD contain the description text for client compatibility; the `description` tag is kept for relay queryability

### Flattening Rules

1. **Simple properties**: Map directly to `["<key>", "<value>"]` tags
   - AMB: `{"name": "Resource Title"}`
   - Nostr: `["name", "Resource Title"]`

2. **Nested objects**: Flatten using `:` delimiter
   - AMB: `{"creator": {"name": "John", "id": "123"}}`
   - Nostr: `["creator:name", "John"]`, `["creator:id", "123"]`

3. **Arrays**: Repeat the same flattened tag key (order is preserved by tag array position)
   - AMB: `{"keywords": ["Math", "Physics"]}`
   - Nostr: `["t", "Math"]`, `["t", "Physics"]` (Nostr-native `t` tag)

4. **Arrays of objects**: Repeat flattened keys for each object
   - AMB: `{"creator": [{"name": "John"}, {"name": "Jane"}]}`
   - Nostr: `["creator:name", "John"]`, `["creator:name", "Jane"]`

5. **Deep nesting**: Continue flattening with additional `:` delimiters
   - AMB: `{"creator": {"affiliation": {"name": "MIT"}}}`
   - Nostr: `["creator:affiliation:name", "MIT"]`

### Property Mappings

This is how we convert each property of the AMB:

#### General:

- `id` → `["d", <id>]` (special case: use Nostr's `d` tag as identifier). The `d` value SHOULD be the resource's canonical, dereferenceable URL when one exists (the AMB spec describes `id` as a dereferenceable HTTP URI). Nostr-native resources without an external URL MAY use an arbitrary stable slug — see the reverse-conversion rules for how the AMB `id` is derived in that case.
- `type` → `["type", <value>]` (repeat for multiple types)
- `name` → `["name", <value>]`
- `description` → `["description", <value>]` AND `"content": <value>` (duplicated for client compatibility and relay queryability)
- `about` (array of concept objects) → Repeat for each:
  - `["about:id", <uri>]`
  - `["about:prefLabel:lang", <label>]`
  - `["about:type", "Concept"]`
- `keywords` → `["t", <keyword>]` (repeat for each keyword, using Nostr `t` tag)
- `inLanguage` → `["inLanguage", <languageCode>]` (repeat for each language)
- `image` → `["image", <uri>]`
- `trailer` (MediaObject) →
  - `["trailer:contentUrl", <url>]`
  - `["trailer:type", <"VideoObject"|"AudioObject">]`
  - `["trailer:encodingFormat", <format>]` (optional)
  - `["trailer:contentSize", <bytes>]` (optional)
  - `["trailer:sha256", <hash>]` (optional)
  - `["trailer:embedUrl", <url>]` (optional)
  - `["trailer:bitrate", <kbps>]` (optional)

#### Provenance:

- `creator` (array of Person/Organization objects) → For each creator, use **one** of the following (never both for the same person):
  - **Nostr-native (creator has a Nostr pubkey)**: `["p", <pubkey-hex>, <relay-hint>, "creator"]` — no additional `creator:*` tags for this person. Their name and metadata are resolved from their kind:0 profile.
    - **Detection**: If a creator object's `id` is a `nostr:` URI per [NIP-21](https://github.com/nostr-protocol/nips/blob/master/21.md) encoding an `npub` or `nprofile`, converters MUST decode it and emit the `p` tag form instead of flattened `creator:*` tags. (This is valid AMB input: the AMB schema constrains `creator.id` only to `format: uri`; the ORCID/GND/Wikidata/ROR list is a SHOULD-level recommendation.)
    - **Relay hint precedence**: relay embedded in the `nprofile` (first entry) → converter-configured default → empty string.
  - **External (no Nostr identity)**:
    - `["creator:id", <uri>]` (optional, e.g., ORCID, GND)
    - `["creator:name", <name>]`
    - `["creator:type", <"Person"|"Organization">]`
    - `["creator:honorificPrefix", <title>]` (optional, for persons)
    - `["creator:affiliation:id", <uri>]` (optional)
    - `["creator:affiliation:name", <name>]` (optional)
    - `["creator:affiliation:type", "Organization"]` (optional)
- `contributor` (array of Person/Organization objects) → Same structure as `creator` (including `nostr:` URI detection), using role `"contributor"` in the `p` tag

  > **Known limitation:** A person with both a Nostr identity and external identifiers (e.g., an ORCID) is represented by the `p` tag alone; their external identifier, `affiliation`, and `honorificPrefix` are not carried in the event, since kind:0 profiles have no standard fields for them. This deviates from the AMB SHOULD-level recommendation to reference ORCID/GND/Wikidata/ROR and is accepted as a trade-off for having exactly one unambiguous representation per person.

- `dateCreated` → `["dateCreated", <ISO8601Date>]`
- `datePublished` → `["datePublished", <ISO8601Date>]`
- `dateModified` → `["dateModified", <ISO8601Date>]`
- `publisher` (array of Organization/Person objects) → Repeat for each:
  - `["publisher:id", <uri>]` (optional)
  - `["publisher:name", <name>]`
  - `["publisher:type", <"Organization"|"Person">]`
- `funder` (array of Person/Organization/FundingScheme objects) → Repeat for each:
  - `["funder:id", <uri>]` (optional)
  - `["funder:name", <name>]`
  - `["funder:type", <"Person"|"Organization"|"FundingScheme">]`

> **Note:** No `p`-tag role is defined for `publisher` or `funder`. A `nostr:` URI in their `id` is emitted verbatim as the flattened `publisher:id`/`funder:id` value.

#### Costs and Rights:

- `isAccessibleForFree` → `["isAccessibleForFree", <"true"|"false">]`
- `license` (object) →
  - `["license:id", <license_uri>]`
- `conditionsOfAccess` (Concept object) →
  - `["conditionsOfAccess:id", <uri>]`
  - `["conditionsOfAccess:prefLabel:lang", <label>]` (optional)
  - `["conditionsOfAccess:type", "Concept"]` (optional)

#### Educational:

- `learningResourceType` (array of Concept objects) → Repeat for each:
  - `["learningResourceType:id", <uri>]`
  - `["learningResourceType:prefLabel:lang", <label>]` (optional)
  - `["learningResourceType:type", "Concept"]` (optional)
- `audience` (array of Concept objects) → Repeat for each:
  - `["audience:id", <uri>]`
  - `["audience:prefLabel:lang", <label>]` (optional)
  - `["audience:type", "Concept"]` (optional)
- `teaches` (array of Concept objects) → Repeat for each:
  - `["teaches:id", <uri>]`
  - `["teaches:prefLabel:lang", <label>]` (optional)
- `assesses` (array of Concept objects) → Repeat for each:
  - `["assesses:id", <uri>]`
  - `["assesses:prefLabel:lang", <label>]` (optional)
- `competencyRequired` (array of Concept objects) → Repeat for each:
  - `["competencyRequired:id", <uri>]`
  - `["competencyRequired:prefLabel:lang", <label>]` (optional)
- `educationalLevel` (array of Concept objects) → Repeat for each:
  - `["educationalLevel:id", <uri>]`
  - `["educationalLevel:prefLabel:lang", <label>]` (optional)
  - `["educationalLevel:type", "Concept"]` (optional)
- `interactivityType` (Concept object) →
  - `["interactivityType:id", <uri>]`
  - `["interactivityType:prefLabel:lang", <label>]` (optional)
  - `["interactivityType:type", "Concept"]` (optional)
- `suggestedAge` (object with integer bounds; AMB requires at least one of `minValue`/`maxValue`) →
  - `["suggestedAge:minValue", <integer>]` (optional)
  - `["suggestedAge:maxValue", <integer>]` (optional)

#### Relations:

- `isBasedOn` (array of objects) → Repeat for each:
  - **Nostr-native (if referenced resource is addressable AMB event)**: `["a", "30142:<pubkey>:<d-value>", <relay>, "isBasedOn"]`
  - **Fallback (for external URIs)**:
    - `["isBasedOn:id", <uri>]` (omit when the relation has no `id` — AMB allows name-only `isBasedOn` references; never emit a literal `"undefined"`)
    - `["isBasedOn:name", <name>]` (optional)
- `isPartOf` (array of objects) → Repeat for each:
  - **Nostr-native (if referenced resource is addressable AMB event)**: `["a", "30142:<pubkey>:<d-value>", <relay>, "isPartOf"]`
  - **Fallback (for external URIs)**:
    - `["isPartOf:id", <uri>]`
    - `["isPartOf:name", <name>]` (optional)
    - `["isPartOf:type", <type>]` (optional)
- `hasPart` (array of objects) → Repeat for each:
  - **Nostr-native (if referenced resource is addressable AMB event)**: `["a", "30142:<pubkey>:<d-value>", <relay>, "hasPart"]`
  - **Fallback (for external URIs)**:
    - `["hasPart:id", <uri>]`
    - `["hasPart:name", <name>]` (optional)
    - `["hasPart:type", <type>]` (optional)

#### Meta-Metadata:

- `mainEntityOfPage` (array of WebPage objects) → Repeat for each:
  - `["mainEntityOfPage:id", <uri>]`
  - `["mainEntityOfPage:type", "WebContent"]`
  - `["mainEntityOfPage:provider:id", <uri>]` (optional)
  - `["mainEntityOfPage:provider:name", <name>]` (optional)
  - `["mainEntityOfPage:provider:type", <type>]` (optional)
  - `["mainEntityOfPage:dateCreated", <ISO8601Date>]` (optional)
  - `["mainEntityOfPage:dateModified", <ISO8601Date>]` (optional)

#### Technical:

- `duration` → `["duration", <ISO8601Duration>]` (format: PnYnMnDTnHnMnS)
- `encoding` (array of MediaObject objects) → Repeat for each:
  - `["encoding:type", "MediaObject"]`
  - `["encoding:contentUrl", <url>]` (or use `embedUrl`)
  - `["encoding:embedUrl", <url>]` (or use `contentUrl`)
  - `["encoding:encodingFormat", <format>]` (optional, IANA media type)
  - `["encoding:contentSize", <bytes>]` (optional)
  - `["encoding:sha256", <hash>]` (optional)
  - `["encoding:bitrate", <kbps>]` (optional)
- `caption` (array of MediaObject objects) → Repeat for each:
  - `["caption:id", <uri>]`
  - `["caption:type", "MediaObject"]`
  - `["caption:encodingFormat", <format>]` (optional, IANA media type)
  - `["caption:inLanguage", <languageCode>]` (optional)

#### External References:

Supplementary "see also" references use the Nostr-native `r` tag (per NIP-24). These are Nostr-native metadata for client interoperability and do not map to a specific AMB property on reverse conversion.

- `["r", <url>]` - Repeat for each external reference

Examples:
- `["r", "https://oersi.org/resources/xyz"]` - Original source URL
- `["r", "https://doi.org/10.1234/example"]` - DOI reference
- `["r", "urn:isbn:978-3-16-148410-0"]` - ISBN reference

#### Extension Properties (ext namespace):

Properties not standardized in AMB-core SHOULD use the `ext` namespace. The shape mirrors AMB-core's flattening, with one extra leading segment that identifies the publishing authority. This enables non-AMB-conformant metadata to coexist with AMB-core in a single event without collision risk.

- Tag form: `["ext:<ns>:<facet>", "<value>"]` (scalar) or `["ext:<ns>:<facet>:<sub>", "<value>"]` (structured)
  - `<ns>` — namespace authority slug. MUST NOT contain `:`. Lowercase; `.` and `-` are permitted. Stable per authority. Examples: `ekw`, `oersi`, `org.edufeed.ekw`.
  - `<facet>` — field name within the namespace. MUST NOT contain `:`. Examples: `bistum`, `ressourcentyp`, `fach`.
  - `<sub>` — property suffix, identical to AMB-core, drawn from a **closed set**: `id`, `type`, `name`, or `prefLabel:<lang>`. A key with no `<sub>` is a scalar (see below).
- Example (single concept):
  - `["ext:ekw:bistum:id", "https://w3id.org/kim/ekw/bistum/hannover"]`
  - `["ext:ekw:bistum:prefLabel:de", "Hannover"]`
  - `["ext:ekw:bistum:type", "Concept"]`
- Multiple values for the same `<ns>:<facet>` pair repeat the tag triple, exactly as AMB-core arrays do (boundary on repeated `id`).
- Implementations MUST NOT fold ext entries into AMB-core properties on reverse conversion. They surface as a sibling `ext` object — see Example 3 and the reverse-conversion section.

##### Namespace selection and collision avoidance

`<ns>` is exactly **one** colon-free segment. Authorship, form identity, and deployment MUST NOT be encoded inside `<ns>`; a key such as `ext:30168:<pubkey>:<d-tag>:<facet>:id` is **invalid** under this NIP.

The rationale is the same as [NIP-32](https://github.com/nostr-protocol/nips/blob/master/32.md), which keeps a namespace in a separate tag position rather than inside the tag name: a single-segment `<ns>` lets the same logical facet unify across authors, forks and deployments of the same vocabulary, which is what makes `#ext:<ns>:<facet>:id` a usable filter. Multi-segment namespaces are also not parseable without out-of-band knowledge — see the parsing rule below.

To avoid collisions without a central registry, authorities SHOULD use **reverse domain name notation**, as NIP-32 recommends for `l` namespaces:

- `ext:org.edufeed.ekw:bistum:id`
- `ext:org.edufeed.ekw.konfi:zielgruppen:id`

A sub-vocabulary is a namespace of its own (`org.edufeed.ekw.konfi`), never a colon inside `<facet>`. Short unqualified slugs (`ekw`, `oersi`) remain valid and are common in existing data, but new authorities SHOULD prefer reverse-DNS.

##### Scalar ext properties

An ext key with no `<sub>` carries a plain literal value:

- `["ext:ekw:bibleReference", "Mt 5,1-12"]`
- `["ext:ekw:methodOther", "Bibliolog"]`

Repeated keys form an array of strings. On reverse conversion these surface as `output.ext.<ns>.<facet>` holding an array of strings, alongside — and structurally distinct from — concept facets, which hold an array of objects. Consumers MUST support both forms and MUST NOT discard a key merely because it lacks a `<sub>`.

##### Parsing rule (normative)

Consumers MUST parse ext keys **left-anchored** on `:`, with fixed arity:

```
ext-key = "ext" ":" ns ":" facet [ ":" sub ]
sub     = "id" / "type" / "name" / "prefLabel" ":" lang
```

1. Split the key on `:`. The first segment MUST be `ext`.
2. The second segment is `<ns>`; the third is `<facet>`. Both MUST be non-empty.
3. Everything after the third segment, rejoined with `:`, is `<sub>`. If absent, the tag is a scalar.
4. If `<sub>` is present it MUST match the closed set above. `prefLabel` MUST be followed by exactly one language segment.
5. A key that does not match this grammar MUST be ignored — consumers MUST NOT guess a segmentation, and MUST NOT absorb surplus segments into `<ns>`, `<facet>` or `<sub>`. Implementations SHOULD emit a warning so malformed producers are discoverable.

Rule 5 is load-bearing. Right-anchored heuristics ("the last segment is the sub, everything before it is the namespace") appear reasonable but assign different `(ns, facet)` pairs than left-anchored parsing whenever a key carries surplus segments, so two conformant-looking implementations can derive different metadata from identical bytes.

Producers MUST NOT emit keys outside this grammar. In particular, `<ns>` and `<facet>` MUST be checked for `:` before serialization.

##### Form-emitted ext (Edufeed convention)

When a kind 30168 form produces ext fields, `<ns>` is derived from the form's `d`-tag (a colon-free slug per Edufeed convention), optionally reverse-DNS qualified — e.g. a form with `d`-tag `amb-basic` emits `ext:amb-basic:<fieldId>:id` or `ext:org.edufeed.forms.amb-basic:<fieldId>:id`. The form-author's pubkey is **not** in `<ns>` — it's discoverable via the resource's `["a", "30168:<pub>:<d>", "<relay>", "form"]` back-ref. If two authors choose the same `<ns>`, the back-ref disambiguates which form was used; clients can layer `#a 30168:<pub>:<d>` to narrow.

##### Migrating legacy shapes

Two non-conformant shapes exist in deployed data and both SHOULD be migrated.

**Unprefixed namespaces.** Some events use a de-facto `<ns>:<facet>:<sub>` shape without the `ext:` prefix (notably from `amb-nostr-converter` and EKW pipelines). Producers SHOULD migrate to the prefixed form. Consumers MAY accept the unprefixed form for backward compatibility during a transition period, but the `ext:` prefix is the only forward-compatible shape because AMB-core may introduce new top-level properties that would otherwise collide.

**Surplus segments.** Keys carrying more than one namespace or facet segment — `ext:<ns>:<sub-vocabulary>:<facet>:<sub>` or `ext:30168:<pubkey>:<d-tag>:<facet>:<sub>` — predate the parsing rule above and are ambiguous by construction. Producers MUST migrate them by promoting the surplus segment into `<ns>`:

| Legacy | Conformant |
| --- | --- |
| `ext:ekw:konfi:zielgruppen:id` | `ext:org.edufeed.ekw.konfi:zielgruppen:id` |
| `ext:30168:<pub>:amb-basic:fach:id` | `ext:amb-basic:fach:id` (pubkey moves to the `a` back-ref) |

Because the two segmentations are indistinguishable to a consumer, there is no safe backward-compatibility shim: per rule 5, consumers MUST ignore these keys rather than guess. Migration is a re-publish of the affected events.

## How to convert an AMB nostr-event to AMB metadata

To convert a Nostr event back to AMB metadata:

1. **Extract tags**: Get the `tags` array from the Nostr event
2. **Group by prefix**: Collect all tags that share the same prefix (before the first `:`)
3. **Reconstruct nesting**: Use the `:` delimiter to rebuild nested object structure
4. **Handle arrays**: Multiple tags with identical keys become array elements
5. **Preserve order**: Array order is determined by tag order in the event
6. **Special mappings**:
   - `d` tag → `id` property: if the `d` value is an absolute URI, use it verbatim; otherwise derive the `id` as `nostr:<naddr1...>` (the [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) `naddr` encoding of kind `30142`, the event's `pubkey`, and the `d` value). Consumers MAY substitute a dereferenceable landing-page URL they control for the derived `nostr:` URI.
   - `content` field → `description` property (prefer over `description` tag if both exist)
   - `t` tags → `keywords` array
   - `r` tags → Nostr-native supplementary references (no AMB equivalent; not included in AMB output)
   - `p` tags with role → Nostr-native creator/contributor (see below)
   - `a` tags with role → Nostr-native relation (see below)
   - Convert string booleans to actual booleans (e.g. `isAccessibleForFree`)
   - Convert numeric strings back to integers where the AMB schema requires numbers (`suggestedAge:minValue`/`suggestedAge:maxValue`)
   - Parse ISO8601 dates if needed for validation
7. **Add `@context`**: The output MUST include `"@context": ["https://w3id.org/kim/amb/context.jsonld", {"@language": "<lang>"}]` — the AMB schema requires `@context` at the top level. The language is implementation-configurable (default: `de`).

   > **Known limitation:** Custom or extended `@context` entries from a source AMB document (e.g. an additional `"https://schema.org"` entry) are not stored in the event and therefore cannot be restored on reverse conversion — the canonical two-element context is always reconstructed. Documents using only the standard AMB context round-trip losslessly.
8. **Nostr-native `p` tags** (creator/contributor): For each `["p", <pubkey-hex>, <relay-hint>, <role>]` where `<role>` is `"creator"` or `"contributor"`, clients SHOULD fetch the user's kind:0 profile (using the relay hint and NIP-65) to resolve their `name`. Map to an AMB creator/contributor object:
   ```json
   {
     "name": "<name from kind:0 profile>",
     "type": "Person",
     "id": "nostr:<nprofile1...>"
   }
   ```
   The `id` uses the NIP-19 `nprofile` encoding (which includes the pubkey and relay hint(s)) prefixed with `nostr:` per NIP-21. The `type` (`"Person"` or `"Organization"`) should be determined from the kind:0 profile if possible; implementations MAY default to `"Person"` when unknown.

   If the kind:0 profile cannot be fetched (or the converter operates offline), `name` MUST fall back to the NIP-19 `npub` encoding of the pubkey — the AMB schema requires `name` and `type` on every creator/contributor object, so output must never omit them. Profile-aware clients SHOULD replace the fallback with the resolved profile name once available.
9. **Nostr-native `a` tags** (relations): For each `["a", "30142:<pubkey>:<d-value>", <relay-hint>, <role>]` where `<role>` is `"isBasedOn"`, `"isPartOf"`, or `"hasPart"`, map to the corresponding AMB relation object:
   ```json
   {
     "id": "nostr:<naddr1...>",
     "type": "LearningResource"
   }
   ```
   The `id` uses the NIP-19 `naddr` encoding (which includes kind, pubkey, d-tag, and relay hint(s)) prefixed with `nostr:` per NIP-21.
10. **Extension tags (`ext:` prefix)**: Parse each key whose first segment is `ext` using the normative left-anchored rule in *Extension Properties*, ignoring any key that does not match. Group the surviving tags by `(<namespace>, <facet>)`. Within each pair, apply the same flattening rules as AMB-core (boundary on repeated `id`, `prefLabel:<lang>` → `prefLabel.<lang>`); keys with no `<sub>` yield an array of strings instead. Place the result under `output.ext.<namespace>.<facet>`. Implementations MUST NOT merge ext entries into AMB-core properties.


## How to query for AMB nostr-events in supporting relays

AMB-supporting relays MUST support the standard NIP-01 filter fields and SHOULD support NIP-50 full-text search with field-specific filtering.

### Standard Nostr Filters (NIP-01)

Clients can query AMB events using standard Nostr filter fields:

- `kinds` — filter by event kind (always `30142` for AMB events)
- `authors` — filter by pubkey
- `ids` — filter by event ID
- `#d` — filter by the addressable event identifier (d-tag)
- `since` / `until` — filter by `created_at` timestamp range

### AMB Tag Filters

In addition to standard single-letter tag filters, AMB-supporting relays SHOULD support filtering by the colon-delimited tag names used in AMB events. The tag name in the filter maps directly to the flattened tag key in the event:

| Tag Filter | Description |
|---|---|
| `#t` | Filter by keyword |
| `#r` | Filter by external reference URL |
| `#p` | Filter by creator/contributor pubkey |
| `#a` | Filter by addressable event reference |
| `#about:id` | Filter by subject (controlled vocabulary URI) |
| `#learningResourceType:id` | Filter by resource type URI |
| `#educationalLevel:id` | Filter by educational level URI |
| `#audience:id` | Filter by target audience URI |
| `#ext:<ns>:<facet>:id` | Filter by extension property URI within a namespace |
| `#ext:<ns>:<facet>:prefLabel:<lang>` | Filter by extension property label |
| `#ext:<ns>:<facet>` | Filter by scalar extension property value |

Any colon-delimited tag name present in AMB events can be used as a filter. Multiple values for the same tag are matched with OR logic. Different tag filters are combined with AND logic.

### NIP-50 Full-Text Search

AMB-supporting relays SHOULD implement [NIP-50](https://github.com/nostr-protocol/nips/blob/master/50.md) to allow full-text search across AMB metadata fields (at minimum: `name`, `description`, `keywords`).

Relays MAY additionally support field-specific search filtering using dot-notation within the `search` string. The dot-notation maps to the nested AMB field structure (e.g., `publisher.name` maps to the `name` subfield of `publisher` objects):

| Field Path | Description |
|---|---|
| `publisher.name` | Publisher organization name |
| `creator.name` | Content creator name |
| `about.prefLabel.<lang>` | Subject/topic label (e.g., `about.prefLabel.de`) |
| `learningResourceType.prefLabel.<lang>` | Resource type label |
| `audience.prefLabel.<lang>` | Target audience label |
| `educationalLevel.prefLabel.<lang>` | Educational level label |
| `ext.<ns>.<facet>.id` | Extension property URI |
| `ext.<ns>.<facet>.prefLabel.<lang>` | Extension property label |
| `ext.<ns>.<facet>.type` | Extension property RDF type |

Free-text terms and field filters can be mixed in the search string. Multiple values for the same base field are combined with OR logic.

### Query Examples

#### JSON Filter Objects

```json
// All AMB events
{"kinds": [30142]}

// Events by a specific author
{"kinds": [30142], "authors": ["<pubkey-hex>"]}

// Lookup by addressable event coordinate (kind + pubkey + d-tag)
{"kinds": [30142], "authors": ["<pubkey-hex>"], "#d": ["<d-tag-value>"]}

// Events created in a time range
{"kinds": [30142], "since": 1700000000, "until": 1800000000}

// Filter by keyword
{"kinds": [30142], "#t": ["Mathematik"]}

// Filter by subject URI
{"kinds": [30142], "#about:id": ["http://w3id.org/kim/schulfaecher/s1017"]}

// Filter by learning resource type URI
{"kinds": [30142], "#learningResourceType:id": ["http://w3id.org/openeduhub/vocabs/new_lrt/video"]}

// Filter by educational level URI
{"kinds": [30142], "#educationalLevel:id": ["https://w3id.org/kim/educationalLevel/level_06"]}

// Filter by external reference
{"kinds": [30142], "#r": ["https://doi.org/10.1234/example"]}

// Filter by creator/contributor pubkey
{"kinds": [30142], "#p": ["<pubkey-hex>"]}

// Filter by addressable event reference
{"kinds": [30142], "#a": ["30142:<pubkey-hex>:<d-tag-value>"]}

// NIP-50 full-text search
{"kinds": [30142], "search": "pythagorean theorem"}

// NIP-50 search with field-specific filter
{"kinds": [30142], "search": "publisher.name:e-teaching.org"}

// NIP-50 combined: free text + field filter
{"kinds": [30142], "search": "forschung publisher.name:e-teaching.org"}

// NIP-50 multiple values for same field (OR logic)
{"kinds": [30142], "search": "about.prefLabel.de:Mathematik about.prefLabel.de:Physik"}
```

#### nak CLI Examples

```bash
# All AMB events
nak req -k 30142 ws://relay.example.com

# By author
nak req -a <pubkey-hex> -k 30142 ws://relay.example.com

# By d-tag
nak req -d "https://oersi.org/resources/example123" -k 30142 ws://relay.example.com

# Time range
nak req --since 1700000000 --until 1800000000 -k 30142 ws://relay.example.com

# By keyword
nak req -t t=Mathematik -k 30142 ws://relay.example.com

# By subject URI
nak req -t about:id=http://w3id.org/kim/schulfaecher/s1017 -k 30142 ws://relay.example.com

# By learning resource type
nak req -t learningResourceType:id=http://w3id.org/openeduhub/vocabs/new_lrt/video -k 30142 ws://relay.example.com

# By external reference
nak req -t r=https://doi.org/10.1234/example -k 30142 ws://relay.example.com

# By creator/contributor pubkey
nak req -p <pubkey-hex> -k 30142 ws://relay.example.com

# Full-text search
nak req --search "pythagorean theorem" -k 30142 ws://relay.example.com

# Field-specific search
nak req --search "publisher.name:e-teaching.org" -k 30142 ws://relay.example.com

# Combined: free text + field filter
nak req --search "forschung publisher.name:e-teaching.org" -k 30142 ws://relay.example.com
```

> **Note:** Relays that require [NIP-42](https://github.com/nostr-protocol/nips/blob/master/42.md) authentication need `--sec <key> --auth` flags with `nak`.

### Reference Implementations

- **[amb-relay](https://git.edufeed.org/edufeed/amb-relay)** — Nostr relay specialized for AMB events, built on the khatru relay framework
- **[nostrlib/eventstore/typesense30142](https://git.edufeed.org/edufeed/nostrlib/src/branch/master/eventstore/typesense30142)** — Typesense-backed eventstore for kind 30142 events with full query documentation in its [README](https://git.edufeed.org/edufeed/nostrlib/src/branch/master/eventstore/typesense30142/README.md)

## Examples

### Example 1: Simple Educational Resource

```json
{
  "kind": 30142,
  "id": "6ba638a3786cfce89af1702a36c59e0bd9206863afa5cb6b1299aaf0d9f48c84",
  "pubkey": "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
  "created_at": 1743419457,
  "tags": [
    ["d", "https://oersi.org/resources/aHR0cHM6Ly9hdi50aWIuZXUvbWVkaWEvNjY5ODM=11"],
    ["type", "LearningResource"],
    ["name", "Pythagorean Theorem Video"],
    ["description", "An introductory video explaining the Pythagorean theorem"],
    ["about:id", "http://w3id.org/kim/schulfaecher/s1017"],
    ["about:prefLabel:de", "Mathematik"],
    ["about:type", "Concept"],
    ["about:id", "http://w3id.org/kim/schulfaecher/s1005"],
    ["about:prefLabel:de", "Deutsch"],
    ["about:type", "Concept"],
    ["learningResourceType:id", "http://w3id.org/openeduhub/vocabs/new_lrt/7a6e9608-2554-4981-95dc-47ab9ba924de"],
    ["learningResourceType:prefLabel:de", "Video"],
    ["learningResourceType:type", "Concept"],
    ["t", "Pythagoras"],
    ["t", "Geometrie"],
    ["t", "Mathematik"],
    ["inLanguage", "de"],
    ["license:id", "https://creativecommons.org/licenses/by/4.0/"],
    ["isAccessibleForFree", "true"]
  ],
  "content": "An introductory video explaining the Pythagorean theorem",
  "sig": "6b0b78d56dea322864d35ea3b6d7e892d0e62bed96cd11ecb27d6c1d0b6d0cd68cd9ec82419946a5fb3c8d4a21eca88c9a5dad47a3b3e466ba18787224a613ef"
}
```

### Example 2: Resource with Nostr-Native and External Creators

This example demonstrates both creator types: a Nostr-native creator (using a `p` tag only) and an external creator without a Nostr identity (using `creator:*` flattened tags).

```json
{
  "kind": 30142,
  "id": "7ca749b4897efdc98fe2803dc60f68c9e1cd29764e8a55d1e9ef47a46ba4fe75",
  "pubkey": "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
  "created_at": 1743419500,
  "tags": [
    ["d", "https://example.org/courses/physics-101"],
    ["type", "LearningResource"],
    ["type", "Course"],
    ["name", "Introduction to Physics"],
    ["description", "A comprehensive introduction to classical mechanics"],
    ["p", "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798", "wss://relay.example.com", "creator"],
    ["creator:id", "https://orcid.org/0000-0009-8765-4321"],
    ["creator:name", "Prof. John Doe"],
    ["creator:type", "Person"],
    ["creator:honorificPrefix", "Prof."],
    ["creator:affiliation:name", "Stanford University"],
    ["creator:affiliation:type", "Organization"],
    ["dateCreated", "2024-01-15"],
    ["datePublished", "2024-02-01"],
    ["about:id", "https://w3id.org/kim/hochschulfaechersystematik/n079"],
    ["about:prefLabel:de", "Informatik"],
    ["about:type", "Concept"],
    ["learningResourceType:id", "https://w3id.org/kim/hcrt/course"],
    ["learningResourceType:prefLabel:de", "Kurs"],
    ["audience:id", "http://purl.org/dcx/lrmi-vocabs/educationalAudienceRole/student"],
    ["audience:prefLabel:de", "Student"],
    ["audience:type", "Concept"],
    ["educationalLevel:id", "https://w3id.org/kim/educationalLevel/level_06"],
    ["educationalLevel:prefLabel:en", "Bachelor or equivalent"],
    ["inLanguage", "en"],
    ["license:id", "https://creativecommons.org/licenses/by-sa/4.0/"],
    ["isAccessibleForFree", "true"],
    ["r", "https://example.org/courses/physics-101"],
    ["r", "https://doi.org/10.1234/physics-intro"]
  ],
  "content": "A comprehensive introduction to classical mechanics",
  "sig": "8d1c89f5da33ec9a2b456def78a90b1cd23e456f78a90b12cd34e567f89a012b34c56d78e9f0a12bc3d45e6f78901a23b45c67d89e0f1a2b3c4d5e6f7890123a"
}
```

In this example:
- The first creator has a Nostr pubkey, so only a `p` tag with role `"creator"` is used. Their name and metadata are resolved from their kind:0 profile.
- The second creator (Prof. John Doe) has no Nostr identity, so the `creator:*` flattened tags provide their name, type, affiliation, and ORCID.

### Example 3: Resource with Extension Namespace

This example demonstrates the `ext:` namespace, used here to attach an EKW-specific `bistum` (diocese) facet that is not part of AMB-core. The same flattening grammar applies; the only difference is the leading `ext:<ns>:` prefix.

```json
{
  "kind": 30142,
  "id": "9f4c2a1b8e7d3a6f5c2b9d8e7a1c4f3b8e2d9a7c5f1b3e8d6a4c2f9b7e5d3a1c",
  "pubkey": "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
  "created_at": 1764000000,
  "tags": [
    ["d", "https://ekw.de/resources/abc"],
    ["type", "LearningResource"],
    ["name", "Religiöse Bildung im Bistum Hannover"],
    ["about:id", "https://w3id.org/kim/hochschulfaechersystematik/n270"],
    ["about:prefLabel:de", "Theologie"],
    ["about:type", "Concept"],
    ["ext:ekw:bistum:id", "https://w3id.org/kim/ekw/bistum/hannover"],
    ["ext:ekw:bistum:prefLabel:de", "Hannover"],
    ["ext:ekw:bistum:type", "Concept"],
    ["ext:ekw:bistum:id", "https://w3id.org/kim/ekw/bistum/wuerttemberg"],
    ["ext:ekw:bistum:prefLabel:de", "Württemberg"],
    ["ext:ekw:bistum:type", "Concept"]
  ],
  "content": "",
  "sig": "..."
}
```

On reverse conversion to AMB metadata, the `ext` block surfaces as a sibling object (it MUST NOT be folded into AMB-core):

```json
{
  "type": ["LearningResource"],
  "name": "Religiöse Bildung im Bistum Hannover",
  "about": [
    {"id": "https://w3id.org/kim/hochschulfaechersystematik/n270", "prefLabel": {"de": "Theologie"}, "type": "Concept"}
  ],
  "ext": {
    "ekw": {
      "bistum": [
        {"id": "https://w3id.org/kim/ekw/bistum/hannover", "prefLabel": {"de": "Hannover"}, "type": "Concept"},
        {"id": "https://w3id.org/kim/ekw/bistum/wuerttemberg", "prefLabel": {"de": "Württemberg"}, "type": "Concept"}
      ]
    }
  }
}
```

Consumers that don't recognize the `ekw` namespace ignore it; consumers that do can render `bistum` generically (one row per concept, label resolved by language).

## Tools

### Using `nak` to create AMB events

You can use [`nak`](https://github.com/fiatjaf/nak) to create AMB events. There are two approaches:

#### Flag-based (inline tags)

```bash
# Simple resource with Nostr-native t tags
nak event \
  -k 30142 \
  --tag d="https://oersi.org/resources/example123" \
  --tag type="LearningResource" \
  --tag name="Pythagorean Theorem Video" \
  --tag description="An introductory video" \
  --tag about:id="http://w3id.org/kim/schulfaecher/s1017" \
  --tag about:prefLabel:de="Mathematik" \
  --tag t="Pythagoras" \
  --tag t="Geometrie" \
  --tag inLanguage="de" \
  --tag license:id="https://creativecommons.org/licenses/by/4.0/" \
  --sec <key> --auth ws://relay.example.com

# Resource with Nostr-native creator (p tag)
nak event \
  -k 30142 \
  --tag d="https://example.org/resource/456" \
  --tag name="Physics Course" \
  -p "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798;wss://relay.example.com;creator" \
  --sec <key> --auth ws://relay.example.com
```

#### JSON on stdin (pipe-based)

This approach gives full control over the tag structure and is useful for scripting:

```bash
echo '{
  "tags": [
    ["d", "https://example.org/courses/physics-101"],
    ["type", "LearningResource"],
    ["name", "Introduction to Physics"],
    ["description", "A comprehensive introduction to classical mechanics"],
    ["inLanguage", "en"],
    ["t", "physics"],
    ["t", "mechanics"],
    ["creator:name", "Dr. Jane Smith"],
    ["creator:type", "Person"],
    ["license:id", "https://creativecommons.org/licenses/by-sa/4.0/"]
  ],
  "content": "A comprehensive introduction to classical mechanics"
}' | nak event -k 30142 --sec <key> --auth ws://relay.example.com
```

## References

- [AMB Specification](https://dini-ag-kim.github.io/amb/latest/)
- [Nostr Protocol (NIP-01)](https://github.com/nostr-protocol/nips/blob/master/01.md) - including addressable events (formerly NIP-33, merged into NIP-01)
- [bech32-encoded entities (NIP-19)](https://github.com/nostr-protocol/nips/blob/master/19.md) - `nprofile` and `naddr` encodings for reverse conversion
- [`nostr:` URI scheme (NIP-21)](https://github.com/nostr-protocol/nips/blob/master/21.md) - `nostr:` prefix for bech32 identifiers in AMB output
- [Extra Metadata Fields and Tags (NIP-24)](https://github.com/nostr-protocol/nips/blob/master/24.md) - `r` and `t` tag conventions
- [Live Activities (NIP-53)](https://github.com/nostr-protocol/nips/blob/master/53.md) - precedent for `p` tag roles
- [Relay List Metadata (NIP-65)](https://github.com/nostr-protocol/nips/blob/master/65.md) - relay discovery for `p` tag relay hints
- [JSON-Flattening Concept](https://localizely.com/json-flattener/)
