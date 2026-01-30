# Edufeed: AMB-NIP

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

1. **Simple properties**: Map directly to tags
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

- `id` → `["d", <id>]` (special case: use Nostr's `d` tag as identifier)
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
  - **External (no Nostr identity)**:
    - `["creator:id", <uri>]` (optional, e.g., ORCID, GND)
    - `["creator:name", <name>]`
    - `["creator:type", <"Person"|"Organization">]`
    - `["creator:honorificPrefix", <title>]` (optional, for persons)
    - `["creator:affiliation:id", <uri>]` (optional)
    - `["creator:affiliation:name", <name>]` (optional)
    - `["creator:affiliation:type", "Organization"]` (optional)
- `contributor` (array of Person/Organization objects) → Same structure as `creator`, using role `"contributor"` in the `p` tag
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

#### Relations:

- `isBasedOn` (array of objects) → Repeat for each:
  - **Nostr-native (if referenced resource is addressable AMB event)**: `["a", "30142:<pubkey>:<d-value>", <relay>, "isBasedOn"]`
  - **Fallback (for external URIs)**:
    - `["isBasedOn:id", <uri>]`
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

## How to convert an AMB nostr-event to AMB metadata

To convert a Nostr event back to AMB metadata:

1. **Extract tags**: Get the `tags` array from the Nostr event
2. **Group by prefix**: Collect all tags that share the same prefix (before the first `:`)
3. **Reconstruct nesting**: Use the `:` delimiter to rebuild nested object structure
4. **Handle arrays**: Multiple tags with identical keys become array elements
5. **Preserve order**: Array order is determined by tag order in the event
6. **Special mappings**:
   - `d` tag → `id` property
   - `content` field → `description` property (prefer over `description` tag if both exist)
   - `t` tags → `keywords` array
   - `r` tags → Nostr-native supplementary references (no AMB equivalent; not included in AMB output)
   - `p` tags with role → Nostr-native creator/contributor (see below)
   - `a` tags with role → Nostr-native relation (see below)
   - Convert string booleans to actual booleans
   - Parse ISO8601 dates if needed for validation
7. **Nostr-native `p` tags** (creator/contributor): For each `["p", <pubkey-hex>, <relay-hint>, <role>]` where `<role>` is `"creator"` or `"contributor"`, clients MUST fetch the user's kind:0 profile (using the relay hint and NIP-65) to resolve their `name`. Map to an AMB creator/contributor object:
   ```json
   {
     "name": "<name from kind:0 profile>",
     "id": "nostr:<nprofile1...>"
   }
   ```
   The `id` uses the NIP-19 `nprofile` encoding (which includes the pubkey and relay hint(s)) prefixed with `nostr:` per NIP-21. The `type` (`"Person"` or `"Organization"`) should be determined from the kind:0 profile if possible; implementations MAY default to `"Person"` when unknown.
8. **Nostr-native `a` tags** (relations): For each `["a", "30142:<pubkey>:<d-value>", <relay-hint>, <role>]` where `<role>` is `"isBasedOn"`, `"isPartOf"`, or `"hasPart"`, map to the corresponding AMB relation object:
   ```json
   {
     "id": "nostr:<naddr1...>",
     "type": "LearningResource"
   }
   ```
   The `id` uses the NIP-19 `naddr` encoding (which includes kind, pubkey, d-tag, and relay hint(s)) prefixed with `nostr:` per NIP-21.


## How to query for AMB nostr-events in supporting relays

TODO: This section will be completed once relay support is implemented.

Query capabilities should include:
- Filter by `kind:30142` for all AMB events
- Filter by author (`pubkey`)
- Filter by specific subjects using `#about:id` tag
- Filter by learning resource type using `#learningResourceType:id` tag
- Filter by educational level using `#educationalLevel:id` tag
- Full-text search in `name` and `description` tags

## Examples

### Example 1: Simple Educational Resource

```json
{
  "kind": 30142,
  "id": "6ba638a3786cfce89af1702a36c59e0bd9206863afa5cb6b1299aaf0d9f48c84",
  "pubkey": "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
  "created_at": 1743419457,
  "tags": [
    ["d", "oersi.org/resources/aHR0cHM6Ly9hdi50aWIuZXUvbWVkaWEvNjY5ODM=11"],
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

## Tools

### Using `nak` to create AMB events

You can use [`nak`](https://github.com/fiatjaf/nak) to create AMB events with the flattened tag structure:

```bash
# Example 1: Simple resource with Nostr-native t tags
nak event \
  --kind 30142 \
  --tag d="oersi.org/resources/example123" \
  --tag type="LearningResource" \
  --tag name="Pythagorean Theorem Video" \
  --tag description="An introductory video" \
  --tag about:id="http://w3id.org/kim/schulfaecher/s1017" \
  --tag about:prefLabel:de="Mathematik" \
  --tag about:type="Concept" \
  --tag t="Pythagoras" \
  --tag t="Geometrie" \
  --tag inLanguage="de" \
  --tag license:id="https://creativecommons.org/licenses/by/4.0/"

# Example 2: Resource with Nostr-native creator (p tag only)
nak event \
  --kind 30142 \
  --tag d="https://example.org/resource/456" \
  --tag name="Physics Course" \
  --tag p="79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798;wss://relay.example.com;creator"

# Example 3: Resource with non-Nostr creator (flattened tags)
nak event \
  --kind 30142 \
  --tag d="https://example.org/resource/789" \
  --tag name="Chemistry Course" \
  --tag creator:name="Prof. John Doe" \
  --tag creator:type="Person" \
  --tag creator:affiliation:name="MIT" \
  --tag creator:affiliation:type="Organization"
```

## References

- [AMB Specification](https://dini-ag-kim.github.io/amb/latest/)
- [Nostr Protocol (NIP-01)](https://github.com/nostr-protocol/nips/blob/master/01.md)
- [Addressable Events (NIP-33)](https://github.com/nostr-protocol/nips/blob/master/33.md)
- [bech32-encoded entities (NIP-19)](https://github.com/nostr-protocol/nips/blob/master/19.md) - `nprofile` and `naddr` encodings for reverse conversion
- [`nostr:` URI scheme (NIP-21)](https://github.com/nostr-protocol/nips/blob/master/21.md) - `nostr:` prefix for bech32 identifiers in AMB output
- [Extra Metadata Fields and Tags (NIP-24)](https://github.com/nostr-protocol/nips/blob/master/24.md) - `r` and `t` tag conventions
- [Live Activities (NIP-53)](https://github.com/nostr-protocol/nips/blob/master/53.md) - precedent for `p` tag roles
- [Relay List Metadata (NIP-65)](https://github.com/nostr-protocol/nips/blob/master/65.md) - relay discovery for `p` tag relay hints
- [JSON-Flattening Concept](https://localizely.com/json-flattener/)
