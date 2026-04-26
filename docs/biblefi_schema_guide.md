# BibleFi Schema Guide

## 1. Overview

**BibleFi** is the world's first Christian-faith-based decentralized finance (DeFi) application, built on **Base chain** (`chain_id: 8453`) and deployed as both a native Base App and a **Farcaster mini-app**. It is rooted in biblical stewardship principles:

- **Tithing** — Malachi 3:10: *"Bring the whole tithe into the storehouse."*
- **Firstfruits** — Proverbs 3:9–10: *"Honor the Lord with your wealth, with the firstfruits of all your crops."*
- **Generosity** — 2 Corinthians 9:7: *"God loves a cheerful giver."*

All BibleFi schemas conform to **JSON Schema Draft 2020-12** and enforce `additionalProperties: false` throughout all nested objects to ensure strict data contracts between agents, on-chain records, and APIs.

---

## 2. Schema Catalog

| Schema File | $id | Version | Status | Description |
|---|---|---|---|---|
| `scripture_record.schema.json` | `…/v2/scripture_record.schema.json` | 2.0.0 | Stable | Bible passages with content hash, AI metadata, hermeneutics, attestations |
| `scripture_record.v1.schema.json` | `…/v1/scripture_record.schema.json` | 1.0.0 | Deprecated | Legacy passage record (retained until 2028-03) |
| `tithe_transaction.schema.json` | `…/v1/tithe_transaction.schema.json` | 1.0.0 | Stable | Biblical giving transactions on Base chain |
| `farcaster_frame_event.schema.json` | `…/v1/farcaster_frame_event.schema.json` | 1.0.0 | Stable | Farcaster Frame interaction events |
| `agent_envelope.schema.json` | `…/v2/agent_envelope.schema.json` | 2.0.0 | Stable | Universal agent message wrapper with routing and security context |
| `church_record.schema.json` | `…/church_record.schema.json` | 1.0.0 | Stable | Church profiles with on-chain treasury |
| `defi_strategy.schema.json` | `…/defi_strategy.schema.json` | 1.0.0 | Stable | DeFi yield strategies with tithe allocation |
| `security_finding.schema.json` | `…/security_finding.schema.json` | 1.0.0 | Stable | Security audit findings |
| `wallet_record.schema.json` | `…/wallet_record.schema.json` | 1.0.0 | Stable | User wallet records |
| `user_profile.schema.json` | `…/user_profile.schema.json` | 1.0.0 | Stable | User profiles |

---

## 3. Schema Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AgentEnvelope (v2)                              │
│  payload_type → any schema below                                        │
│  farcaster_context → links to FarcasterFrameEvent                      │
│  base_chain_context → links to TitheTransaction / ScriptureRecord      │
└────────────────────────────┬────────────────────────────────────────────┘
                             │ wraps
         ┌───────────────────┼───────────────────────┐
         ▼                   ▼                       ▼
┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐
│ ScriptureRecord │  │ TitheTransaction │  │  FarcasterFrameEvent     │
│    (v2.0.0)     │  │    (v1.0.0)      │  │       (v1.0.0)           │
│                 │  │                  │  │                          │
│ content_hash ───┼──┤ scripture_ref ───┘  │ biblefi_action ──────────┤
│ attestations    │  │ asset (USDC/ETH)    │   scripture_record_id    │
│ hermeneutics    │  │ attestation (EAS)   │   tithe_transaction_id   │
│ stewardship_tags│  │ giving_strategy_id──┼──▶ defi_strategy         │
└────────────────-┘  └──────────────────┘  └──────────────────────────┘
         │                   │
         ▼                   ▼
┌─────────────────┐  ┌──────────────────┐
│  ChurchRecord   │  │   DefiStrategy   │
│  (v1.0.0)       │  │    (v1.0.0)      │
│  treasury_addr  │  │  tithe_alloc %   │
└─────────────────┘  └──────────────────┘
```

---

## 4. Content Hash Computation

The `content_hash` field in `scripture_record` uses **Keccak-256** of the UTF-8 encoded NFC-normalized, trimmed text. This enables on-chain integrity verification via BibleFi attestation contracts on Base chain.

**Formula:** `content_hash = keccak256(normalize_nfc(text.trim()))`

### JavaScript (using viem)

```javascript
import { keccak256, toBytes } from 'viem';

function computeContentHash(text) {
  // 1. Unicode NFC normalization
  const normalized = text.normalize('NFC');
  // 2. Trim whitespace
  const trimmed = normalized.trim();
  // 3. Encode as UTF-8 bytes and hash
  return keccak256(toBytes(trimmed));
}

// Example: John 3:16 (KJV)
const text = "For God so loved the world, that he gave his only begotten Son, that whosoever believeth in him should not perish, but have everlasting life.";
const hash = computeContentHash(text);
console.log(hash); // 0x...
```

### Python (using eth_hash)

```python
from eth_hash.auto import keccak
import unicodedata

def compute_content_hash(text: str) -> str:
    # 1. Unicode NFC normalization
    normalized = unicodedata.normalize('NFC', text)
    # 2. Trim whitespace
    trimmed = normalized.strip()
    # 3. Encode as UTF-8 bytes and hash
    hash_bytes = keccak(trimmed.encode('utf-8'))
    return '0x' + hash_bytes.hex()

# Example
text = "For God so loved the world..."
content_hash = compute_content_hash(text)
print(content_hash)  # 0x...
```

---

## 5. Tithing Flow End-to-End

The BibleFi tithing flow connects scripture to on-chain giving through Farcaster Frames:

```
1. Scripture View (FarcasterFrameEvent)
   frame_action: "view"
   biblefi_action.action_type: "scripture_view"
   biblefi_action.scripture_record_id: <uuid>
         │
         ▼
2. Tithe Initiate (FarcasterFrameEvent)
   frame_action: "tithe"
   biblefi_action.action_type: "tithe_initiate"
   biblefi_action.amount_wei: "10000000" (10 USDC)
         │
         ▼
3. Wallet Connection & Signing
   connected_wallet: "0x..."
   chain_id: 8453 (Base mainnet)
         │
         ▼
4. TitheTransaction Created
   transaction_type: "tithe"
   amount_wei: "10000000"
   asset.symbol: "USDC"
   scripture_reference: { book: "Malachi", chapter: 3, verses: 10 }
   smart_contract_routed: true
         │
         ▼
5. Base Chain Settlement (chain_id: 8453)
   tx_hash: "0x..."
   block_number: 12345678
         │
         ▼
6. EAS Attestation
   attestation.attestation_type: "eas_attestation"
   attestation.chain_id: 8453
         │
         ▼
7. Tithe Complete (FarcasterFrameEvent)
   frame_action: "tithe"
   biblefi_action.action_type: "tithe_complete"
   biblefi_action.tithe_transaction_id: <uuid>
```

---

## 6. Agent Envelope Routing

All BibleFi agent messages are wrapped in `AgentEnvelope` v2.0.0. The `routing` object controls message delivery, while `security_context` carries trust metadata.

> **⚠️ Security note:** `security_context` fields are **self-asserted** by the sender. Do not use them for authorization decisions without independent cryptographic verification of the envelope signature. A `trust_level: "verified"` claim is meaningless unless the `signature` field has been verified against a trusted key registry. See `TRUST_BOUNDARIES.md` in `Biblefi/BibleFi`.

```json
{
  "envelope_id": "550e8400-e29b-41d4-a716-446655440000",
  "schema_version": "2.0.0",
  "created_at": "2026-03-30T03:00:00Z",
  "payload_type": "tithe_transaction",
  "payload": { "...": "see TitheTransaction example below" },
  "routing": {
    "destination_agent": "tithe-router-agent",
    "source_agent": "farcaster-frame-agent",
    "priority": "high",
    "ttl_seconds": 300,
    "retry_count": 0,
    "correlation_id": "660e8400-e29b-41d4-a716-446655440001"
  },
  "security_context": {
    "classification": "confidential",
    "requires_verification": true,
    "trust_level": "verified",
    "sandboxed": true
  },
  "farcaster_context": {
    "cast_hash": "0xabc123",
    "fid": 12345,
    "frame_action": "tithe",
    "channel_id": "/biblefi",
    "frame_uri": "https://biblefi.io/frames/tithe"
  },
  "base_chain_context": {
    "chain_id": 8453,
    "block_number": 12345678,
    "tx_hash": "0xdef456def456def456def456def456def456def456def456def456def456def456",
    "gas_used": 21000,
    "contract_address": "0x7bEda57074AA917FF0993fb329E16C2c188baF08"
  }
}
```

---

## 7. Complete Example Payloads

### 7.1 ScriptureRecord v2 — John 3:16 (KJV)

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
    "manuscript_tradition": "textus_receptus"
  },
  "text": "For God so loved the world, that he gave his only begotten Son, that whosoever believeth in him should not perish, but have everlasting life.",
  "content_hash": "0x1c3c3e74f870fef93de9fa0be5b34dc4e1d4c64e9c4f7fa53e9f5a71f1c2d3e4",
  "ai_metadata": {
    "embedding_model": "text-embedding-3-large",
    "embedding_version": "3",
    "embedding_dimensions": 3072,
    "semantic_clusters": ["salvation", "love", "eternal life", "faith"],
    "sentiment_score": 0.95,
    "reading_level": {
      "flesch_kincaid_grade": 8.2,
      "flesch_reading_ease": 72.4,
      "grade_label": "middle_school"
    },
    "theological_themes": [
      { "theme": "soteriology", "confidence": 0.98 },
      { "theme": "christology", "confidence": 0.92 },
      { "theme": "covenant", "confidence": 0.75 }
    ],
    "named_entities": [
      { "name": "God", "type": "deity" },
      { "name": "Son", "type": "person", "biblical_id": "jesus_christ" }
    ]
  },
  "hermeneutics": {
    "literary_genre": "gospel",
    "speech_act": "declaration",
    "covenant_context": "new_covenant",
    "canonical_position": "nt_gospel",
    "christological_significance": "Central soteriological statement identifying Jesus as God's Son given for world salvation",
    "interpretive_traditions": [
      {
        "tradition": "reformed",
        "interpretation": "Demonstrates God's sovereign love in electing a people through Christ",
        "source": "Westminster Confession of Faith"
      }
    ],
    "original_language_notes": {
      "greek_keywords": [
        { "word": "ἀγαπάω", "strongs_id": "G25", "gloss": "to love (unconditionally)" },
        { "word": "μονογενής", "strongs_id": "G3439", "gloss": "only begotten, unique" }
      ]
    }
  },
  "stewardship_tags": ["generosity", "sacrifice"],
  "attestations": [
    {
      "chain_id": 8453,
      "contract_address": "0x7bEda57074AA917FF0993fb329E16C2c188baF08",
      "attestation_type": "eas_attestation",
      "tx_hash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
      "block_number": 12345678,
      "attested_at": "2026-03-30T03:00:00Z",
      "attested_by": "biblefi-attestation-agent"
    }
  ],
  "created_at": "2026-03-30T03:00:00Z",
  "updated_at": "2026-03-30T03:00:00Z"
}
```

### 7.2 TitheTransaction — 10% Tithe in USDC (Malachi 3:10)

```json
{
  "transaction_id": "660e8400-e29b-41d4-a716-446655440001",
  "schema_version": "1.0.0",
  "sender_wallet": "0x1234567890123456789012345678901234567890",
  "recipient_wallet": "0xabcdef0123456789abcdef0123456789abcdef01",
  "recipient_church_id": "770e8400-e29b-41d4-a716-446655440002",
  "amount_wei": "10000000",
  "amount_usd": 10.00,
  "asset": {
    "symbol": "USDC",
    "contract_address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "chain_id": 8453,
    "decimals": 6,
    "coingecko_id": "usd-coin"
  },
  "chain_id": 8453,
  "tx_hash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
  "block_number": 12345679,
  "transaction_type": "tithe",
  "scripture_reference": {
    "book": "Malachi",
    "chapter": 3,
    "verses": 10,
    "notes": "Bring the whole tithe into the storehouse"
  },
  "percentage_of_income": 10.0,
  "is_recurring": true,
  "recurrence_interval": "monthly",
  "smart_contract_routed": true,
  "contract_address": "0x7bEda57074AA917FF0993fb329E16C2c188baF08",
  "attestation": {
    "chain_id": 8453,
    "contract_address": "0x7bEda57074AA917FF0993fb329E16C2c188baF08",
    "attestation_type": "eas_attestation",
    "tx_hash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
    "attested_at": "2026-03-30T03:01:00Z",
    "attested_by": "biblefi-tithe-attestor"
  },
  "tags": ["tithe", "malachi-3-10", "monthly", "usdc"],
  "created_at": "2026-03-30T03:00:00Z",
  "updated_at": "2026-03-30T03:01:00Z"
}
```

### 7.3 FarcasterFrameEvent — Tithe Initiate

```json
{
  "event_id": "880e8400-e29b-41d4-a716-446655440003",
  "schema_version": "1.0.0",
  "fid": 12345,
  "username": "faithful_giver",
  "verified_address": "0x1234567890123456789012345678901234567890",
  "frame_action": "tithe",
  "frame_uri": "https://biblefi.io/frames/tithe/malachi-3-10",
  "cast_hash": "0xcast123abc",
  "channel_id": "/biblefi",
  "timestamp": "2026-03-30T03:00:00Z",
  "button_index": 1,
  "connected_wallet": "0x1234567890123456789012345678901234567890",
  "chain_id": 8453,
  "biblefi_action": {
    "action_type": "tithe_initiate",
    "scripture_record_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount_wei": "10000000",
    "asset_symbol": "USDC",
    "estimated_gas_wei": "50000000000000"
  },
  "created_at": "2026-03-30T03:00:00Z"
}
```

---

## 8. Validation with Ajv

Install Ajv with JSON Schema Draft 2020-12 support:

```bash
npm install ajv ajv-formats
```

```javascript
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';
import { readFileSync } from 'fs';

const ajv = new Ajv({ strict: true });
addFormats(ajv);

// Load and compile schema
const schema = JSON.parse(readFileSync('./schemas/scripture_record.schema.json', 'utf8'));
const validate = ajv.compile(schema);

// Validate a record
const record = { /* ...your scripture record... */ };
const valid = validate(record);

if (!valid) {
  console.error('Validation errors:', validate.errors);
} else {
  console.log('Valid ScriptureRecord!');
}
```

For use in CI pipelines, install `ajv-cli`:

```bash
npm install -g ajv-cli
ajv validate -s schemas/scripture_record.schema.json -d examples/john_3_16.json --spec=draft2020
```
