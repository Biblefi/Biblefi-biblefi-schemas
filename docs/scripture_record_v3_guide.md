# Scripture Record v3.0.0 Migration Guide

This guide covers what's new in `scripture_record` v3.0.0, breaking changes from v2, computation algorithms, and a full example JSON payload.

**Schema `$id`:** `https://schemas.biblefi.io/v3/scripture_record.schema.json`

---

## What's New in v3.0.0

| Feature | Description |
|---------|-------------|
| `canonical_fingerprint` | New required field: SHA3-256 + BLAKE3 + verse bitmask composite fingerprint |
| `linguistic_analysis` | New optional section: syntactic complexity, parallelism type, rhetorical devices, hapax legomena |
| `theological_graph` | New optional section: graph-theoretic concept map with nodes, edges, and centrality scores |
| `canon_authority` | New optional section: multi-tradition canonical status + textual variants + transmission chain hash |
| `defi_integration` | New optional section: tokenization status, tithe routing, impact score, stake pool |
| `translation.manuscript_tradition` | New optional field on translation object |
| `translation.original_language` | New optional field on translation object |
| `hermeneutics.typology` | New typology sub-object with type_text and antitype_text |
| `hermeneutics.original_language_notes[].morphology` | New optional morphology field per word |
| `ai_metadata.lexical_density` | New optional AI metric |
| `ai_metadata.intertextuality_score` | New optional AI metric |
| `cross_references[].confidence_score` | New optional numeric confidence |

---

## Breaking Changes from v2.0.0

| Field / Behavior | v2.0.0 | v3.0.0 |
|-----------------|--------|--------|
| `canonical_fingerprint` | Not present | **Required** — must be computed and included |
| `verses` type | String only | oneOf: integer \| integer[] \| {from, to} object |
| `schema_version` expected value | `"2.0.0"` | `"3.0.0"` |
| `$id` | `…/v2/scripture_record.schema.json` | `…/v3/scripture_record.schema.json` |
| `attestations` | Object (singular) | Array of attestation objects |
| `hermeneutics.typology` | Not present | Optional nested object with additionalProperties: false |

---

## `content_hash` Computation

The `content_hash` is the **Keccak-256** hash of the NFC-normalized, whitespace-trimmed UTF-8 encoding of the scripture text.

### Algorithm (step by step)

1. Take the raw `text` string.
2. Apply **Unicode NFC normalization** (`String.normalize('NFC')` in JS, `unicodedata.normalize('NFC', s)` in Python).
3. **Trim** leading and trailing whitespace.
4. **Encode** to UTF-8 bytes.
5. Compute **Keccak-256** of the bytes.
6. Hex-encode with `0x` prefix.

### JavaScript

```js
import { keccak256, toUtf8Bytes } from 'ethers';

function computeContentHash(text) {
  const normalized = text.normalize('NFC');
  const trimmed = normalized.trim();
  return keccak256(toUtf8Bytes(trimmed));
}

const hash = computeContentHash(
  'For God so loved the world, that he gave his only begotten Son, ' +
  'that whosoever believeth in him should not perish, but have everlasting life.'
);
console.log(hash);
// 0x<64 hex chars>
```

### Python

```python
from eth_hash.auto import keccak  # pip install eth-hash[pycryptodome]
import unicodedata

def compute_content_hash(text: str) -> str:
    normalized = unicodedata.normalize('NFC', text)
    trimmed = normalized.strip()
    return '0x' + keccak(trimmed.encode('utf-8')).hex()

hash_val = compute_content_hash(
    'For God so loved the world, that he gave his only begotten Son, '
    'that whosoever believeth in him should not perish, but have everlasting life.'
)
print(hash_val)
```

---

## `canonical_fingerprint` Computation

The canonical fingerprint uniquely identifies a scripture record by combining multiple cryptographic algorithms over a canonical string.

### Canonical String Format

```
{book}:{chapter}:{verse_key}:{translation_abbreviation}:{content_hash}
```

Where `verse_key` is:
- Single verse: `"16"`
- Verse range: `"1-5"`
- Verse list: `"1,3,5"` (sorted ascending)

### JavaScript

```js
import { sha3_256 } from 'js-sha3';           // npm install js-sha3
import { blake3 } from '@noble/hashes/blake3'; // npm install @noble/hashes

function buildVerseKey(verses) {
  if (typeof verses === 'number') return String(verses);
  if (Array.isArray(verses)) return [...verses].sort((a, b) => a - b).join(',');
  return `${verses.from}-${verses.to}`;
}

function computeCanonicalFingerprint(book, chapter, verses, translationAbbr, contentHash) {
  const verseKey = buildVerseKey(verses);
  const canonicalStr = `${book}:${chapter}:${verseKey}:${translationAbbr}:${contentHash}`;

  // SHA3-256 of canonical string
  const sha3 = sha3_256(canonicalStr);

  // BLAKE3 of canonical string
  const b3 = Buffer.from(blake3(new TextEncoder().encode(canonicalStr))).toString('hex');

  // Verse bitmask: bit N-1 set for each verse N
  const verseNums = typeof verses === 'number'
    ? [verses]
    : Array.isArray(verses)
      ? verses
      : Array.from({ length: verses.to - verses.from + 1 }, (_, i) => verses.from + i);
  let mask = 0n;
  for (const v of verseNums) {
    mask |= (1n << BigInt(v - 1));
  }

  return {
    sha3_256: sha3,
    blake3: b3,
    verse_bitmask: '0x' + mask.toString(16),
    canonical_key: `${book}:${chapter}:${verseKey}:${translationAbbr}`,
    algorithm_version: '1.0'
  };
}

// Example: John 3:16 KJV
const fp = computeCanonicalFingerprint(
  'John', 3, 16, 'KJV',
  '0xabcd...ef'  // content_hash
);
console.log(fp.canonical_key);  // "John:3:16:KJV"
```

### Python

```python
import hashlib
from blake3 import blake3  # pip install blake3

def build_verse_key(verses) -> str:
    if isinstance(verses, int):
        return str(verses)
    if isinstance(verses, list):
        return ','.join(str(v) for v in sorted(verses))
    return f"{verses['from']}-{verses['to']}"

def compute_canonical_fingerprint(book, chapter, verses, translation_abbr, content_hash):
    verse_key = build_verse_key(verses)
    canonical_str = f"{book}:{chapter}:{verse_key}:{translation_abbr}:{content_hash}"
    encoded = canonical_str.encode('utf-8')

    # SHA3-256
    sha3 = hashlib.sha3_256(encoded).hexdigest()

    # BLAKE3
    b3 = blake3(encoded).hexdigest()

    # Verse bitmask
    if isinstance(verses, int):
        verse_nums = [verses]
    elif isinstance(verses, list):
        verse_nums = verses
    else:
        verse_nums = list(range(verses['from'], verses['to'] + 1))

    mask = 0
    for v in verse_nums:
        mask |= (1 << (v - 1))

    return {
        'sha3_256': sha3,
        'blake3': b3,
        'verse_bitmask': hex(mask),
        'canonical_key': f"{book}:{chapter}:{verse_key}:{translation_abbr}",
        'algorithm_version': '1.0'
    }
```

---

## `theological_graph` Usage Example

```json
{
  "theological_graph": {
    "nodes": [
      {
        "id": "gods_love",
        "concept": "The love of God",
        "concept_type": "divine_attribute",
        "weight": 1.0
      },
      {
        "id": "world",
        "concept": "The kosmos (all of humanity)",
        "concept_type": "human_condition",
        "weight": 0.8
      },
      {
        "id": "son_given",
        "concept": "The giving of the Son",
        "concept_type": "salvific_action",
        "weight": 0.95
      },
      {
        "id": "eternal_life",
        "concept": "Everlasting life",
        "concept_type": "eschatological_reality",
        "weight": 0.9
      },
      {
        "id": "belief",
        "concept": "Faith / believing",
        "concept_type": "ethical_imperative",
        "weight": 0.85
      }
    ],
    "edges": [
      {
        "source": "gods_love",
        "target": "son_given",
        "relationship": "causes",
        "strength": 1.0
      },
      {
        "source": "son_given",
        "target": "eternal_life",
        "relationship": "enables",
        "strength": 0.95
      },
      {
        "source": "belief",
        "target": "eternal_life",
        "relationship": "requires",
        "strength": 0.9
      },
      {
        "source": "gods_love",
        "target": "world",
        "relationship": "defines",
        "strength": 0.8
      }
    ],
    "centrality_scores": {
      "most_central_concept": "son_given",
      "graph_density": 0.4,
      "clustering_coefficient": 0.67
    }
  }
}
```

---

## `linguistic_analysis` Usage Example

```json
{
  "linguistic_analysis": {
    "word_count": 26,
    "unique_word_count": 22,
    "avg_word_length": 4.5,
    "hapax_legomena": ["begotten"],
    "syntactic_complexity_index": 0.42,
    "parallelism_type": "synthetic",
    "discourse_unit": "verse_unit",
    "rhetorical_devices": ["hyperbole", "metaphor"],
    "tense_distribution": {
      "past_ratio": 0.3,
      "present_ratio": 0.5,
      "future_ratio": 0.2
    }
  }
}
```

---

## Full Example: John 3:16 KJV

```json
{
  "record_id": "550e8400-e29b-41d4-a716-446655440000",
  "schema_version": "3.0.0",
  "book": "John",
  "chapter": 3,
  "verses": 16,
  "translation": {
    "abbreviation": "KJV",
    "full_name": "King James Version",
    "language": "en",
    "year_published": 1611,
    "manuscript_tradition": "textus_receptus",
    "original_language": "greek"
  },
  "text": "For God so loved the world, that he gave his only begotten Son, that whosoever believeth in him should not perish, but have everlasting life.",
  "content_hash": "0x98b0f3b378b4d5c83bc63d5a9d6dd22f1bc19044f7d175c9f04a4c20e2c5f2d1",
  "canonical_fingerprint": {
    "sha3_256": "a3f1c2e4d5b6a7e8f9012345678901234567890abcdef0123456789abcdef0123",
    "blake3": "b4e2d3f5c6a7b8e9f0123456789012345678901234567890abcdef0123456789ab",
    "verse_bitmask": "0x8000",
    "canonical_key": "John:3:16:KJV",
    "algorithm_version": "1.0"
  },
  "provenance": {
    "source_organization": "Public Domain",
    "license": "public_domain",
    "attribution_required": false
  },
  "ai_metadata": {
    "embedding_model": "text-embedding-3-large",
    "embedding_version": "3",
    "embedding_dimensions": 3072,
    "semantic_clusters": ["salvation", "divine_love", "eternal_life"],
    "sentiment_score": 0.95,
    "reading_level": {
      "flesch_kincaid_grade": 8.2,
      "flesch_reading_ease": 62.5,
      "grade_label": "middle_school"
    },
    "theological_themes": [
      { "theme": "soteriology", "confidence": 0.99 },
      { "theme": "christology", "confidence": 0.95 },
      { "theme": "covenant", "confidence": 0.7 }
    ],
    "named_entities": [
      { "name": "God", "type": "deity", "biblical_id": "yahweh" },
      { "name": "Son", "type": "person", "biblical_id": "jesus_christ" },
      { "name": "world", "type": "nation" }
    ],
    "lexical_density": 0.85,
    "intertextuality_score": 0.92
  },
  "cross_references": [
    {
      "book": "Romans",
      "chapter": 5,
      "verses": 8,
      "relationship": "parallel",
      "strength": "explicit",
      "direction": "bidirectional",
      "notes": "Paul's parallel statement: 'God demonstrates his own love for us in this: While we were still sinners, Christ died for us.'",
      "confidence_score": 0.97
    },
    {
      "book": "1 John",
      "chapter": 4,
      "verses": { "from": 9, "to": 10 },
      "relationship": "expansion",
      "strength": "explicit",
      "direction": "forward",
      "confidence_score": 0.95
    }
  ],
  "translation_graph": [
    {
      "abbreviation": "NIV",
      "language": "en",
      "text": "For God so loved the world that he gave his one and only Son, that whoever believes in him shall not perish but have eternal life.",
      "year_published": 1978,
      "semantic_similarity_to_primary": 0.97,
      "notable_differences": "Renders 'monogenes' as 'one and only' vs KJV 'only begotten'. 'believeth' → 'believes'.",
      "back_translation_divergence_score": 0.12
    },
    {
      "abbreviation": "ESV",
      "language": "en",
      "text": "For God so loved the world, that he gave his only Son, that whoever believes in him should not perish but have eternal life.",
      "year_published": 2001,
      "semantic_similarity_to_primary": 0.98,
      "back_translation_divergence_score": 0.08
    }
  ],
  "attestations": [
    {
      "chain_id": 8453,
      "contract_address": "0x1234567890123456789012345678901234567890",
      "token_id": "1",
      "tx_hash": "0xabcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789ab",
      "attestation_type": "nft_mint",
      "block_number": 12345678,
      "attested_at": "2026-01-01T00:00:00Z",
      "attested_by": "0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef"
    }
  ],
  "hermeneutics": {
    "literary_genre": "gospel",
    "speech_act": "declaration",
    "covenant_context": "new_covenant",
    "canonical_position": "nt_gospel",
    "christological_significance": "The Johannine statement of the Gospel: divine love expressed through the incarnation and atoning work of the eternal Son of God, offering universal salvation through faith.",
    "interpretive_traditions": ["reformed", "catholic", "eastern_orthodox", "arminian"],
    "original_language_notes": [
      {
        "word": "ἠγάπησεν",
        "language": "greek",
        "transliteration": "ēgapēsen",
        "strongs_id": "G25",
        "gloss": "loved",
        "semantic_domain": "love, affection",
        "morphology": "verb:aorist:active:indicative:3singular"
      },
      {
        "word": "μονογενῆ",
        "language": "greek",
        "transliteration": "monogenē",
        "strongs_id": "G3439",
        "gloss": "only begotten / one and only",
        "semantic_domain": "uniqueness, divine sonship",
        "morphology": "adjective:accusative:masculine:singular"
      }
    ]
  },
  "linguistic_analysis": {
    "word_count": 26,
    "unique_word_count": 22,
    "avg_word_length": 4.5,
    "hapax_legomena": ["begotten"],
    "syntactic_complexity_index": 0.42,
    "parallelism_type": "synthetic",
    "discourse_unit": "verse_unit",
    "rhetorical_devices": ["hyperbole", "metaphor"],
    "tense_distribution": {
      "past_ratio": 0.3,
      "present_ratio": 0.5,
      "future_ratio": 0.2
    }
  },
  "theological_graph": {
    "nodes": [
      { "id": "gods_love", "concept": "The love of God", "concept_type": "divine_attribute", "weight": 1.0 },
      { "id": "world", "concept": "The kosmos (humanity)", "concept_type": "human_condition", "weight": 0.8 },
      { "id": "son_given", "concept": "Giving of the Son (Incarnation)", "concept_type": "salvific_action", "weight": 0.95 },
      { "id": "belief", "concept": "Faith / believing", "concept_type": "ethical_imperative", "weight": 0.85 },
      { "id": "eternal_life", "concept": "Everlasting life", "concept_type": "eschatological_reality", "weight": 0.9 }
    ],
    "edges": [
      { "source": "gods_love", "target": "son_given", "relationship": "causes", "strength": 1.0 },
      { "source": "son_given", "target": "eternal_life", "relationship": "enables", "strength": 0.95 },
      { "source": "belief", "target": "eternal_life", "relationship": "requires", "strength": 0.9 },
      { "source": "gods_love", "target": "world", "relationship": "defines", "strength": 0.8 }
    ],
    "centrality_scores": {
      "most_central_concept": "son_given",
      "graph_density": 0.4,
      "clustering_coefficient": 0.67
    }
  },
  "canon_authority": {
    "protestant_canon": true,
    "catholic_deuterocanon": false,
    "eastern_orthodox_additional": false,
    "ethiopian_orthodox_additional": false,
    "textual_variants": []
  },
  "defi_integration": {
    "tokenization_status": "tokenized",
    "primary_token_contract": "0x1234567890123456789012345678901234567890",
    "primary_token_id": "1",
    "royalty_bps": 500,
    "tithe_routing": {
      "enabled": true,
      "destination_church_id": "660e8400-e29b-41d4-a716-446655440001",
      "destination_wallet": "0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef",
      "tithe_percentage_bps": 1000,
      "trigger": "on_sale"
    },
    "impact_score": 0.99,
    "citation_frequency": 15000000,
    "stake_pool_address": "0xabcdefabcdefabcdefabcdefabcdefabcdefabcd"
  },
  "revision_history": [
    {
      "revision_id": "770e8400-e29b-41d4-a716-446655440002",
      "changed_at": "2026-03-01T00:00:00Z",
      "change_type": "schema_migration",
      "changed_by": "0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef",
      "notes": "Migrated from scripture_record v2.0.0 to v3.0.0. Added canonical_fingerprint."
    }
  ],
  "topics": ["salvation", "eternal_life", "gods_love", "faith", "incarnation"],
  "tags": ["john3v16", "most_memorized", "gospel_summary"],
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2026-03-01T00:00:00Z"
}
```

---

## Ajv 2020-12 Validation Snippet

```js
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';
import { readFileSync } from 'fs';

const ajv = new Ajv({ strict: true, allErrors: true });
addFormats(ajv);

const schema = JSON.parse(
  readFileSync('./schemas/scripture_record.schema.json', 'utf8')
);
const validate = ajv.compile(schema);

function validateScriptureRecord(record) {
  const valid = validate(record);
  if (!valid) {
    console.error('Validation errors:', validate.errors);
    return false;
  }
  console.log('✅ Valid scripture_record v3.0.0');
  return true;
}
```
