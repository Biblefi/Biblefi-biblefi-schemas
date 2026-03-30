# ScriptureRecord v2.0.0 — Migration & Usage Guide

## What's New in v2.0.0

The v2.0.0 schema is a major upgrade to the `ScriptureRecord` format, adding the following new sections:

- **`content_hash`** (required) — Keccak-256 hash of the canonical text for on-chain integrity verification.
- **`attestations`** array — Replaces the single v1 `attestation` object; supports multiple on-chain proofs (NFT mints, EAS attestations, Hypercerts, POAPs, SBTs, Merkle leaves).
- **`translation_graph`** array — Multi-translation comparison with semantic similarity scores; enables cross-translation analysis.
- **`translation.dialect`** — Dialect or regional variant of the primary translation language.
- **`translation.manuscript_tradition`** — Underlying manuscript tradition (Masoretic, Septuagint, Textus Receptus, etc.).
- **`provenance`** object — Source organization, URL, license, copyright year, and digitization source.
- **`ai_metadata`** object — Embedding model info, semantic clusters, sentiment score, reading level, theological themes, and named entity recognition.
- **`cross_references`** array — Structured cross-references with relationship type, strength, and direction.
- **`hermeneutics`** object — Literary genre, speech act, covenant context, canonical position, Christological significance, typology, interpretive traditions, and original language notes.
- **`revision_history`** array — Immutable audit trail of all changes, with before/after content hashes.
- **`tags`** array — General-purpose categorization tags (in addition to the existing `topics` array).

---

## Breaking Changes

Upgrading from v1.0.0 to v2.0.0 requires the following changes:

### 1. `content_hash` is now a **required** field

Every v2 document **must** include a `content_hash` property. This is a Keccak-256 hex string (`0x` followed by 64 hex digits) computed from the canonical text. See [Computing `content_hash`](#computing-content_hash) below.

### 2. `attestation` (singular) replaced by `attestations` (array)

The v1 `attestation` field (a single object requiring `chain_id`, `contract_address`, and `token_id`) has been removed. In v2, use the `attestations` **array** of objects. Each object requires `chain_id`, `contract_address`, and `attestation_type`. If you had a single v1 attestation, migrate it as the first element in the array.

**v1:**
```json
{
  "attestation": {
    "chain_id": 8453,
    "contract_address": "0xAbc...123",
    "token_id": "42",
    "tx_hash": "0xdeadbeef..."
  }
}
```

**v2:**
```json
{
  "attestations": [
    {
      "chain_id": 8453,
      "contract_address": "0xAbc...123",
      "token_id": "42",
      "tx_hash": "0xdeadbeef...",
      "attestation_type": "nft_mint"
    }
  ]
}
```

### 3. `$id` URI changed to include `/v2/`

The schema `$id` has changed from:
```
https://schemas.biblefi.io/scripture_record.schema.json
```
to:
```
https://schemas.biblefi.io/v2/scripture_record.schema.json
```

Update any `$ref` or validator configuration that references the v1 `$id` URI.

---

## Migration Guide

Follow these steps to migrate an existing v1 ScriptureRecord to v2:

### Step 1 — Update `schema_version`

```json
"schema_version": "2.0.0"
```

### Step 2 — Compute and add `content_hash`

Compute the Keccak-256 hash of the `text` field (NFC-normalized, whitespace-trimmed) and add it as `content_hash`. See [Computing `content_hash`](#computing-content_hash) for code examples.

### Step 3 — Migrate `attestation` → `attestations`

If your v1 document has an `attestation` field, move it into the `attestations` array and add the required `attestation_type` field:

```js
const v2doc = {
  ...v1doc,
  schema_version: "2.0.0",
  content_hash: computeHash(v1doc.text),
  attestations: v1doc.attestation
    ? [{ ...v1doc.attestation, attestation_type: "nft_mint" }]
    : undefined,
};
delete v2doc.attestation;
```

### Step 4 — (Optional) Add new fields

Populate any combination of the new optional fields to enrich the record:
- `provenance` — Add licensing and source information.
- `translation.dialect` and `translation.manuscript_tradition` — Enrich translation metadata.
- `translation_graph` — Add parallel translations for semantic comparison.
- `cross_references` — Link to related passages.
- `ai_metadata` — Add embeddings and theological analysis.
- `hermeneutics` — Add interpretive and literary metadata.
- `revision_history` — Record this migration as the first revision entry.
- `tags` — Add general-purpose filtering tags.

### Step 5 — Validate against v2 schema

Run validation against `schemas/scripture_record.schema.json` (v2) before publishing. See [Validation Example](#validation-example) below.

---

## Computing `content_hash`

The `content_hash` is a Keccak-256 hash of the `text` field after:
1. Unicode **NFC normalization** (canonical decomposition followed by canonical composition)
2. Leading and trailing **whitespace trimming**
3. UTF-8 encoding

The result is a lowercase hex string prefixed with `0x`.

### JavaScript (using [viem](https://viem.sh/))

```js
import { keccak256, toBytes } from 'viem';

/**
 * Compute the content_hash for a ScriptureRecord text field.
 * @param {string} text - The verbatim scripture text.
 * @returns {string} Keccak-256 hash as a 0x-prefixed hex string.
 */
function computeContentHash(text) {
  const canonical = text.normalize('NFC').trim();
  return keccak256(toBytes(canonical));
}

// Example:
const hash = computeContentHash(
  'For God so loved the world, that he gave his only begotten Son, ' +
  'that whosoever believeth in him should not perish, but have everlasting life.'
);
console.log(hash); // 0x<64 hex chars>
```

### Python (using [eth-hash](https://pypi.org/project/eth-hash/))

```python
import unicodedata
from eth_hash.auto import keccak

def compute_content_hash(text: str) -> str:
    """
    Compute the content_hash for a ScriptureRecord text field.

    Args:
        text: The verbatim scripture text.

    Returns:
        Keccak-256 hash as a 0x-prefixed hex string.
    """
    canonical = unicodedata.normalize('NFC', text.strip()).encode('utf-8')
    return '0x' + keccak(canonical).hex()

# Example:
hash_value = compute_content_hash(
    'For God so loved the world, that he gave his only begotten Son, '
    'that whosoever believeth in him should not perish, but have everlasting life.'
)
print(hash_value)  # 0x<64 hex chars>
```

---

## Example Valid v2 Payload

The following is a complete, valid ScriptureRecord v2 document for John 3:16 (KJV) with all major sections populated:

```json
{
  "record_id": "550e8400-e29b-41d4-a716-446655440000",
  "schema_version": "2.0.0",
  "book": "John",
  "chapter": 3,
  "verses": 16,
  "translation": {
    "abbreviation": "KJV",
    "full_name": "King James Version",
    "language": "en",
    "year_published": 1611,
    "dialect": "Early Modern English",
    "manuscript_tradition": "textus_receptus"
  },
  "text": "For God so loved the world, that he gave his only begotten Son, that whosoever believeth in him should not perish, but have everlasting life.",
  "content_hash": "0x6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b",
  "created_at": "2024-01-15T12:00:00Z",
  "updated_at": "2024-01-15T12:00:00Z",
  "provenance": {
    "source_organization": "Public Domain",
    "license": "public_domain",
    "copyright_year": 1611,
    "attribution_required": false,
    "digitization_source": "Project Gutenberg"
  },
  "ai_metadata": {
    "embedding_model": "text-embedding-3-small",
    "embedding_version": "3",
    "embedding_dimensions": 1536,
    "semantic_clusters": ["salvation", "divine-love", "eternal-life"],
    "sentiment_score": 0.92,
    "reading_level": {
      "flesch_kincaid_grade": 8.2,
      "flesch_reading_ease": 65.4,
      "grade_label": "middle_school"
    },
    "theological_themes": [
      { "theme": "soteriology", "confidence": 0.98 },
      { "theme": "christology", "confidence": 0.91 },
      { "theme": "covenant", "confidence": 0.74 }
    ],
    "named_entities": [
      { "name": "God", "type": "deity", "biblical_id": "Q1" },
      { "name": "Son", "type": "deity", "biblical_id": "Q302" },
      { "name": "world", "type": "place" }
    ]
  },
  "cross_references": [
    {
      "book": "Romans",
      "chapter": 5,
      "verses": 8,
      "relationship": "parallel",
      "strength": "explicit",
      "direction": "forward",
      "notes": "Paul echoes John's declaration of divine love expressed through Christ's sacrifice."
    },
    {
      "book": "Isaiah",
      "chapter": 53,
      "verses": { "from": 4, "to": 6 },
      "relationship": "fulfillment",
      "strength": "probable",
      "direction": "backward"
    }
  ],
  "translation_graph": [
    {
      "abbreviation": "NIV",
      "language": "en",
      "text": "For God so loved the world that he gave his one and only Son, that whoever believes in him shall not perish but have eternal life.",
      "content_hash": "0xa1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
      "year_published": 1978,
      "semantic_similarity_to_primary": 0.97,
      "notable_differences": "NIV renders 'monogenes' as 'one and only' vs KJV 'only begotten'; 'eternal life' vs 'everlasting life'."
    },
    {
      "abbreviation": "NVI",
      "language": "es",
      "text": "Porque tanto amó Dios al mundo, que dio a su Hijo unigénito, para que todo el que cree en él no se pierda, sino que tenga vida eterna.",
      "content_hash": "0xb2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3",
      "year_published": 1999,
      "semantic_similarity_to_primary": 0.93,
      "notable_differences": "Spanish 'unigénito' directly corresponds to Greek 'monogenes' (only-begotten)."
    }
  ],
  "attestations": [
    {
      "chain_id": 8453,
      "contract_address": "0x7bEda57074AA917FF0993fb329E16C2c188baF08",
      "token_id": "1",
      "tx_hash": "0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef",
      "attestation_type": "nft_mint",
      "block_number": 12345678,
      "attested_at": "2024-01-15T12:05:00Z",
      "attested_by": "0x742d35Cc6634C0532925a3b8D4C9b5F6a5E6F7a8"
    }
  ],
  "hermeneutics": {
    "literary_genre": "gospel",
    "speech_act": "declaration",
    "covenant_context": "new_covenant",
    "canonical_position": "nt_gospel",
    "christological_significance": "The definitive NT statement on the incarnation as the supreme act of divine love; grounds atonement theology and the offer of eternal life to all who believe.",
    "typology": {
      "is_antitype": true,
      "is_type": false,
      "type_description": "Fulfills the bronze serpent type of Numbers 21:8-9 (referenced in John 3:14-15)."
    },
    "interpretive_traditions": [
      {
        "tradition": "reformed",
        "interpretation": "The 'world' here refers to the elect from every nation rather than every individual without exception, consistent with particular redemption.",
        "source": "Calvin, Commentary on John 3:16"
      },
      {
        "tradition": "arminian",
        "interpretation": "God's love extends to every individual person, and the offer of salvation is genuinely universal; conditioned on faith.",
        "source": "Wesley, Notes on the New Testament"
      }
    ],
    "original_language_notes": {
      "greek_keywords": [
        { "word": "ἠγάπησεν", "strongs_id": "G25", "gloss": "loved (agapaō — unconditional, self-giving love)" },
        { "word": "μονογενῆ", "strongs_id": "G3439", "gloss": "only-begotten, unique" },
        { "word": "αἰώνιον", "strongs_id": "G166", "gloss": "eternal, age-lasting" }
      ]
    }
  },
  "revision_history": [
    {
      "version": "2.0.0",
      "changed_at": "2024-01-15T12:00:00Z",
      "changed_by": "did:ethr:0x742d35Cc6634C0532925a3b8D4C9b5F6a5E6F7a8",
      "change_type": "schema_migration",
      "content_hash_before": "0x0000000000000000000000000000000000000000000000000000000000000000",
      "content_hash_after": "0x6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b",
      "diff_summary": "Initial v2.0.0 record creation; migrated from v1.0.0 schema."
    }
  ],
  "topics": ["salvation", "divine-love", "faith", "eternal-life"],
  "tags": ["memory-verse", "john-3", "most-cited"]
}
```

---

## Validation Example

### Node.js with [Ajv](https://ajv.js.org/) (supports JSON Schema Draft 2020-12)

```js
import Ajv2020 from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';
import { readFileSync } from 'fs';

// Load the v2 schema
const schema = JSON.parse(
  readFileSync('./schemas/scripture_record.schema.json', 'utf8')
);

// Initialize Ajv with Draft 2020-12 support
const ajv = new Ajv2020({ allErrors: true });
addFormats(ajv); // adds 'uuid', 'date-time', 'uri' format validators

const validate = ajv.compile(schema);

// Validate a document
const document = {
  record_id: '550e8400-e29b-41d4-a716-446655440000',
  schema_version: '2.0.0',
  book: 'John',
  chapter: 3,
  verses: 16,
  translation: {
    abbreviation: 'KJV',
    full_name: 'King James Version',
    language: 'en',
  },
  text: 'For God so loved the world...',
  content_hash: '0x6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b',
};

const valid = validate(document);

if (valid) {
  console.log('✅ Document is valid');
} else {
  console.error('❌ Validation errors:', validate.errors);
}
```

Install dependencies:
```bash
npm install ajv ajv-formats
```
