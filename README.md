# BibleFi Schemas

Canonical [JSON Schema](https://json-schema.org/) definitions for the BibleFi protocol. Every message exchanged between agents, every record stored on-chain, and every payload surfaced by BibleFi APIs must conform to one of these schemas.

BibleFi is the world's first Christian-faith-based DeFi dApp, built on **Base chain** (`chain_id: 8453`) and deployed as a native Base App and **Farcaster mini-app**. These schemas encode biblical stewardship principles — tithing (Malachi 3:10), generosity (2 Cor 9:7), firstfruits (Proverbs 3:9-10) — into verifiable, on-chain data structures.

---

## Schema Catalog

| File | Version | Status | Description |
|------|---------|--------|-------------|
| `schemas/scripture_record.schema.json` | 2.0.0 | ✅ Stable | Bible passages with content hash (Keccak-256), AI metadata, hermeneutics, cross-references, and Base chain attestations |
| `schemas/scripture_record.v1.schema.json` | 1.0.0 | ⚠️ Deprecated | Legacy passage record — retained until 2028-03 per versioning policy |
| `schemas/tithe_transaction.schema.json` | 1.0.0 | ✅ Stable | Biblical giving transactions (tithe, offering, firstfruits) on Base chain |
| `schemas/farcaster_frame_event.schema.json` | 1.0.0 | ✅ Stable | Farcaster Frame interaction events for BibleFi mini-app |
| `schemas/agent_envelope.schema.json` | 2.0.0 | ✅ Stable | Universal agent message wrapper with routing, security context, Farcaster and Base chain context |
| `schemas/church_record.schema.json` | 1.0.0 | ✅ Stable | Church profiles with on-chain treasury, location, and contact info |
| `schemas/defi_strategy.schema.json` | 1.0.0 | ✅ Stable | DeFi yield strategies with protocol, asset, tithe allocation, and rebalance policy |
| `schemas/security_finding.schema.json` | 1.0.0 | ✅ Stable | Security audit findings with CVSS and CWE |
| `schemas/wallet_record.schema.json` | 1.0.0 | ✅ Stable | On-chain wallet identity records |
| `schemas/user_profile.schema.json` | 1.0.0 | ✅ Stable | BibleFi user profiles |
| `schemas/agent_task.schema.json` | 1.0.0 | ✅ Stable | Agentic pipeline task specifications |
| `schemas/agent_run_log.schema.json` | 1.0.0 | ✅ Stable | Hourly execution cycle run logs |
| `schemas/scripture_seed_batch.schema.json` | 1.0.0 | ✅ Stable | Batches of validated scriptures seeded to the dApp |
| `schemas/cross_language_validation.schema.json` | 1.0.0 | ✅ Stable | Hebrew / Greek / Aramaic cross-reference results |
| `schemas/theological_validation.schema.json` | 1.0.0 | ✅ Stable | Concordance & dictionary validation results |

---

## Quick Start

All schemas use **JSON Schema Draft 2020-12**. Most schemas enforce `additionalProperties: false` throughout; the current exception is `schemas/agent_envelope.schema.json`, where `AgentEnvelope.payload` intentionally allows a flexible object shape and payload validation is performed out-of-band using the per-type schema.

```bash
npm install ajv ajv-formats
```

```javascript
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';
import { readFileSync } from 'fs';

const ajv = new Ajv();
addFormats(ajv);

const schema = JSON.parse(readFileSync('./schemas/tithe_transaction.schema.json', 'utf8'));
const validate = ajv.compile(schema);

const tithe = {
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "schema_version": "1.0.0",
  "sender_wallet": "0x1234567890123456789012345678901234567890",
  "recipient_wallet": "0xabcdef0123456789abcdef0123456789abcdef01",
  "amount_wei": "10000000",
  "asset": {
    "symbol": "USDC",
    "contract_address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "chain_id": 8453,
    "decimals": 6
  },
  "chain_id": 8453,
  "transaction_type": "tithe",
  "scripture_reference": { "book": "Malachi", "chapter": 3, "verses": 10 }
};

console.log(validate(tithe)); // true
```

---

## Key Concepts

### Content Hash (scripture_record v2)

Every scripture record includes a `content_hash` — a Keccak-256 hash of the NFC-normalized, trimmed UTF-8 text. This enables on-chain integrity verification via BibleFi attestation contracts on Base chain.

```javascript
import { keccak256, toBytes } from 'viem';
const hash = keccak256(toBytes(text.normalize('NFC').trim()));
```

### Tithing Flow

`FarcasterFrameEvent (tithe_initiate)` → `TitheTransaction` → `Base chain tx` → `EAS attestation`

See [docs/biblefi_schema_guide.md](docs/biblefi_schema_guide.md) for the complete tithing flow and example payloads.

### Agent Envelopes

All agent-to-agent messages are wrapped in `AgentEnvelope` v2.0.0, which adds routing, security context (classification, trust level, sandboxing), Farcaster context, and Base chain context.

### Agentic Pipeline

The BibleFi hourly scripture-seeding pipeline uses `agent_task`, `agent_run_log`, `scripture_seed_batch`, `cross_language_validation`, and `theological_validation` schemas. See [ARCHITECTURE.md](./ARCHITECTURE.md) for a full description.

---

## Validation Example (Node.js / Ajv)

```js
import Ajv from "ajv/dist/2020.js";
import addFormats from "ajv-formats";
import envelope from "./schemas/agent_envelope.schema.json" assert { type: "json" };

const ajv = new Ajv({ strict: true });
addFormats(ajv);

const validate = ajv.compile(envelope);

const payload = {
  envelope_id: "550e8400-e29b-41d4-a716-446655440000",
  schema_version: "2.0.0",
  created_at: "2024-01-01T00:00:00Z",
  payload_type: "scripture_record",
  payload: { record_id: "660e8400-e29b-41d4-a716-446655440001" }
};

const valid = validate(payload);
if (!valid) console.error(validate.errors);
```

---

## Versioning

See [VERSIONING.md](./VERSIONING.md) for the full versioning policy. In summary:

| Change type | Version bump |
|-------------|-------------|
| Bug-fix / description update | PATCH (`1.0.x`) |
| New optional field | MINOR (`1.x.0`) |
| Breaking change | MAJOR (`x.0.0`) |

---

## Documentation

- [Schema Guide](docs/biblefi_schema_guide.md) — Full guide with examples, content hash computation, tithing flow, and Ajv validation
- [Versioning Policy](VERSIONING.md) — Schema versioning, deprecation timelines, and migration notes
- [Architecture](ARCHITECTURE.md) — Agentic pipeline architecture & scalability guide

---

## Biblical Foundation

> *"Bring the whole tithe into the storehouse, that there may be food in my house."* — Malachi 3:10

> *"Honor the Lord with your wealth, with the firstfruits of all your crops."* — Proverbs 3:9

> *"Each of you should give what you have decided in your heart to give, not reluctantly or under compulsion, for God loves a cheerful giver."* — 2 Corinthians 9:7

---

## Contributing

1. Fork the repository and create a feature branch.
2. Edit the relevant `.schema.json` file(s) under `schemas/`.
3. Bump the `schema_version` field in the modified schemas according to [VERSIONING.md](./VERSIONING.md).
4. Open a pull request with a clear description of the change and its rationale.

---

## License

[MIT](./LICENSE)
