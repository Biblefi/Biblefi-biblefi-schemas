# BibleFi Schemas

Canonical [JSON Schema](https://json-schema.org/) (Draft 2020-12) definitions for the BibleFi protocol — the world's first Christian-faith-based DeFi dApp, built on **Base chain** and deployed as a **native Base App** and **Farcaster mini-app**.

Every message exchanged between agents, every record stored on-chain, and every payload surfaced by BibleFi APIs must conform to one of these schemas.

---

## Repository Layout

```
schemas/
├── agent_envelope.schema.json          # v2.0.0 — Universal message wrapper
├── scripture_record.schema.json        # v2.0.0 — Bible passage records
├── scripture_record.v1.schema.json     # v1.0.0 — DEPRECATED (retained until 2028-03)
├── defi_strategy.schema.json           # v2.0.0 — DeFi yield/staking strategies
├── church_record.schema.json           # v2.0.0 — Church/ministry profiles
├── tithe_transaction.schema.json       # v1.0.0 — On-chain tithe transactions (NEW)
├── farcaster_frame_event.schema.json   # v1.0.0 — Farcaster Frame events (NEW)
├── wallet_record.schema.json           # v1.0.0 — EVM wallet identity
├── user_profile.schema.json            # v1.0.0 — User profiles
└── security_finding.schema.json        # v1.0.0 — Security audit findings
docs/
└── biblefi_schema_guide.md             # Comprehensive schema guide
VERSIONING.md                           # Schema versioning policy
```

---

## Schema Descriptions

### `agent_envelope` — v2.0.0
Universal message wrapper for all BibleFi agent communications. Carries any BibleFi payload type with cryptographic signature, routing metadata, security context, Farcaster context, and Base chain context. Key fields: `envelope_id`, `payload_type`, `payload`, `sender`, `signature`, `routing` (destination, priority, TTL), `security_context` (classification, trust_level, sandboxed), `farcaster_context`, `base_chain_context` (chain_id defaults to 8453).

### `scripture_record` — v2.0.0
Full Bible passage record with translation metadata, AI analysis, hermeneutics, cross-references, and on-chain attestations. Key fields: `record_id`, `book`, `chapter`, `verses`, `text`, `content_hash` (Keccak-256 for Base chain integrity), `translation` (with manuscript_tradition), `ai_metadata` (theological_themes, sentiment, named_entities), `hermeneutics` (literary_genre, covenant_context, interpretive_traditions), `attestations` (nft_mint, eas_attestation, hypercert, farcaster_frame), `stewardship_tags`, `cross_references`.

### `defi_strategy` — v2.0.0
DeFi yield and staking strategy with Base chain integration. Key fields: `strategy_id`, `protocol`, `asset`, `allocation`, `risk_tier`, `expected_apy`, `tithe_allocation`, `rebalance_policy`, `content_hash`, `biblefi_vault_integration` (vault_type, auto_tithe_enabled, tithe_percentage, scripture_backing), `performance_tracking` (nav_usd, current_apy), `risk_parameters` (max_drawdown, stop_loss, slippage_tolerance_bps), `farcaster_frame_uri`, `attestations`, `approved_by`.

### `church_record` — v2.0.0
Church or ministry organization profile. Key fields: `record_id`, `name`, `denomination`, `location`, `contact`, `on_chain_identity` (wallet_address, ENS, multisig), `verified`, `farcaster_identity` (fid, username, custody_address), `governance` (governance_model, multisig_threshold, dao_enabled), `treasury_stats` (total_balance_usd, tithe_received_ytd), `ministry_programs` (missions, evangelism, youth, etc.), `attestations`, `social_proof`.

### `tithe_transaction` — v1.0.0 ✨ NEW
BibleFi's core on-chain tithe and offering transaction. Key fields: `transaction_id`, `sender_wallet`, `recipient_wallet`, `amount_wei` (uint256 as string), `asset` (USDC, ETH, BFI on Base), `chain_id` (Base = 8453), `transaction_type` (tithe | firstfruits | offering | almsgiving | missions_gift | building_fund | emergency_relief | scholarship), `scripture_reference` (e.g. Malachi 3:10), `percentage_of_income`, `is_recurring`, `smart_contract_routed`, `attestation`.

### `farcaster_frame_event` — v1.0.0 ✨ NEW
Farcaster Frame interaction event for the BibleFi mini-app. Key fields: `event_id`, `fid` (Farcaster ID), `frame_action` (view | click | tithe | stake | mint | vote | ...), `frame_uri`, `cast_hash`, `channel_id` (e.g. `/biblefi`), `connected_wallet`, `chain_id`, `biblefi_action` (action_type, scripture_record_id, strategy_id, church_id, amount_wei).

### `wallet_record` — v1.0.0
EVM wallet identity for BibleFi users and church treasuries. Key fields: `wallet_id`, `address` (EVM), `ens_name`, `chain_ids`, `wallet_type` (eoa | multisig | smart_wallet | hardware), `owner_id` (→ user_profile), `church_id` (→ church_record), `is_treasury`, `balance_snapshot`.

### `user_profile` — v1.0.0
BibleFi user profile. Key fields: `user_id`, `role` (individual | church_admin | auditor | developer | observer), `wallet_addresses`, `church_id`, `farcaster_fid`, `preferred_translation`, `notification_preferences`, `stewardship_profile` (total_tithe_ytd, giving_streak_days).

### `security_finding` — v1.0.0
Security audit finding per BibleFi THREAT_MODEL.md and INCIDENT_RESPONSE.md. Key fields: `finding_id`, `title`, `severity` (critical | high | medium | low | informational), `status`, `cvss` (version, base_score, vector_string), `cwe` (CWE-ID), `affected_component`, `remediation`.

---

## Current Schema Versions

| Schema | Version | Status |
|---|---|---|
| `agent_envelope` | 2.0.0 | ✅ Current |
| `scripture_record` | 2.0.0 | ✅ Current |
| `scripture_record` (v1) | 1.0.0 | ⚠️ Deprecated (until 2028-03) |
| `defi_strategy` | 2.0.0 | ✅ Current |
| `church_record` | 2.0.0 | ✅ Current |
| `tithe_transaction` | 1.0.0 | ✅ Current |
| `farcaster_frame_event` | 1.0.0 | ✅ Current |
| `wallet_record` | 1.0.0 | ✅ Current |
| `user_profile` | 1.0.0 | ✅ Current |
| `security_finding` | 1.0.0 | ✅ Current |

---

## Validation

All schemas use **JSON Schema Draft 2020-12** with `additionalProperties: false` on all nested objects.

```bash
npm install ajv ajv-formats
node -e "
const Ajv = require('ajv/dist/2020');
const addFormats = require('ajv-formats');
const fs = require('fs');
const ajv = new Ajv({ strict: true });
addFormats(ajv);
const schema = JSON.parse(fs.readFileSync('./schemas/scripture_record.schema.json', 'utf8'));
console.log('Schema valid:', ajv.validateSchema(schema));
"
```

See [`docs/biblefi_schema_guide.md`](docs/biblefi_schema_guide.md) for the complete guide including content hash algorithm, Base chain integration details, tithing flow, and full example payloads.

---

## Related Repositories

- **[BibleFi/BibleFi](https://github.com/Biblefi/BibleFi)** — Main app, agents, architecture, ops, security docs
- **[Biblefi/multilinear-toolkit](https://github.com/Biblefi/multilinear-toolkit)** — Math toolkit
