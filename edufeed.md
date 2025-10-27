# Edufeed: AMB-NIP

## Abstract

This NIP defines how to handle the metadata profile ["Allgemeines Metadatenprofil für Bildungsressourcen" (AMB)](https://dini-ag-kim.github.io/amb/latest/) in nostr:

- How to convert AMB metadata to an AMB nostr event
- How to convert an AMB nostr-event to AMB metadata
- How to query for AMB nostr-events in supporting relays

## Event Kind

This NIP defines `kind:30142` as an AMB Metadata Event.
This means this is a replaceable event, that can be addressed using `kind:pubkey:d-tag`.

## How to convert AMB metadata *to* an AMB nostr event

The transformation uses JSON-flattening with `:` as the delimiter to convert nested AMB metadata structures into flat Nostr tags. Additionally, Nostr-native tag conventions are used where applicable for better interoperability and query efficiency.

### Nostr-Native Conventions

This NIP follows Nostr conventions where they align with AMB requirements:

- **`t` tags**: Used for keywords/topics (instead of flattened `keywords` tags)
- **`p` tags**: Used for creator/contributor references when the creator has a Nostr identity (pubkey), with fallback to flattened structure for non-Nostr identifiers
- **`a` tags**: Used for references to other addressable events on Nostr (including other AMB events), with fallback to flattened URIs for external resources

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

**General:**

- `id` → `["d", <id>]` (special case: use Nostr's `d` tag as identifier)
- `type` → `["type", <value>]` (repeat for multiple types)
- `name` → `["name", <value>]`
- `description` → `["description", <value>]`
- `about` (array of concept objects) → Repeat for each:
  - `["about:id", <uri>]`
  - `["about:prefLabel", <label>]`
  - `["about:type", "Concept"]`
  - `["about:inLanguage", <language>]` (if applicable)
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

**Provenance:**

- `creator` (array of Person/Organization objects) → Repeat for each:
  - **Nostr-native (if creator has Nostr pubkey)**: `["p", <npub-hex>, <relay>, "creator"]`
  - **Fallback (for non-Nostr identifiers)**:
    - `["creator:id", <uri>]` (optional, e.g., ORCID, GND)
    - `["creator:name", <name>]`
    - `["creator:type", <"Person"|"Organization">]`
    - `["creator:honorificPrefix", <title>]` (optional, for persons)
    - `["creator:affiliation:id", <uri>]` (optional)
    - `["creator:affiliation:name", <name>]` (optional)
    - `["creator:affiliation:type", "Organization"]` (optional)
- `contributor` (array of Person/Organization objects) → Same structure as `creator`
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

**Costs and Rights:**

- `isAccessibleForFree` → `["isAccessibleForFree", <"true"|"false">]`
- `license` (object) →
  - `["license:id", <license_uri>]`
- `conditionsOfAccess` (Concept object) →
  - `["conditionsOfAccess:id", <uri>]`
  - `["conditionsOfAccess:prefLabel", <label>]` (optional)
  - `["conditionsOfAccess:type", "Concept"]` (optional)
  - `["conditionsOfAccess:inLanguage", <language>]` (optional)

**Educational:**

- `learningResourceType` (array of Concept objects) → Repeat for each:
  - `["learningResourceType:id", <uri>]`
  - `["learningResourceType:prefLabel", <label>]` (optional)
  - `["learningResourceType:type", "Concept"]` (optional)
  - `["learningResourceType:inLanguage", <language>]` (optional)
- `audience` (array of Concept objects) → Repeat for each:
  - `["audience:id", <uri>]`
  - `["audience:prefLabel", <label>]` (optional)
  - `["audience:type", "Concept"]` (optional)
  - `["audience:inLanguage", <language>]` (optional)
- `teaches` (array of Concept objects) → Repeat for each:
  - `["teaches:id", <uri>]`
  - `["teaches:prefLabel", <label>]` (optional)
- `assesses` (array of Concept objects) → Repeat for each:
  - `["assesses:id", <uri>]`
  - `["assesses:prefLabel", <label>]` (optional)
- `competencyRequired` (array of Concept objects) → Repeat for each:
  - `["competencyRequired:id", <uri>]`
  - `["competencyRequired:prefLabel", <label>]` (optional)
- `educationalLevel` (array of Concept objects) → Repeat for each:
  - `["educationalLevel:id", <uri>]`
  - `["educationalLevel:prefLabel", <label>]` (optional)
  - `["educationalLevel:type", "Concept"]` (optional)
  - `["educationalLevel:inLanguage", <language>]` (optional)
- `interactivityType` (Concept object) →
  - `["interactivityType:id", <uri>]`
  - `["interactivityType:prefLabel", <label>]` (optional)
  - `["interactivityType:type", "Concept"]` (optional)
  - `["interactivityType:inLanguage", <language>]` (optional)

**Relations:**

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

**Meta-Metadata:**

- `mainEntityOfPage` (array of WebPage objects) → Repeat for each:
  - `["mainEntityOfPage:id", <uri>]`
  - `["mainEntityOfPage:type", "WebContent"]`
  - `["mainEntityOfPage:provider:id", <uri>]` (optional)
  - `["mainEntityOfPage:provider:name", <name>]` (optional)
  - `["mainEntityOfPage:provider:type", <type>]` (optional)
  - `["mainEntityOfPage:dateCreated", <ISO8601Date>]` (optional)
  - `["mainEntityOfPage:dateModified", <ISO8601Date>]` (optional)

**Technical:**

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

## Conversion Guidelines

### AMB → Nostr Event (JavaScript)

```javascript
function flattenAMBToNostrTags(ambData) {
  const tags = [];
  
  function addTag(key, value) {
    if (value !== null && value !== undefined && value !== '') {
      tags.push([key, String(value)]);
    }
  }
  
  function flattenObject(obj, prefix = '') {
    for (const [key, value] of Object.entries(obj)) {
      const fullKey = prefix ? `${prefix}:${key}` : key;
      
      // Special case: id maps to 'd' tag
      if (key === 'id' && !prefix) {
        addTag('d', value);
        continue;
      }
      
      // Special case: keywords map to 't' tags (Nostr-native)
      if (key === 'keywords' && !prefix && Array.isArray(value)) {
        value.forEach(keyword => {
          addTag('t', keyword);
        });
        continue;
      }
      
      if (Array.isArray(value)) {
        // Arrays: repeat the same key for each element
        value.forEach(item => {
          if (typeof item === 'object' && item !== null) {
            flattenObject(item, fullKey);
          } else {
            addTag(fullKey, item);
          }
        });
      } else if (typeof value === 'object' && value !== null) {
        // Nested objects: flatten recursively
        flattenObject(value, fullKey);
      } else {
        // Simple values
        addTag(fullKey, value);
      }
    }
  }
  
  flattenObject(ambData);
  return tags;
}

// Example usage:
const ambMetadata = {
  id: "https://example.org/resource/123",
  name: "Introduction to Physics",
  creator: [
    {
      name: "Dr. Jane Smith",
      type: "Person",
      affiliation: {
        name: "MIT",
        type: "Organization"
      }
    }
  ],
  keywords: ["Physics", "Education"]
};

const tags = flattenAMBToNostrTags(ambMetadata);
// Result (using Nostr-native conventions):
// [
//   ["d", "https://example.org/resource/123"],
//   ["name", "Introduction to Physics"],
//   ["creator:name", "Dr. Jane Smith"],
//   ["creator:type", "Person"],
//   ["creator:affiliation:name", "MIT"],
//   ["creator:affiliation:type", "Organization"],
//   ["t", "Physics"],
//   ["t", "Education"]
// ]
//
// Note: If creator had Nostr identity, would also include:
// ["p", "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798", "wss://relay.example.com", "creator"]
```

### Nostr Event → AMB Metadata (JavaScript)

```javascript
function unflattenNostrTagsToAMB(tags) {
  const result = {};
  
  // Group tags by their key
  const tagGroups = {};
  const pTags = [];  // Collect p tags separately
  
  tags.forEach(([key, ...values]) => {
    // Special case: 'd' tag maps to 'id'
    if (key === 'd') {
      result.id = values[0];
      return;
    }
    
    // Special case: 'p' tags with 'creator' marker map to creator.id
    if (key === 'p') {
      const [pubkey, relay, marker] = values;
      if (marker === 'creator' || marker === 'contributor') {
        pTags.push({ pubkey, relay, marker });
      }
      return;
    }
    
    // Special case: 't' tags map to 'keywords'
    if (key === 't') {
      if (!result.keywords) {
        result.keywords = [];
      }
      result.keywords.push(values[0]);
      return;
    }
    
    if (!tagGroups[key]) {
      tagGroups[key] = [];
    }
    tagGroups[key].push(values);
  });
  
  // Process p tags (creator/contributor)
  if (pTags.length > 0) {
    result.creator = pTags.map(({ pubkey }) => ({
      id: pubkey
    }));
  }
  
  // Process each tag group
  for (const [key, valuesList] of Object.entries(tagGroups)) {
    const parts = key.split(':');
    
    // Multiple occurrences of same key = array
    if (valuesList.length > 1) {
      // Check if this is an object array (has nested properties)
      const hasNestedProps = Object.keys(tagGroups).some(k => 
        k.startsWith(key + ':')
      );
      
      if (hasNestedProps) {
        // Array of objects - reconstruct each object
        const arrayResult = [];
        valuesList.forEach(values => {
          const obj = {};
          setNestedValue(obj, parts, values[0]);
          arrayResult.push(obj);
        });
        setNestedValue(result, parts, arrayResult);
      } else {
        // Simple array
        setNestedValue(result, parts, valuesList.map(v => v[0]));
      }
    } else {
      // Single occurrence
      setNestedValue(result, parts, valuesList[0][0]);
    }
  }
  
  return result;
}

function setNestedValue(obj, parts, value) {
  const lastPart = parts[parts.length - 1];
  let current = obj;
  
  for (let i = 0; i < parts.length - 1; i++) {
    if (!current[parts[i]]) {
      current[parts[i]] = {};
    }
    current = current[parts[i]];
  }
  
  current[lastPart] = value;
}

// Example usage:
const nostrTags = [
  ["d", "https://example.org/resource/123"],
  ["name", "Introduction to Physics"],
  ["p", "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798", "wss://relay.example.com", "creator"],
  ["creator:affiliation:name", "MIT"],
  ["creator:affiliation:type", "Organization"],
  ["t", "Physics"],
  ["t", "Education"]
];

const ambMetadata = unflattenNostrTagsToAMB(nostrTags);
// Result:
// {
//   id: "https://example.org/resource/123",
//   name: "Introduction to Physics",
//   creator: {
//     // Note: p tag is decoded to get the pubkey
//     id: "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
//     affiliation: {
//       name: "MIT",
//       type: "Organization"
//     }
//   },
//   keywords: ["Physics", "Education"]
// }
```

## How to convert an AMB nostr-event to AMB metadata

To convert a Nostr event back to AMB metadata:

1. **Extract tags**: Get the `tags` array from the Nostr event
2. **Group by prefix**: Collect all tags that share the same prefix (before the first `:`)
3. **Reconstruct nesting**: Use the `:` delimiter to rebuild nested object structure
4. **Handle arrays**: Multiple tags with identical keys become array elements
5. **Preserve order**: Array order is determined by tag order in the event
6. **Special mappings**: 
   - `d` tag → `id` property
   - Convert string booleans to actual booleans
   - Parse ISO8601 dates if needed for validation

### Reconstruction Algorithm

```javascript
function reconstructAMB(nostrEvent) {
  const amb = {
    "@context": [
      "https://w3id.org/kim/amb/context.jsonld",
      { "@language": "de" }
    ],
    type: ["LearningResource"]
  };
  
  // Group tags by their base key (before first ':')
  const tagsByBase = {};
  nostrEvent.tags.forEach(tag => {
    const [key, ...values] = tag;
    const baseKey = key.split(':')[0];
    
    if (!tagsByBase[baseKey]) {
      tagsByBase[baseKey] = [];
    }
    tagsByBase[baseKey].push({ fullKey: key, values });
  });
  
  // Reconstruct each property
  for (const [baseKey, tags] of Object.entries(tagsByBase)) {
    if (baseKey === 'd') {
      amb.id = tags[0].values[0];
      continue;
    }
    
    // Check if this is a simple property or nested
    const hasNested = tags.some(t => t.fullKey.includes(':'));
    
    if (!hasNested) {
      // Simple property
      if (tags.length === 1) {
        amb[baseKey] = tags[0].values[0];
      } else {
        // Multiple values = array
        amb[baseKey] = tags.map(t => t.values[0]);
      }
    } else {
      // Nested property - group by instance
      amb[baseKey] = reconstructNestedProperty(tags);
    }
  }
  
  return amb;
}

function reconstructNestedProperty(tags) {
  // Group tags that belong to the same array instance
  const instances = [];
  let currentInstance = {};
  let lastDepth = 0;
  
  tags.forEach(tag => {
    const parts = tag.fullKey.split(':');
    const depth = parts.length;
    
    // If depth decreased, we're starting a new instance
    if (depth <= lastDepth && Object.keys(currentInstance).length > 0) {
      instances.push(currentInstance);
      currentInstance = {};
    }
    
    // Build nested structure
    let current = currentInstance;
    for (let i = 1; i < parts.length - 1; i++) {
      if (!current[parts[i]]) {
        current[parts[i]] = {};
      }
      current = current[parts[i]];
    }
    current[parts[parts.length - 1]] = tag.values[0];
    
    lastDepth = depth;
  });
  
  if (Object.keys(currentInstance).length > 0) {
    instances.push(currentInstance);
  }
  
  return instances.length === 1 ? instances[0] : instances;
}
```

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
    ["about:prefLabel", "Mathematik"],
    ["about:type", "Concept"],
    ["about:inLanguage", "de"],
    ["about:id", "http://w3id.org/kim/schulfaecher/s1005"],
    ["about:prefLabel", "Deutsch"],
    ["about:type", "Concept"],
    ["about:inLanguage", "de"],
    ["learningResourceType:id", "http://w3id.org/openeduhub/vocabs/new_lrt/7a6e9608-2554-4981-95dc-47ab9ba924de"],
    ["learningResourceType:prefLabel", "Video"],
    ["learningResourceType:type", "Concept"],
    ["learningResourceType:inLanguage", "de"],
    ["t", "Pythagoras"],
    ["t", "Geometrie"],
    ["t", "Mathematik"],
    ["inLanguage", "de"],
    ["license:id", "https://creativecommons.org/licenses/by/4.0/"],
    ["isAccessibleForFree", "true"]
  ],
  "content": "",
  "sig": "6b0b78d56dea322864d35ea3b6d7e892d0e62bed96cd11ecb27d6c1d0b6d0cd68cd9ec82419946a5fb3c8d4a21eca88c9a5dad47a3b3e466ba18787224a613ef"
}
```

### Example 2: Resource with Creator and Affiliation

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
    ["creator:id", "https://orcid.org/0000-0001-2345-6789"],
    ["creator:name", "Dr. Jane Smith"],
    ["creator:type", "Person"],
    ["creator:honorificPrefix", "Dr."],
    ["creator:affiliation:id", "https://ror.org/example123"],
    ["creator:affiliation:name", "Massachusetts Institute of Technology"],
    ["creator:affiliation:type", "Organization"],
    ["p", "2c4b52e2a4f7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c", "wss://relay.example.com", "creator"],
    ["creator:id", "https://orcid.org/0000-0009-8765-4321"],
    ["creator:name", "Prof. John Doe"],
    ["creator:type", "Person"],
    ["creator:honorificPrefix", "Prof."],
    ["creator:affiliation:name", "Stanford University"],
    ["creator:affiliation:type", "Organization"],
    ["dateCreated", "2024-01-15"],
    ["datePublished", "2024-02-01"],
    ["about:id", "https://w3id.org/kim/hochschulfaechersystematik/n079"],
    ["about:prefLabel", "Informatik"],
    ["about:type", "Concept"],
    ["learningResourceType:id", "https://w3id.org/kim/hcrt/course"],
    ["learningResourceType:prefLabel", "Course"],
    ["audience:id", "http://purl.org/dcx/lrmi-vocabs/educationalAudienceRole/student"],
    ["audience:prefLabel", "student"],
    ["audience:type", "Concept"],
    ["educationalLevel:id", "https://w3id.org/kim/educationalLevel/level_06"],
    ["educationalLevel:prefLabel", "Bachelor or equivalent"],
    ["inLanguage", "en"],
    ["license:id", "https://creativecommons.org/licenses/by-sa/4.0/"],
    ["isAccessibleForFree", "true"]
  ],
  "content": "",
  "sig": "8d1c89f5da33ec9a2b456def78a90b1cd23e456f78a90b12cd34e567f89a012b34c56d78e9f0a12bc3d45e6f78901a23b45c67d89e0f1a2b3c4d5e6f7890123a"
}
```

## Tools

### Decoding Nostr Identifiers with njump

[`njump`](https://njump.me/) is a helpful tool for decoding and navigating Nostr identifiers, particularly useful when working with `p` tags (pubkeys) and `a` tags (addressable events):

- **For `p` tag pubkeys**: Paste the hex pubkey or npub identifier at `https://njump.me/<pubkey-or-npub>` to view the profile and see all resources published by that user
  - Example: `https://njump.me/79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`
  - Or: `https://njump.me/npub1...`

- **For `a` tag addressable events**: Use `https://njump.me/naddr1...` to view the specific event
  - Example: `https://njump.me/naddr1qqqgcjsyup9qqqg6cjv` to navigate to an addressable resource

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
  --tag about:prefLabel="Mathematik" \
  --tag about:type="Concept" \
  --tag about:inLanguage="de" \
  --tag t="Pythagoras" \
  --tag t="Geometrie" \
  --tag inLanguage="de" \
  --tag license:id="https://creativecommons.org/licenses/by/4.0/"

# Example 2: Resource with creator (with Nostr identity p tag)
nak event \
  --kind 30142 \
  --tag d="https://example.org/resource/456" \
  --tag name="Physics Course" \
  --tag p="79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798;wss://relay.example.com;creator" \
  --tag creator:affiliation:name="MIT" \
  --tag creator:affiliation:type="Organization"

# Example 3: Resource with relation to another AMB event (using a tag)
nak event \
  --kind 30142 \
  --tag d="https://example.org/resource/789" \
  --tag name="Physics Module Part 1" \
  --tag a="30142:79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798:physics-course;wss://relay.example.com;isPartOf"
```

## References

- [AMB Specification](https://dini-ag-kim.github.io/amb/latest/)
- [Nostr Protocol (NIP-01)](https://github.com/nostr-protocol/nips/blob/master/01.md)
- [Addressable Events (NIP-33)](https://github.com/nostr-protocol/nips/blob/master/33.md)
- [JSON-Flattening Concept](https://localizely.com/json-flattener/)
