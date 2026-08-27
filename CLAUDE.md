# Edufeed NIP Project

## Overview

This project defines a Nostr Implementation Possibility (NIP) for integrating the **AMB (Allgemeines Metadatenprofil für Bildungsressourcen)** metadata standard into Nostr. AMB is a German educational metadata profile for describing teaching and learning resources, based on schema.org and LRMI.

## Key Files

- `AMB.md` - NIP-AMB, the main specification (kind `30142`)
- `VOCAB.md` - NIP-VOCAB, controlled vocabularies (kinds `39734`-`39739`)
- `DIDACTIC.md` - NIP-DIDACTIC, funded projects and teaching measures (kinds `30143`, `30144`)
- `BREAKING.md` - breaking changes to the edufeed NIPs (upstream deleted its copy; ours is kept)
- `.forgejo/workflows/publish-nip.yml` - publishes these NIPs to Nostr as `kind:30817`

This repository is a fork of `nostr-protocol/nips`. Our contributions are purely additive:
the files above, and nothing else. Sync with `git fetch upstream && git merge upstream/master`;
the only recurring conflict is `BREAKING.md`, which upstream deleted and we keep.

## Event Kind

- **Kind 30142** - AMB Metadata Event (addressable/parameterized replaceable)

## Core Concepts

### AMB to Nostr Conversion

The NIP uses JSON-flattening with `:` delimiter to convert nested AMB structures to flat Nostr tags:
- `{"creator": {"name": "John"}}` → `["creator:name", "John"]`

### Nostr-Native Mappings

- `id` → `d` tag (addressable event identifier)
- `keywords` → `t` tags (hashtags)
- Creator with Nostr identity → `p` tag with "creator" marker
- References to other AMB events → `a` tags with relationship markers

### AMB Property Categories

1. **General**: id, type, name, description, about, keywords, inLanguage, image, trailer
2. **Provenance**: creator, contributor, dateCreated, datePublished, dateModified, publisher, funder
3. **Rights**: isAccessibleForFree, license, conditionsOfAccess
4. **Educational**: learningResourceType, audience, teaches, assesses, competencyRequired, educationalLevel, interactivityType
5. **Relations**: isBasedOn, isPartOf, hasPart
6. **Technical**: duration, encoding, caption
7. **Meta-Metadata**: mainEntityOfPage

## Integration Points with Nostr

### Tags Used
- `d` - Event identifier (maps to AMB `id`)
- `t` - Keywords/hashtags
- `p` - Creator/contributor references (when they have Nostr pubkeys), format: `["p", <pubkey>, <relay-hint>, <role>]`
- `a` - References to other addressable events (isBasedOn, isPartOf, hasPart), format: `["a", "30142:<pubkey>:<d>", <relay-hint>, <relationship>]`
- `r` - External URL references (original source, DOI, ISBN, related web resources)

### Content Field
- `content` SHOULD contain the description text for client compatibility
- `description` tag is kept for relay queryability (duplicated)

### Custom Flattened Tags
All AMB properties use `:` delimiter for nested structures:
- `about:id`, `about:prefLabel:de`, `about:type`
- `creator:name`, `creator:affiliation:name`
- `license:id`
- `learningResourceType:id`, `learningResourceType:prefLabel:de`

## Controlled Vocabularies (External)

- **Subjects**: Hochschulfächersystematik, Schulfächerliste
- **Resource Types**: HCRT, OEHRT
- **Audience**: LRMI Educational Audience Roles
- **Educational Level**: Bildungsstufen vocabulary
- **Licenses**: Creative Commons URIs

## Resolved Design Decisions

- **Relay hints**: Embedded in `p` and `a` tags (3rd position), not standalone tags
- **External references**: Use `r` tag (per NIP-24), no clash with relay list usage
- **Content field**: Description duplicated in both `content` field and `description` tag

## Open Questions / TODOs

- Relay query support for custom flattened tags (e.g., `#about:id`) - requires specialized relay
- Full-text search capabilities for `name` and `description`
- Handling array reconstruction when converting back to AMB (order-dependent parsing)

## References

- [AMB Specification](https://dini-ag-kim.github.io/amb/latest/)
- [NIP-01: Basic Protocol](https://github.com/nostr-protocol/nips/blob/master/01.md)
- [NIP-33: Addressable Events](https://github.com/nostr-protocol/nips/blob/master/33.md) (now in NIP-01)
