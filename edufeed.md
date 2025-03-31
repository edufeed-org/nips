# Edufeed: AMB-NIP

## Abstract

This NIP defines how to handle the metadata profile ["Allgemeines Metadatenprofil für Bildugnsressourcen" (AMB)](https://dini-ag-kim.github.io/amb/latest/) in nostr:

- How to convert AMB metadata to an AMB nostr event
- How to convert an AMB nostr-event to AMB metadata
- How to query for AMB nostr-events in supporting relays

## Event Kind

This NIP defines `kind:30142` as an AMB Metadata Event.
This means this is a replacable event, that can be adressed using `kind:pubkey:d-tag`.

## How to convert AMB metadata *to* an AMB nostr event

The transformation is quite straightforward.
For the attributes of the AMB we use tags.

For attributes expecting values of controlled vocabularies, we use this scheme:

`[<attribute_name>, <id> <prefLabel>, <languageCode>, <type>]`

For properties that expect arrays, each value is represented as a separate tag with the same key.

This is how we convert each property of the AMB:

General:

- `id`: `["d", <id>]` (we use nostr's `d`-tag here as identifier)
- `type`: `["type", <type1>, <type2>, ...]`
- `name`: `["name", <name>]`
- `description`: `["description", <description>, <languageCode>]` (language optional, not specified in AMB)
- `about`: `["about", <id>, <prefLabel>, <language>,]`
- `keywords`: `["keywords", <keyword1>, <keyword2>, ...]`
- `inLanguage`: `["inLanguage", <languageCode1>, <languageCode2>, ...]`
- `image`: `["image", <image>]`
- `trailer`: `["image", <contentUrl>, <type>, <encodingFormat>, <contentSize>, <sha256>, <embedUrl>, <bitRate>]`

Provenience:

- `creator`: `["creator", <id>, <name>, <type>, <affiliationName>, <affiliationType>, <affiliationId>]`
- `contributor`: `["contributor", <id>, <name>, <type>, <affiliationName>, <affiliationType>, <affiliationId>]`
- `dateCreated`: `["dateCreated", <ISO8601Date>]`
- `datePublished`: `["datePublished", <ISO8601Date>]`
- `dateModified`: `["dateModified", <ISO8601Date>]`
- `publisher`: `["publisher", <id>, <name>, <type>]`
- `funder`: `["funder", <id>, <name>, <type>]`

Costs And Rights:

- `isAccessibleForFree`: `["isAccessibleForFree", <isAccessibleForFree>]`
- `license`: `["license", <license_uri>, <license_shortName>]` (`license_shortName` optional, not specified in AMB)
- `conditionsOfAccess`: `["conditionsOfAccess", <id>, <prefLabel>,, <language> ]`

Educational:

- `learningResourceType`: `["learningResourceType", <id>, <label>, <language>]`
- `audience`: `["audience", <id>, <prefLabel>, <language>]`
- `teaches`: `["teaches", <id>, <prefLabel>, <language>]`
- `assesses`: `["assesses", <id>, <prefLabel>, <language>]`
- `competencyRequired`: `["competencyRequired", <id>, <prefLabel>, <language>]`
- `educationalLevel`: `["educationalLevel", <id>, <prefLabel>, <language>]`
- `interactivityType`: `["interactivityType", <id>, <prefLabel>, <language>]`

Relations:

- `isBasedOn` `["isBasedOn", <id>, <name>]` (additional attributes unsupported right now)
- `isPartOf`: `["isPartOf", <id>, <name>, <type>]`
- `hasPart`: `["hasPart", <id>, <name>, <type>]`

Meta-Metadata:

- unsupported right now

Technical:

- `duration`: `["duration", <duration>]`
- `encoding` (unsupported right now)
- `caption`: `["caption", <id>, <encodingFormat>, <language>, <type>]`


## How to convert an AMB nostr-event to AMB metadata. 

TODO

## How to query for AMB nostr-events in supporting relays.

TODO

## Examples

### Example 1 

```json
{
  "kind": 30142,
  "id": "6ba638a3786cfce89af1702a36c59e0bd9206863afa5cb6b1299aaf0d9f48c84",
  "pubkey": "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
  "created_at": 1743419457,
  "tags": [
    [
      "name",
      "hello world"
    ],
    [
      "description",
      "noch eine beschreibung",
      "de"
    ],
    [
      "d",
      "oersi.org/resources/aHR0cHM6Ly9hdi50aWIuZXUvbWVkaWEvNjY5ODM\\=11"
    ],
    [
      "about",
      "http://w3id.org/kim/schulfaecher/s1017",
      "Mathematik",
      "de"
    ],
    [
      "about",
      "http://w3id.org/kim/schulfaecher/s1005",
      "Deutsch",
      "de"
    ],
    [
      "learningResourceType",
      "http://w3id.org/openeduhub/vocabs/new_lrt/7a6e9608-2554-4981-95dc-47ab9ba924de",
      "Video",
      "de"
    ],
    [
      "keywords",
      "Pythagoras",
      "Geometrie",
      "neu"
    ],
    [
      "inLanguage",
      "de"
    ]
  ],
  "content": "",
  "sig": "6b0b78d56dea322864d35ea3b6d7e892d0e62bed96cd11ecb27d6c1d0b6d0cd68cd9ec82419946a5fb3c8d4a21eca88c9a5dad47a3b3e466ba18787224a613ef"
}
```

## Tools

To create "Example 1" you could for example use [`nak`](https://github.com/fiatjaf/nak)

```bash
nak event \
  --k 30142 \
  --t name="hello world" \
  --t description="noch eine beschreibung;de" \
  --t d="oersi.org/resources/aHR0cHM6Ly9hdi50aWIuZXUvbWVkaWEvNjY5ODM\=11" \
  --t about="http://w3id.org/kim/schulfaecher/s1017;Mathematik;de" \
  --t about="http://w3id.org/kim/schulfaecher/s1005;Deutsch;de" \
  --t learningResourceType="http://w3id.org/openeduhub/vocabs/new_lrt/7a6e9608-2554-4981-95dc-47ab9ba924de;Video;de" \
  --t keywords="Pythagoras;Geometrie;neu" \
  --tag inLanguage="de"
```

