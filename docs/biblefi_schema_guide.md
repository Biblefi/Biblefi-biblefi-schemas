# BibleFi Schema Guide

A comprehensive reference for the BibleFi canonical JSON Schema suite — powering the world's first Christian-faith-based DeFi dApp on Base chain and Farcaster.

---

## 1. BibleFi Schema Overview

BibleFi is a **native Base App** and **Farcaster mini-app** that fuses Christian stewardship principles with decentralized finance. Every piece of data — from Bible passages to tithe transactions — conforms to a versioned JSON Schema (Draft 2020-12) stored in this repository.

| Schema | Version | Purpose |
|---|---|---|
| `agent_envelope` | 2.0.0 | Universal message wrapper for all BibleFi agent communications |
| `scripture_record` | 2.0.0 | Bible passage records with AI metadata, hermeneutics, and on-chain attestations |
| `defi_strategy` | 2.0.0 | DeFi yield/staking strategies with tithe allocation and vault integration |
| `church_record` | 2.0.0 | Church/ministry profiles with Farcaster identity, governance, and treasury |
| `tithe_transaction` | 1.0.0 | On-chain tithe and offering transactions with scripture backing |
| `farcaster_frame_event` | 1.0.0 | Farcaster Frame interaction events for the BibleFi mini-app |
| `wallet_record` | 1.0.0 | EVM wallet identity for individuals and church treasuries |
| `user_profile` | 1.0.0 | User profiles with role, wallet links, and stewardship history |
| `security_finding` | 1.0.0 | Security audit findings with CVSS, CWE, and remediation tracking |
| `scripture_record` (v1) | 1.0.0 | **DEPRECATED** — retained for 24 months (until 2028-03) |

---

## 2. Schema Relationship Map

```
┌──────────────────────────────────────────────────────────────┐
│                      agent_envelope                          │
│  wraps all payload types via payload_type discriminator      │
└───────────────┬──────────────────────────────────────────────┘
                │
    ┌───────────┼─────────────────────────────────┐
    │           │                                 │
    ▼           ▼                                 ▼
scripture_   defi_strategy              farcaster_frame_event
record       │  ▲                           │
    ▲        │  │ biblefi_vault_integration  ├─ biblefi_action.scripture_record_id
    │        │  │ scripture_backing          ├─ biblefi_action.strategy_id
    │        │  │                            └─ biblefi_action.church_id
    │        ▼  │
    │    tithe_transaction ──────────────────────────┐
    │        │  scripture_reference ──► scripture_record
    │        │  giving_strategy_id ───► defi_strategy
    │        │  recipient_church_id ──► church_record
    │        │  sender_user_id ───────► user_profile
    │        └────────────────────────────────────────┘
    │
church_record ◄─── user_profile.church_id
    ▲               wallet_record.church_id
    │
wallet_record ◄─── user_profile.wallet_addresses
    ▲
user_profile ◄──── wallet_record.owner_id
```

---

## 3. Content Hash Algorithm

The `content_hash` field in `scripture_record` and `defi_strategy` is a **Keccak-256** hash of the canonical content, used for on-chain integrity verification via BibleFi attestation contracts on Base chain (chain ID 8453).

**Algorithm:** `keccak256(normalize_nfc(content.trim()))` where `content` is the UTF-8 canonical text.

### JavaScript (Node.js + viem)

```js
import { keccak256, toHex, toBytes } from 'viem';

function computeContentHash(text) {
  // 1. Unicode NFC normalization
  const normalized = text.normalize('NFC');
  // 2. Trim whitespace
  const trimmed = normalized.trim();
  // 3. Keccak-256 of UTF-8 bytes
  const bytes = new TextEncoder().encode(trimmed);
  return keccak256(bytes);
}

// Example: John 3:16
const hash = computeContentHash(
  'For God so loved the world that he gave his one and only Son, ' +
  'that whoever believes in him shall not perish but have eternal life.'
);
console.log(hash);
// 0x<64 hex chars>
```

### Python (pysha3 / eth_hash)

```python
from unicodedata import normalize
from eth_hash.auto import keccak

def compute_content_hash(text: str) -> str:
    # 1. Unicode NFC normalization
    normalized = normalize('NFC', text)
    # 2. Trim whitespace
    trimmed = normalized.strip()
    # 3. Keccak-256 of UTF-8 bytes
    digest = keccak(trimmed.encode('utf-8'))
    return '0x' + digest.hex()

# Example: Malachi 3:10
hash_value = compute_content_hash(
    '"Bring the whole tithe into the storehouse, that there may be food in my house. '
    'Test me in this," says the LORD Almighty, "and see if I will not throw open the '
    'floodgates of heaven and pour out so much blessing that there will not be room enough to store it."'
)
print(hash_value)
```

---

## 4. Base Chain Integration

BibleFi is deployed on **Base chain mainnet** (chain ID **8453**), Coinbase's L2 Ethereum rollup.

| Field | Value |
|---|---|
| Chain Name | Base Mainnet |
| Chain ID | `8453` |
| Native Token | ETH |
| BibleFi Token | `BFI` (ERC-20) |
| Block Explorer | https://basescan.org |

### Attestation Types for BibleFi

| Type | Description |
|---|---|
| `nft_mint` | NFT minted as a giving receipt or scripture record |
| `eas_attestation` | Ethereum Attestation Service on Base |
| `hypercert` | Hypercerts for impact tracking (missions, giving) |
| `poap` | Proof of Attendance — devotionals, church events |
| `sbt` | Soulbound Token — church membership, verified giver |
| `merkle_leaf` | Leaf in a Merkle tree for batch attestations |
| `farcaster_frame` | Attestation via Farcaster Frame interaction |
| `custom` | Custom BibleFi attestation contract |

---

## 5. Tithing Flow

The complete BibleFi tithing flow from scripture to on-chain transaction:

```
1. User opens BibleFi in Farcaster/Warpcast (/biblefi channel)
   └─► FarcasterFrameEvent { frame_action: "view", biblefi_action.action_type: "scripture_view" }

2. User views Malachi 3:10 (tithe scripture)
   └─► ScriptureRecord { book: "Malachi", chapter: 3, verses: 10,
                         stewardship_tags: ["tithe"], content_hash: "0x..." }

3. User clicks "Tithe Now" button in frame
   └─► FarcasterFrameEvent { frame_action: "tithe", biblefi_action.action_type: "tithe_initiate" }

4. BibleFi creates TitheTransaction
   └─► TitheTransaction {
         transaction_type: "tithe",
         percentage_of_income: 10,
         scripture_reference: { book: "Malachi", chapter: 3, verses: 10 },
         smart_contract_routed: true,
         sender_wallet: "0x<user>",
         recipient_wallet: "0x<church>"
       }

5. Transaction confirmed on Base chain (chain_id: 8453)
   └─► TitheTransaction.tx_hash set, TitheTransaction.attestation created

6. FarcasterFrameEvent { biblefi_action.action_type: "tithe_complete" } emitted

7. ChurchRecord.treasury_stats updated
   └─► treasury_stats.tithe_received_usd_ytd += amount_usd

8. UserProfile.stewardship_profile updated
   └─► total_tithe_usd_ytd += amount_usd, giving_streak_days++
```

---

## 6. Agent Integration

BibleFi agents communicate exclusively via `agent_envelope`. The envelope's `routing` and `security_context` fields map directly to BibleFi's agent taxonomy (see `docs/agents/` in BibleFi/BibleFi):

| Envelope Field | Agent Taxonomy Reference |
|---|---|
| `sender.agent_type` | `AGENT_TAXONOMY.md` — agent type identifiers |
| `routing.priority` | `JOB_CATALOG.md` — job priority levels |
| `security_context.trust_level` | `TRUST_BOUNDARIES.md` — trust tier definitions |
| `security_context.sandboxed` | `SANDBOXING_POLICY.md` — all agents run sandboxed |
| `payload_type` | `OUTPUT_CONTRACT.md` — valid output payload types |

**Example: Scripture agent sending a verse to the tithe agent:**

```json
{
  "envelope_id": "550e8400-e29b-41d4-a716-446655440000",
  "schema_version": "2.0.0",
  "created_at": "2026-03-30T03:00:00Z",
  "payload_type": "scripture_record",
  "payload": { "...": "ScriptureRecord object" },
  "sender": {
    "agent_id": "7c7d4a2b-1234-5678-abcd-ef0123456789",
    "agent_type": "scripture_agent"
  },
  "routing": {
    "destination_agent": "tithe_agent",
    "priority": "normal",
    "correlation_id": "a1b2c3d4-0000-0000-0000-000000000001"
  },
  "security_context": {
    "classification": "internal",
    "trust_level": "high",
    "sandboxed": true
  },
  "base_chain_context": {
    "chain_id": 8453
  }
}
```

---

## 7. Validation Example (Node.js + Ajv)

```js
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';
import { readFileSync } from 'fs';

const ajv = new Ajv({ strict: true });
addFormats(ajv);

// Load and compile a schema
const scriptureSchema = JSON.parse(
  readFileSync('./schemas/scripture_record.schema.json', 'utf8')
);
const validate = ajv.compile(scriptureSchema);

// Validate a record
const record = {
  record_id: '550e8400-e29b-41d4-a716-446655440001',
  schema_version: '2.0.0',
  book: 'John',
  chapter: 3,
  verses: 16,
  translation: {
    abbreviation: 'NIV',
    full_name: 'New International Version',
    language: 'en',
    year_published: 1978,
    manuscript_tradition: 'nestle_aland'
  },
  text: 'For God so loved the world that he gave his one and only Son, ' +
        'that whoever believes in him shall not perish but have eternal life.',
  content_hash: '0x' + '0'.repeat(64),
  stewardship_tags: ['generosity'],
  hermeneutics: {
    literary_genre: 'gospel',
    covenant_context: 'new_covenant',
    canonical_position: 'nt_gospel'
  }
};

const valid = validate(record);
if (!valid) {
  console.error('Validation errors:', validate.errors);
} else {
  console.log('Record is valid!');
}
```

---

## 8. Complete Valid Example Payloads

### TitheTransaction — Malachi 3:10 tithe on Base chain

```json
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440010",
  "schema_version": "1.0.0",
  "sender_wallet": "0xAbCd1234567890AbCd1234567890AbCd12345678",
  "recipient_wallet": "0xEf0123456789AbCd0123456789AbCdEf01234567",
  "recipient_church_id": "9b1d6fea-a1e2-4b3c-8d4e-1f2a3b4c5d6e",
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
  "tx_hash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
  "block_number": 24000000,
  "transaction_type": "tithe",
  "scripture_reference": {
    "book": "Malachi",
    "chapter": 3,
    "verses": 10
  },
  "percentage_of_income": 10,
  "is_recurring": true,
  "recurrence_interval": "monthly",
  "smart_contract_routed": true,
  "contract_address": "0xBibleFiRouter000000000000000000000000000",
  "attestation": {
    "chain_id": 8453,
    "contract_address": "0xBibleFiAttest00000000000000000000000000",
    "attestation_type": "eas_attestation",
    "tx_hash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
    "block_number": 24000001,
    "attested_at": "2026-03-30T03:05:00Z",
    "attested_by": "BibleFi Protocol"
  },
  "notes": "Monthly tithe — Malachi 3:10 — Bring the whole tithe into the storehouse",
  "tags": ["tithe", "monthly", "USDC", "base"],
  "created_at": "2026-03-30T03:00:00Z",
  "updated_at": "2026-03-30T03:05:00Z"
}
```

### FarcasterFrameEvent — tithe_initiate action

```json
{
  "event_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "schema_version": "1.0.0",
  "fid": 12345,
  "username": "biblefi_user",
  "frame_action": "tithe",
  "frame_uri": "https://biblefi.app/frames/tithe/malachi-3-10",
  "cast_hash": "0xcast1234567890abcdef",
  "channel_id": "/biblefi",
  "timestamp": "2026-03-30T03:00:00Z",
  "payload": {
    "button_index": 1,
    "input_text": ""
  },
  "connected_wallet": "0xAbCd1234567890AbCd1234567890AbCd12345678",
  "chain_id": 8453,
  "biblefi_action": {
    "action_type": "tithe_initiate",
    "scripture_record_id": "550e8400-e29b-41d4-a716-446655440001",
    "church_id": "9b1d6fea-a1e2-4b3c-8d4e-1f2a3b4c5d6e",
    "amount_wei": "10000000"
  },
  "created_at": "2026-03-30T03:00:00Z",
  "updated_at": "2026-03-30T03:00:00Z"
}
```

---

## Scripture References for Stewardship

BibleFi agents use these scriptures to back giving transactions:

| Scripture | Category | Notes |
|---|---|---|
| Malachi 3:10 | Tithe | "Bring the whole tithe into the storehouse" |
| Proverbs 3:9-10 | Firstfruits | "Honor the LORD with your wealth, with the firstfruits of all your crops" |
| 2 Corinthians 9:7 | Offering | "God loves a cheerful giver" |
| Luke 21:1-4 | Widow's Offering | Sacrificial giving beyond means |
| Genesis 28:22 | Vow Tithe | Jacob's vow — "a tenth of all" |

