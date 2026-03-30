# BibleFi Schema Versioning Policy

## Principles

BibleFi schemas follow [Semantic Versioning (SemVer)](https://semver.org/):

- **MAJOR** — Breaking changes (removed fields, changed types, tightened constraints)
- **MINOR** — Backward-compatible additions (new optional fields, new enum values)
- **PATCH** — Non-breaking fixes (typos in descriptions, clarifications, example updates)

## Current Schema Versions

| Schema | Version | Status | Notes |
|--------|---------|--------|-------|
| `scripture_record` | 2.0.0 | ✅ Stable | Supersedes v1.0.0. Adds content_hash (required), attestations[], hermeneutics, AI metadata, stewardship_tags |
| `scripture_record` (v1 compat) | 1.0.0 | ⚠️ Deprecated | Retained until 2028-03 per 24-month policy. File: `scripture_record.v1.schema.json` |
| `tithe_transaction` | 1.0.0 | ✅ Stable | New in v2 suite |
| `farcaster_frame_event` | 1.0.0 | ✅ Stable | New in v2 suite |
| `agent_envelope` | 2.0.0 | ✅ Stable | Adds routing, security_context, farcaster_context, base_chain_context |
| `church_record` | 1.0.0 | ✅ Stable | |
| `defi_strategy` | 1.0.0 | ✅ Stable | |
| `security_finding` | 1.0.0 | ✅ Stable | |
| `wallet_record` | 1.0.0 | ✅ Stable | |
| `user_profile` | 1.0.0 | ✅ Stable | |

## Migration: scripture_record v1.0.0 → v2.0.0

### Breaking Changes

| Change | v1.0.0 | v2.0.0 |
|--------|--------|--------|
| `content_hash` | Not present | **Required** — Keccak-256 of NFC-normalized trimmed text |
| `attestation` | Single object `{chain_id, contract_address, token_id}` | Replaced by `attestations[]` array supporting multiple attestation types |
| `translation.dialect` | Not present | Optional string |
| `translation.manuscript_tradition` | Not present | Optional enum |

### Migration Steps

1. **Compute `content_hash`** for each existing record:
   ```javascript
   import { keccak256, toBytes } from 'viem';
   record.content_hash = keccak256(toBytes(record.text.normalize('NFC').trim()));
   ```

2. **Migrate `attestation` → `attestations[]`** (if present):
   ```javascript
   if (record.attestation) {
     record.attestations = [{
       chain_id: record.attestation.chain_id,
       contract_address: record.attestation.contract_address,
       token_id: record.attestation.token_id,
       tx_hash: record.attestation.tx_hash,
       attestation_type: "nft_mint"
     }];
     delete record.attestation;
   }
   ```

3. **Update `schema_version`** from `"1.0.0"` to `"2.0.0"`.

## Migration: agent_envelope v1.0.0 → v2.0.0

### Non-Breaking Additions

All new fields in `agent_envelope` v2.0.0 are optional:
- `routing` — Message routing metadata
- `security_context` — Classification, trust level, sandboxing
- `farcaster_context` — Farcaster Frame origin metadata
- `base_chain_context` — Base chain transaction context

### Updated payload_type Enum

The `payload_type` enum now includes:
`tithe_transaction`, `farcaster_frame_event`, `governance_vote`, `wallet_record`, `user_profile`

If your producer sets `payload_type` to an unlisted value, update to the new enum.

## Deprecation Policy

Deprecated schemas are retained for **24 months** after their deprecation date. After that period, they may be removed.

| Schema | Deprecated | Removal Date |
|--------|-----------|--------------|
| `scripture_record` v1.0.0 | 2026-03 | 2028-03 |

## $id Versioning Convention

Stable schemas use versioned `$id` URIs:
- v2: `https://schemas.biblefi.io/v2/scripture_record.schema.json`
- v1 (deprecated): `https://schemas.biblefi.io/v1/scripture_record.schema.json`

New schemas without a breaking-change history use unversioned URIs:
- `https://schemas.biblefi.io/tithe_transaction.schema.json`
- `https://schemas.biblefi.io/farcaster_frame_event.schema.json`
