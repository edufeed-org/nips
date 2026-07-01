# NIP-DIDACTIC

Funded Didactic Projects, Teaching Measures, and Project Publications
---------------------------------------------------------------------

`draft` `optional`

## Abstract

This NIP defines three addressable event kinds for representing **funded
didactic projects** (and their byproducts) on Nostr:

| Kind   | Entity        | Schema-type                     |
|-------:|---------------|---------------------------------|
| `30143` | Projekt       | `Project`                       |
| `30144` | Maßnahme      | `TeachingMeasure`               |
| `30145` | Publikation   | `ScholarlyArticle` (or similar) |

The first producer of these kinds is [transferkiosk.net][tk] (the Stiftung
Innovation in der Hochschullehre / *Stil* portal), but the shapes are generic
enough to also describe BMBF / DFG / EU-funded teaching innovation projects.

This NIP is a companion to [NIP-AMB][amb]:

- NIP-AMB (kind `30142`) models *consumable* learning resources (videos,
  textbooks, OER). It does not have a slot for the funded project that
  produced a resource, nor for the documented teaching intervention that a
  resource arose from, nor for the scholarly publication that reflects on it.
- NIP-DIDACTIC fills those gaps. The three kinds reuse NIP-AMB's flattening
  grammar and Nostr-native tag conventions so AMB-aware tooling can be
  extended uniformly.
- Vocabularies for the controlled fields below are published per
  [NIP-VOCAB][vocab] (kinds `39737` ConceptScheme / `39738` Concept) and
  referenced from event tags.

[tk]: https://transferkiosk.net/
[amb]: AMB.md
[vocab]: VOCAB.md

### Authorship and trust model

These events are **third-party catalog records**. The event `pubkey` is the
publisher/indexer that mirrors the source catalogue (e.g. the edufeed importer
for transferkiosk.net) — **not** the project, its host institution, or a
publication's authors. Tags such as `host:*`, `funder:*`, and `author:*`
describe entities named in the source data; they are claims attested by the
publishing pubkey, not self-published identities. A consumer that needs
provenance should treat the `pubkey` as the asserting party and follow the `r`
tag back to the source record. When a described person or organisation has its
own Nostr identity, a `p` tag links to it (see below); absent that, the
flattened `*:name` fallbacks carry display metadata only, not identity.

## Shared Conventions

All three kinds inherit the NIP-AMB flattening grammar:

- **`d` tag** — stable identifier, addressed as `kind:pubkey:d`.
- **`type` tag** — human-readable schema-type label. Authoritative routing is
  done via the kind number.
- **Flattening with `:`** — nested objects are flattened with `:` as the
  delimiter (`funder:program:name`, `host:location:name`).
- **Arrays** — repeat the same tag key in order. When an array element is itself
  an object spanning several tags (a concept triple, an `author`, an `editor`),
  that element's tags MUST be emitted contiguously and led by its anchor key:
  `:id` for concept triples, `:name` for persons/organisations. A repeat of the
  anchor key marks the start of the next element. This is the "boundary on
  repeated `:id`" rule inherited from [NIP-AMB][amb]; consumers reassemble
  objects by splitting each repeated group at its anchor key.
- **`content` field** — duplicates the `description` tag for client display.
- **`t` tag** — keywords / Stichwort.
- **`r` tag** — external URLs (source page, DOI, project website).
- **`i` tag** — external identifier URIs (ORCID, ROR, GND, w3id.org URIs).
- **`p` tag** — for project members / authors with a Nostr identity:
  `["p", <pubkey-hex>, <relay-hint>, <role>]` where `<role>` is
  `"creator"`, `"contributor"`, `"author"`, or `"participant"`. Persons
  without a Nostr identity use the flattened `creator:*` / `author:*` /
  `participant:*` fallbacks.
- **`ext:<ns>:<facet>:<sub>`** — extension namespace for source-specific
  fields not standardised here, exactly as in NIP-AMB. The transferkiosk
  producer uses `ext:tk:*`.

### Concept references (flat-concept triple)

Wherever a tag references a term from a controlled vocabulary, three flat
tags together describe the concept:

```
["<facet>:id",              "<concept URI>"]
["<facet>:prefLabel:<lang>", "<label>"]
["<facet>:type",             "Concept"]
```

When a facet carries several concepts, the three tags repeat per concept. The
`<facet>:id` tag MUST come first within each concept and marks the concept
boundary (see *Arrays* above); consumers group each `:prefLabel`/`:type` with
the most recent `:id`.

The `<concept URI>` is **always** an external (resolvable or stable) URI —
never a Nostr coordinate. If the vocabulary is also published on Nostr per
NIP-VOCAB, an additional companion `a` tag SHOULD be emitted so vocab-aware
clients can fetch the underlying `kind:39738` event for label translation,
narrower/broader concepts, etc.:

```
["a", "39738:<pubkey>:<concept-d>", "<relay>", "<facet>"]
```

The fourth element matches the flattened tag prefix (`audience`,
`activity`, `about`, …). Consumers without a vocab-aware index can ignore
the `a` and still group by `<facet>:id`.

### Cross-entity relations

A relation between two events of any of the three kinds is expressed with
an `a` tag whose fourth element is a relationship marker:

| Marker         | Direction                | Meaning                                                  |
|----------------|--------------------------|----------------------------------------------------------|
| `isPartOf`     | Maßnahme → Projekt; sub-Projekt → Verbund-Projekt | The subject belongs inside the referenced object.    |
| `hasPart`      | Inverse of `isPartOf`    | Optional; mostly used by indexers.                       |
| `isOutputOf`   | Publikation → Projekt    | The publication is an output of the referenced project.  |
| `documents`    | Publikation → Maßnahme   | The publication reports on a specific measure.           |

The first three reuse NIP-AMB vocabulary. `isOutputOf` and `documents`
are introduced by this NIP.

## Kind 30143 — Projekt

A funded didactic / teaching-innovation project. Time-bounded, hosted at
one or more institutions, with at least one funding source.

### Required tags

| Tag          | Notes                                          |
|--------------|------------------------------------------------|
| `d`          | Stable identifier (e.g. source URL).           |
| `type`       | `"Project"`.                                   |
| `name`       | Project title.                                 |
| `description`| Project summary. ALSO duplicated into `content`. |

### Optional tags

| Tag                                  | Source/meaning                                |
|--------------------------------------|-----------------------------------------------|
| `acronym`                            | Project acronym / Kurztitel.                  |
| `projectNumber`                      | Funder-internal number, e.g. `"FR-599/2023"`. |
| `startDate`, `endDate`               | ISO 8601 dates.                               |
| `extendedUntil`                      | ISO 8601 date; optional extension cutoff.     |
| `status`                             | e.g. `veroeffentlicht`, `laufend`.            |
| `inLanguage`                         | BCP47 code, repeatable.                       |
| `image`                              | URL of a representative image.                |
| `dateCreated` / `datePublished` / `dateModified` | ISO 8601 (per NIP-AMB).        |
| `funder:name` / `funder:type` / `funder:id`         | Funding body. `type=FundingScheme` for programs. |
| `funder:program:name` / `funder:program:id`         | Funding program inside the funder.            |
| `funder:programLine:name` / `funder:programLine:id` | Sub-line inside the program.                  |
| `host:id` / `host:name` / `host:type`               | Hosting institution. `type=Organization`.     |
| `host:location:id` / `host:location:name`           | E.g. federal state.                           |
| `host:kind:id` / `host:kind:name`                   | Institution type (`Universität`, `Fachhochschule`, …). |
| `participant:name` / `participant:role` / `participant:id` | Non-Nostr team members.                  |
| `p`                                  | Team members with Nostr identity, role `"participant"`. |
| `r`                                  | Source URL, project website, …                |
| `t`                                  | Keywords.                                     |

### Concept references

Concept triples (with companion `a` tag) using these facets:

| Facet         | Meaning                              | Vocab d-tag (transferkiosk producer) |
|---------------|--------------------------------------|---------------------------------------|
| `about`       | Subject area (Fächergruppe/-bereich) | `tk-faechergruppen`, `tk-fachbereiche` |
| `objective`   | Project goal (Projektziel)           | `tk-projektziele`                     |

### Relations

- `["a", "30143:<pub>:<d>", "<relay>", "isPartOf"]` — sub-project of a Verbund-Projekt.

### Example

```jsonc
{
  "kind": 30143,
  "tags": [
    ["d", "https://transferkiosk.net/p/101553"],
    ["type", "Project"],
    ["name", "Lernspiel für die Orthopädie und Unfallchirurgie"],
    ["acronym", "LeOpädchi"],
    ["projectNumber", "FR-599/2023"],
    ["description", "Ein wesentlicher Teil akuter medizinischer Konsultationen geht auf Erkrankungen und Verletzungen des Bewegungsapparates zurück …"],
    ["startDate", "2024-04-01"],
    ["endDate", "2026-03-31"],
    ["status", "veroeffentlicht"],
    ["datePublished", "2025-12-19"],
    ["inLanguage", "de"],
    ["host:id", "https://transferkiosk.net/vocab/tk-einrichtung/100448"],
    ["host:name", "Universitätsklinikum Bonn"],
    ["host:type", "Organization"],
    ["host:location:name", "Nordrhein-Westfalen"],
    ["host:kind:name", "Universitäten"],
    ["funder:name", "Freiraum 2023"],
    ["funder:type", "FundingScheme"],
    ["funder:program:name", "Freiraum"],
    ["funder:programLine:name", "Förderlinie C"],
    ["about:id", "https://transferkiosk.net/vocab/tk-faechergruppen/100004"],
    ["about:prefLabel:de", "Humanmedizin/Gesundheitswissenschaften"],
    ["about:type", "Concept"],
    ["a", "39738:d2689e2f…:tk-faechergruppen/100004", "wss://relay.edufeed.org", "about"],
    ["objective:id", "https://transferkiosk.net/vocab/tk-projektziele/100007"],
    ["objective:prefLabel:de", "Hochschule: Flexibilisierung & Individualisierung"],
    ["objective:type", "Concept"],
    ["a", "39738:d2689e2f…:tk-projektziele/100007", "wss://relay.edufeed.org", "objective"],
    ["r", "https://transferkiosk.net/p/101553"]
  ],
  "content": "Ein wesentlicher Teil akuter medizinischer Konsultationen …"
}
```

## Kind 30144 — Maßnahme

A documented concrete teaching intervention within a Projekt. Carries
narrative sections plus rich classification across many didactic axes.

### Required tags

| Tag          | Notes                                                              |
|--------------|--------------------------------------------------------------------|
| `d`          | Stable identifier (e.g. source URL).                               |
| `type`       | `"TeachingMeasure"`.                                               |
| `name`       | Measure title (Maßnahmentitel).                                    |
| `description`| Summary (Zusammenfassung). Also duplicated into `content`.         |
| `a` (isPartOf) | `["a", "30143:<pub>:<d>", "<relay>", "isPartOf"]` to parent Projekt. |

### Narrative section tags

Each MAY appear at most once. The value is short text; long prose SHOULD be
duplicated into `content` (as markdown headings) and the tag kept only for
queryability.

| Tag             | Source                                |
|-----------------|----------------------------------------|
| `summary`       | Zusammenfassung                        |
| `challenge`     | Herausforderungen                      |
| `approach`      | Herangehensweise                       |
| `context`       | Zusammenhang                           |
| `prerequisite`  | Voraussetzungen                        |
| `procedure`     | Vorgehen                               |
| `outcome`       | Effekte                                |
| `learnings`     | Learnings                              |
| `recommendation`| Empfehlung                             |
| `tips`          | Tipps                                  |

### Method / format / tool annotations

Recommended and not-recommended values use distinct tag names so consumers
can render the polarity:

| Tag                    | Meaning                              |
|------------------------|--------------------------------------|
| `method:recommended`   | A method the author recommends.      |
| `method:notRecommended`| A method the author warns against.   |
| `format:recommended`   | …                                    |
| `format:notRecommended`| …                                    |
| `tool:recommended`     | …                                    |
| `tool:notRecommended`  | …                                    |

### Concept references

| Facet            | Meaning                            | Vocab d-tag (transferkiosk producer) |
|------------------|------------------------------------|---------------------------------------|
| `audience`       | Zielgruppe                          | `tk-zielgruppen`                      |
| `about`          | Fachbereich / Fächergruppe          | `tk-massnahme-fachbereiche`, `tk-massnahme-faechergruppen` |
| `activity`       | Aktivität                           | `tk-aktivitaeten`                     |
| `actionScope`    | Aktionsradius                       | `tk-aktionsradius`                    |
| `actionField`    | Handlungsfeld                       | `tk-handlungsfeld`                    |
| `studyModel`     | Studienmodell                       | `tk-studienmodell`                    |
| `effortLevel`    | Zeitaufwand                         | `tk-zeitaufwand`                      |
| `staffNeed`      | Personalbedarf                      | `tk-personalbedarf`                   |
| `materialNeed`   | Sachmittelbedarf                    | `tk-sachmittelbedarf`                 |
| `investmentNeed` | Investitionsmittelbedarf            | `tk-investitionsmittelbedarf`         |
| `transferability`| Transferierbarkeit                  | `tk-transferierbarkeit`               |

### Extension namespace (`ext:tk:*`)

The following transferkiosk-specific facets are not in NIP-DIDACTIC core
and use the ext namespace:

| Facet                       | Source                           |
|-----------------------------|----------------------------------|
| `ext:tk:projectCyclePhase`  | Projektzyklusphase               |
| `ext:tk:qualityDimension`   | Qualitätsdimension               |
| `ext:tk:qualityDevelopment` | Qualitätsentwicklung             |
| `ext:tk:evaluationLevel`    | Evaluationsebene                 |
| `ext:tk:evaluationCriterion`| Evaluationskriterium             |
| `ext:tk:evaluationMethod`   | Evaluationsverfahren             |
| `ext:tk:fachbezug`          | Fachbezug                        |

Each follows the flat-concept-triple grammar (`ext:tk:<facet>:id`,
`ext:tk:<facet>:prefLabel:<lang>`, `ext:tk:<facet>:type`).

### Relations

- `["a", "30143:<pub>:<d>", "<relay>", "isPartOf"]` — REQUIRED, to parent Projekt.

## Kind 30145 — Publikation

A bibliographic record for a scholarly output, typically an output of a
Projekt.

### Required tags

| Tag          | Notes                                                              |
|--------------|--------------------------------------------------------------------|
| `d`          | Stable identifier. Prefer the DOI URI when present, else source URL. |
| `type`       | `"ScholarlyArticle"` (or `"Book"`, `"Chapter"`, etc).              |
| `name`       | Publication title.                                                 |
| `description`| Abstract (Kurzbeschreibung). Also duplicated into `content`.       |

### Optional tags

| Tag                       | Source                                            |
|---------------------------|---------------------------------------------------|
| `inLanguage`              | BCP47 code.                                       |
| `datePublished`           | ISO 8601; if only year is known, use `YYYY`.      |
| `publicationType:id` etc. | Publikationsart (concept triple, vocab `tk-publikationsart`). |
| `publisher:name`          | Verlag.                                           |
| `publicationLocation`     | Ort.                                              |
| `pageRange`               | Umfang / Seitenzahlen.                            |
| `author:name`             | Author display name (when no Nostr identity).     |
| `author:id`               | ORCID URI when known.                             |
| `author:type`             | `"Person"`.                                       |
| `p` (role `"author"`)     | Nostr-identified authors.                         |
| `editor:name` / `editor:id` / `editor:type` | Herausgeber:innen.              |
| `license:id`              | License URI (CC URI when applicable).             |
| `r`                       | DOI, source URL.                                  |
| `i`                       | DOI URI, ISBN, …                                  |

### Relations

- `["a", "30143:<pub>:<d>", "<relay>", "isOutputOf"]` — to parent Projekt.
- `["a", "30144:<pub>:<d>", "<relay>", "documents"]` — OPTIONAL, to a
  specific Maßnahme the publication reports on.

## Querying

### NIP-01 filter examples

```jsonc
// all projects
{"kinds": [30143]}

// project by its addressable coordinate
{"kinds": [30143], "authors": ["<pubkey>"], "#d": ["https://transferkiosk.net/p/101553"]}

// all measures belonging to a specific project
{
  "kinds": [30144],
  "#a": ["30143:<pubkey>:https://transferkiosk.net/p/101553"]
}

// all publications that came out of any project by this publisher
{"kinds": [30145], "authors": ["<pubkey>"]}

// projects with a specific subject area
{"kinds": [30143], "#about:id": ["https://transferkiosk.net/vocab/tk-faechergruppen/100004"]}

// measures targeting a specific audience
{"kinds": [30144], "#audience:id": ["https://transferkiosk.net/vocab/tk-zielgruppen/100000"]}
```

### nak CLI examples

```sh
PUB=<publisher-pubkey-hex>

# fetch a project
nak req -k 30143 -a $PUB -d https://transferkiosk.net/p/101553 wss://relay.edufeed.org

# all measures of a project
nak req -k 30144 -a $PUB --tag a=30143:$PUB:https://transferkiosk.net/p/101553 wss://relay.edufeed.org

# publications by subject (DOI)
nak req -k 30145 --tag i=https://doi.org/10.3224/gender.v17i3.02 wss://relay.edufeed.org
```

## Reverse Conversion (Nostr event → JSON-LD-ish)

Mirrors NIP-AMB's reverse-conversion section:

1. Extract `tags` array.
2. Group flat tags by prefix; reassemble nested objects via `:`.
3. Duplicate-key tags become array elements (order preserved).
4. `d` tag → `id`.
5. `t` tags → `keywords` array.
6. `r` tags → out-of-band see-also references (no JSON-LD equivalent).
7. `p` tags with role → `creator`/`contributor`/`author`/`participant`,
   resolved via kind:0 profiles.
8. `a` tags with marker `isPartOf`/`hasPart`/`isOutputOf`/`documents` →
   `isPartOf`/`hasPart`/`isOutputOf`/`documents` properties whose `id` is a
   `nostr:naddr1…` URI (NIP-19 / NIP-21).
9. `a` tags with marker matching a flattened-concept facet → resolve to
   the referenced `kind:39738` event for label translation, then merge
   into the concept object under `<facet>` alongside the `:id/:prefLabel`
   triple.
10. `ext:<ns>:<facet>:<sub>` tags → sibling `ext.<ns>.<facet>` array of
    concept objects, exactly as in NIP-AMB. Producers MUST NOT fold ext
    into core properties on reverse conversion.

## Reference Implementations

- [transferkiosk crawler + converter](https://git.edufeed.org/edufeed/edufeed-data/src/branch/master/transferkiosk)
  — Python reference: `crawl.py` (raw fetch), `extract-vocabs.py`,
  `publish-vocabs.py` (NIP-VOCAB publish), `convert.py` (raw → 30143/30144/30145).
- [edufeed amb-relay](https://git.edufeed.org/edufeed/amb-relay) — generic
  Nostr relay that already supports addressable events in this range.

## References

- [NIP-AMB](AMB.md) — flattening grammar and concept triple inherited here.
- [NIP-VOCAB](VOCAB.md) — kinds `39737`/`39738`/`39739` for published vocabs.
- [NIP-01](01.md) — addressable events.
- [NIP-19](19.md) — `nprofile`/`naddr` encodings for reverse conversion.
- [NIP-21](21.md) — `nostr:` URI prefix.
- [NIP-65](65.md) — relay discovery for `p`-tag relay hints.
