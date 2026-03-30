# BibleFi Schemas

Canonical [JSON Schema](https://json-schema.org/) definitions for the BibleFi protocol. Every message exchanged between agents, every record stored on-chain, and every payload surfaced by BibleFi APIs must conform to one of these schemas.

All schemas use **JSON Schema Draft 2020-12** and enforce `additionalProperties: false` throughout.

---

## Repository Layout

```
schemas/
├── agent_envelope.schema.json       # Universal payload wrapper for all BibleFi messages
├── church_record.schema.json        # Church / faith community record
├── defi_strategy.schema.json        # DeFi tithing and giving strategy
├── scripture_record.schema.json     # Scripture record v3.0.0 (current)
├── scripture_record.v1.schema.json  # Scripture record v1.0.0 (deprecated, retained until 2028-03)
├── security_finding.schema.json     # Smart contract security finding
├── user_profile.schema.json         # BibleFi user profile
└── wallet_record.schema.json        # EVM wallet record

docs/
└── scripture_record_v3_guide.md     # Migration guide: v2 → v3, full examples

.github/workflows/
└── validate-schemas.yml             # CI: validates all schemas are valid JSON

VERSIONING.md                        # SemVer policy, deprecation schedule, version table
```

---

## Schema Descriptions

### `agent_envelope.schema.json` — v1.0.0
Universal envelope wrapping all BibleFi agent payloads. Provides routing, correlation, signature verification, and cross-chain metadata.

**Key fields:** `envelope_id`, `payload_type` (enum), `payload`, `agent_id`, `issued_at`, `signature` (ed25519/secp256k1/rsa-pss), `routing` (cross-chain bridge metadata)

---

### `church_record.schema.json` — v1.0.0
Represents a church or faith community participating in the BibleFi protocol, including treasury wallet, governance model, and on-chain identity.

**Key fields:** `church_id`, `name`, `denomination`, `wallet_address`, `treasury_wallet`, `governance_model` (elder_led/congregational/episcopal/presbyterian/apostolic/dao), `on_chain_id`, `verified`

---

### `defi_strategy.schema.json` — v1.0.0
Describes a BibleFi DeFi tithing or giving strategy — how digital assets are deployed across protocols in accordance with biblical stewardship principles.

**Key fields:** `strategy_id`, `strategy_type` (yield_farming/staking/tithe_routing/endowment/etc.), `protocols` (array with allocation_bps), `ethical_filters`, `auto_tithe_enabled`, `tithe_percentage_bps`

---

### `scripture_record.schema.json` — v3.0.0 ⭐
The centerpiece schema. Cryptographically verifiable, AI-enriched, hermeneutically annotated, and on-chain attested scripture records.

**Key fields:**
- `record_id`, `book`, `chapter`, `verses`, `text`
- `content_hash` — Keccak-256 of NFC-normalized text
- `canonical_fingerprint` — SHA3-256 + BLAKE3 + verse_bitmask composite fingerprint
- `translation` — with manuscript_tradition and original_language
- `ai_metadata` — embeddings, theological themes, named entities, sentiment, reading level
- `hermeneutics` — genre, speech_act, covenant_context, typology, original_language_notes
- `linguistic_analysis` — syntactic complexity, parallelism type, rhetorical devices
- `theological_graph` — graph-theoretic concept relationship map
- `canon_authority` — multi-tradition canonical status and textual variants
- `cross_references`, `translation_graph`, `attestations`, `defi_integration`, `revision_history`

---

### `scripture_record.v1.schema.json` — v1.0.0 ⚠️ DEPRECATED
Preserved v1 archive. **Deprecated: use v3.0.0.** Retained until 2028-03.

---

### `security_finding.schema.json` — v1.0.0
Smart contract security finding records from audits. Tracks severity, status, affected contracts, CWE/SWC IDs, and on-chain attestation of resolution.

**Key fields:** `finding_id`, `severity` (critical/high/medium/low/informational/gas_optimization), `status`, `contract_address`, `cvss_score`, `on_chain_attestation`

---

### `user_profile.schema.json` — v1.0.0
BibleFi user profile with role-based access, wallet associations, church affiliation, and notification preferences.

**Key fields:** `user_id`, `display_name`, `role` (individual/church_admin/auditor/developer/observer), `wallet_addresses`, `church_id`, `notification_preferences`, `verified`

---

### `wallet_record.schema.json` — v1.0.0
EVM-compatible wallet record with multi-chain support, balance snapshots, and church treasury designation.

**Key fields:** `wallet_id`, `address`, `chain_ids`, `wallet_type` (eoa/multisig/smart_wallet/hardware), `is_treasury`, `balance_snapshot`, `owner_id`

---

## Validation Examples

### JavaScript (Ajv)

```js
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';
import { readFileSync } from 'fs';

const ajv = new Ajv({ strict: true });
addFormats(ajv);

const schema = JSON.parse(readFileSync('./schemas/scripture_record.schema.json', 'utf8'));
const validate = ajv.compile(schema);

const record = {
  record_id: '550e8400-e29b-41d4-a716-446655440000',
  schema_version: '3.0.0',
  book: 'John',
  chapter: 3,
  verses: 16,
  translation: {
    abbreviation: 'KJV',
    full_name: 'King James Version',
    language: 'en'
  },
  text: 'For God so loved the world, that he gave his only begotten Son, that whosoever believeth in him should not perish, but have everlasting life.',
  content_hash: '0x' + '<keccak256-of-text>',
  canonical_fingerprint: {
    sha3_256: '<64-hex-chars>',
    blake3: '<64-hex-chars>',
    verse_bitmask: '0x8000',
    canonical_key: 'John:3:16:KJV',
    algorithm_version: '1.0'
  }
};

const valid = validate(record);
if (!valid) console.error(validate.errors);
else console.log('Valid!');
```

### Python (jsonschema)

```python
import json
from jsonschema import validate, Draft202012Validator

with open('schemas/scripture_record.schema.json') as f:
    schema = json.load(f)

record = {
    "record_id": "550e8400-e29b-41d4-a716-446655440000",
    "schema_version": "3.0.0",
    "book": "John",
    "chapter": 3,
    "verses": 16,
    "translation": {
        "abbreviation": "KJV",
        "full_name": "King James Version",
        "language": "en"
    },
    "text": "For God so loved the world...",
    "content_hash": "0x" + "a" * 64,
    "canonical_fingerprint": {
        "sha3_256": "a" * 64,
        "blake3": "b" * 64,
        "verse_bitmask": "0x8000",
        "canonical_key": "John:3:16:KJV",
        "algorithm_version": "1.0"
    }
}

validate(instance=record, schema=schema, cls=Draft202012Validator)
print("Valid!")
```

---

## `content_hash` Computation

The `content_hash` is the **Keccak-256** hash of the NFC-normalized, trimmed UTF-8 scripture text.

### JavaScript

```js
import { keccak256, toUtf8Bytes } from 'ethers';

function computeContentHash(text) {
  // 1. NFC normalize
  const normalized = text.normalize('NFC');
  // 2. Trim whitespace
  const trimmed = normalized.trim();
  // 3. Keccak-256 hash
  return keccak256(toUtf8Bytes(trimmed));
}

const hash = computeContentHash(
  'For God so loved the world, that he gave his only begotten Son...'
);
// Returns: "0x<64-hex-chars>"
```

### Python

```python
from eth_hash.auto import keccak
import unicodedata

def compute_content_hash(text: str) -> str:
    # 1. NFC normalize
    normalized = unicodedata.normalize('NFC', text)
    # 2. Trim whitespace
    trimmed = normalized.strip()
    # 3. Keccak-256 hash
    return '0x' + keccak(trimmed.encode('utf-8')).hex()

hash_val = compute_content_hash(
    'For God so loved the world, that he gave his only begotten Son...'
)
```

---

## `canonical_fingerprint` Computation

```js
import { sha3_256 } from 'js-sha3';
import { blake3 } from '@noble/hashes/blake3';

function buildVerseKey(verses) {
  if (typeof verses === 'number') return String(verses);
  if (Array.isArray(verses)) return verses.sort((a,b)=>a-b).join(',');
  return `${verses.from}-${verses.to}`;
}

function computeCanonicalFingerprint(book, chapter, verses, translationAbbr, contentHash) {
  const verseKey = buildVerseKey(verses);
  const canonicalStr = `${book}:${chapter}:${verseKey}:${translationAbbr}:${contentHash}`;

  // SHA3-256
  const sha3 = sha3_256(canonicalStr);

  // BLAKE3
  const b3 = Buffer.from(blake3(canonicalStr)).toString('hex');

  // Verse bitmask: set bits for all verses present
  let mask = 0n;
  const verseNums = typeof verses === 'number' ? [verses]
    : Array.isArray(verses) ? verses
    : Array.from({length: verses.to - verses.from + 1}, (_,i) => verses.from + i);
  for (const v of verseNums) mask |= (1n << BigInt(v - 1));

  return {
    sha3_256: sha3,
    blake3: b3,
    verse_bitmask: '0x' + mask.toString(16),
    canonical_key: `${book}:${chapter}:${verseKey}:${translationAbbr}`,
    algorithm_version: '1.0'
  };
}
```

---

## Current Schema Versions

| Schema | Version | Status | `$id` |
|--------|---------|--------|-------|
| `agent_envelope` | 1.0.0 | ✅ Stable | `https://schemas.biblefi.io/v1/agent_envelope.schema.json` |
| `church_record` | 1.0.0 | ✅ Stable | `https://schemas.biblefi.io/v1/church_record.schema.json` |
| `defi_strategy` | 1.0.0 | ✅ Stable | `https://schemas.biblefi.io/v1/defi_strategy.schema.json` |
| `scripture_record` | 3.0.0 | ✅ Stable | `https://schemas.biblefi.io/v3/scripture_record.schema.json` |
| `scripture_record.v1` | 1.0.0 | ⚠️ Deprecated | `https://schemas.biblefi.io/v1/scripture_record.schema.json` |
| `security_finding` | 1.0.0 | ✅ Stable | `https://schemas.biblefi.io/v1/security_finding.schema.json` |
| `user_profile` | 1.0.0 | ✅ Stable | `https://schemas.biblefi.io/v1/user_profile.schema.json` |
| `wallet_record` | 1.0.0 | ✅ Stable | `https://schemas.biblefi.io/v1/wallet_record.schema.json` |
