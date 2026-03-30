# ScriptureRecord v2.0.0 — Migration & Usage Guide

## 1. What's New in v2.0.0

### Core enhancements
- **`content_hash`** (required) — Keccak-256 cryptographic hash of the NFC-normalised, trimmed passage text for tamper-evident verification.
- **`tags`** — Additional free-form tagging array (alongside existing `topics`).
- **`schema_version`** updated to `2.0.0`.
- `"$id"` updated to `https://schemas.biblefi.io/v2/scripture_record.schema.json`.
- `additionalProperties: false` enforced on **every** nested object.

### New top-level sections

- **`translation`** — Now includes `dialect`, `manuscript_tradition` (enum), and richer BCP-47 `language` validation.
- **`provenance`** — Source organisation, license (enum), copyright year, attribution flag, digitisation source.
- **`ai_metadata`** — Embedding model/version/dimensions, semantic clusters, sentiment score, reading level (Flesch-Kincaid), theological themes (with confidence scores), and named entities.
- **`cross_references`** — Array of related passages with typed relationships, strength, direction, and notes.
- **`translation_graph`** — Parallel translation texts with semantic similarity scores and notable difference notes.
- **`attestations`** (array) — Replaces the single v1 `attestation` object. Supports multiple chains with `nft_mint`, `eas_attestation`, `hypercert`, `poap`, `sbt`, `merkle_leaf`, and `custom` types. Each entry can include a full Merkle proof.
- **`hermeneutics`** — Literary genre, speech act, covenant context, canonical position, christological significance, typology, interpretive traditions, and original-language keywords (Hebrew/Greek/Aramaic, with Strong's IDs).
- **`revision_history`** — Immutable append-only log of every change with before/after content hashes and change type classification.

---

## 2. Breaking Changes

| Change | v1.0.0 | v2.0.0 |
|---|---|---|
| `content_hash` | Not present | **Required** |
| `translation` required fields | `[abbreviation, full_name, language]` | Same, but object now also accepts `dialect` and `manuscript_tradition` |
| `attestation` (singular) | Optional object with `chain_id`, `contract_address`, `token_id` required | **Removed** — replaced by `attestations` array |
| `$id` URI | `https://schemas.biblefi.io/scripture_record.schema.json` | `https://schemas.biblefi.io/v2/scripture_record.schema.json` |
| `additionalProperties` | Not enforced in nested objects | Enforced throughout |

### Migration checklist

1. **Compute and add `content_hash`** — see [Section 4](#4-how-to-compute-content_hash).
2. **Rename `attestation` → `attestations`** and wrap the single object in an array. Add the new required field `attestation_type` (e.g. `"nft_mint"`).
3. **Update `schema_version`** from `"1.0.0"` to `"2.0.0"`.
4. **Remove any unknown properties** on nested objects — `additionalProperties: false` is now strict throughout.
5. Update any `$ref` / `$id` lookups from the v1 URI to `https://schemas.biblefi.io/v2/scripture_record.schema.json`.

---

## 3. Step-by-Step Migration Guide (v1 → v2)

### Step 1 — Update `schema_version`

```json
// Before
"schema_version": "1.0.0"

// After
"schema_version": "2.0.0"
```

### Step 2 — Compute and add `content_hash`

See [Section 4](#4-how-to-compute-content_hash) for language-specific instructions.

```json
"content_hash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab"
```

### Step 3 — Migrate `attestation` → `attestations`

```json
// Before (v1)
"attestation": {
  "chain_id": 8453,
  "contract_address": "0xAbCd1234...",
  "token_id": "42",
  "tx_hash": "0xdeadbeef..."
}

// After (v2)
"attestations": [
  {
    "chain_id": 8453,
    "contract_address": "0xAbCd1234...",
    "token_id": "42",
    "tx_hash": "0xdeadbeef...",
    "attestation_type": "nft_mint"
  }
]
```

### Step 4 — Remove unknown properties from nested objects

Because `additionalProperties: false` is now enforced on all nested objects, any extra fields you were storing under `translation` or `attestation` will cause validation errors. Move them to the new purpose-built sections or remove them.

### Step 5 — (Optional) Populate new fields

Gradually enrich records with `provenance`, `ai_metadata`, `hermeneutics`, `cross_references`, and `translation_graph` as your pipeline supports them. All new sections are optional.

---

## 4. How to Compute `content_hash`

`content_hash` is the **Keccak-256** hash of the passage text after:
1. Unicode NFC normalisation
2. Leading/trailing whitespace trimming
3. UTF-8 encoding

### JavaScript (viem)

```js
import { keccak256, toBytes } from 'viem';

const hash = keccak256(toBytes(text.normalize('NFC').trim()));
// hash → "0x..." (66-character hex string)
```

### Python (eth-hash)

```python
from eth_hash.auto import keccak
import unicodedata

h = '0x' + keccak(unicodedata.normalize('NFC', text.strip()).encode()).hex()
# h → "0x..." (66-character hex string)
```

> **Important:** The hash must cover **only the `text` field value** — do not include any other fields, whitespace padding, or newlines introduced by JSON serialisation.

---

## 5. Complete Valid v2 Example — John 3:16 (KJV)

```json
{
  "record_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "schema_version": "2.0.0",
  "book": "John",
  "chapter": 3,
  "verses": 16,
  "translation": {
    "abbreviation": "KJV",
    "full_name": "King James Version",
    "language": "en",
    "year_published": 1611,
    "manuscript_tradition": "textus_receptus"
  },
  "text": "For God so loved the world, that he gave his only begotten Son, that whosoever believeth in him should not perish, but have everlasting life.",
  "content_hash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z",
  "topics": ["salvation", "God's love", "eternal life"],
  "tags": ["john316", "gospel", "memory-verse"],
  "provenance": {
    "source_organization": "Bible Foundation",
    "license": "public_domain",
    "attribution_required": false
  },
  "ai_metadata": {
    "embedding_model": "text-embedding-ada-002",
    "embedding_version": "2",
    "embedding_dimensions": 1536,
    "semantic_clusters": ["divine-love", "soteriology", "incarnation"],
    "sentiment_score": 0.92,
    "reading_level": {
      "flesch_kincaid_grade": 8.2,
      "flesch_reading_ease": 68.5,
      "grade_label": "middle_school"
    },
    "theological_themes": [
      { "theme": "soteriology", "confidence": 0.99 },
      { "theme": "christology", "confidence": 0.95 },
      { "theme": "covenant", "confidence": 0.72 }
    ],
    "named_entities": [
      { "name": "God", "type": "deity", "biblical_id": "H430" },
      { "name": "Son", "type": "person", "biblical_id": "G5207" }
    ]
  },
  "cross_references": [
    {
      "book": "Romans",
      "chapter": 5,
      "verses": 8,
      "relationship": "parallel",
      "strength": "explicit",
      "direction": "bidirectional",
      "notes": "Both passages emphasise God's love demonstrated through Christ's sacrifice."
    },
    {
      "book": "Isaiah",
      "chapter": 53,
      "verses": { "from": 4, "to": 6 },
      "relationship": "fulfillment",
      "strength": "probable",
      "direction": "forward"
    }
  ],
  "translation_graph": [
    {
      "abbreviation": "NIV",
      "language": "en",
      "text": "For God so loved the world that he gave his one and only Son, that whoever believes in him shall not perish but have eternal life.",
      "content_hash": "0x1234abcd5678ef901234abcd5678ef901234abcd5678ef901234abcd5678ef90ab",
      "year_published": 1978,
      "semantic_similarity_to_primary": 0.97,
      "notable_differences": "'one and only' vs 'only begotten'; 'believes' vs 'believeth' (modernised verb forms)"
    }
  ],
  "attestations": [
    {
      "chain_id": 8453,
      "contract_address": "0x7bEda57074AA917FF0993fb329E16C2c188baF08",
      "token_id": "1001",
      "tx_hash": "0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef00",
      "attestation_type": "nft_mint",
      "block_number": 12345678,
      "attested_at": "2024-01-15T10:35:00Z",
      "attested_by": "0xBibleFiProtocol"
    }
  ],
  "hermeneutics": {
    "literary_genre": "gospel",
    "speech_act": "declaration",
    "covenant_context": "new_covenant",
    "canonical_position": "nt_gospel",
    "christological_significance": "The most concise statement of the Gospel: Christ as the gift of God's love, the object of saving faith, and the source of eternal life.",
    "typology": {
      "is_antitype": true,
      "is_type": false,
      "type_description": "Fulfils the type of the bronze serpent (Numbers 21:8-9) referenced in John 3:14-15."
    },
    "interpretive_traditions": [
      {
        "tradition": "reformed",
        "interpretation": "Emphasises unconditional election — 'the world' refers to the elect from all nations rather than every individual without exception.",
        "source": "Calvin's Commentary on John 3:16"
      },
      {
        "tradition": "arminian",
        "interpretation": "'The world' and 'whosoever' indicate God's universal saving intent, with salvation conditioned on individual faith.",
        "source": "Wesley's Notes on the New Testament"
      }
    ],
    "original_language_notes": {
      "greek_keywords": [
        { "word": "ἠγάπησεν", "strongs_id": "G25", "gloss": "loved (aorist — a decisive, historical act of love)" },
        { "word": "μονογενῆ", "strongs_id": "G3439", "gloss": "only begotten / one and only" },
        { "word": "πιστεύων", "strongs_id": "G4100", "gloss": "believing (present participle — ongoing trust)" }
      ]
    }
  },
  "revision_history": [
    {
      "version": "2.0.0",
      "changed_at": "2024-01-15T10:30:00Z",
      "changed_by": "biblefi-ingest-pipeline",
      "content_hash_before": "0x0000000000000000000000000000000000000000000000000000000000000000",
      "content_hash_after": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
      "change_type": "schema_migration",
      "diff_summary": "Initial creation under v2.0.0 schema.",
      "reviewed_by": "curator@biblefi.io"
    }
  ]
}
```

---

## 6. Ajv Validation Example (Node.js)

```js
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';
import { readFileSync } from 'fs';

const ajv = new Ajv({ strict: true });
addFormats(ajv);

const schema = JSON.parse(
  readFileSync('schemas/scripture_record.schema.json', 'utf8')
);

const validate = ajv.compile(schema);

const record = {
  record_id: 'f47ac10b-58cc-4372-a567-0e02b2c3d479',
  schema_version: '2.0.0',
  book: 'John',
  chapter: 3,
  verses: 16,
  translation: {
    abbreviation: 'KJV',
    full_name: 'King James Version',
    language: 'en'
  },
  text: 'For God so loved the world...',
  content_hash: '0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab'
};

const valid = validate(record);

if (!valid) {
  console.error('Validation errors:', validate.errors);
} else {
  console.log('Record is valid ✓');
}
```

Install dependencies:

```bash
npm install ajv ajv-formats
```

> **Note:** Use `ajv/dist/2020.js` to load the JSON Schema Draft 2020-12 validator. The default `ajv` export targets Draft-07.
