# BibleFi Schema Versioning Policy

BibleFi schemas use [Semantic Versioning 2.0.0](https://semver.org/) (`MAJOR.MINOR.PATCH`) to communicate the scope and impact of every change.

---

## Version Fields

Every schema document contains a top-level `schema_version` field (type `string`, pattern `^\d+\.\d+\.\d+$`).
Every JSON payload **must** declare the version of the schema it was validated against so consumers can apply the correct validation logic.

The `$id` URI of each schema file also encodes a stable identifier. When a MAJOR version bump occurs the URI will be updated to include the major version:

```
https://schemas.biblefi.io/v2/<schema-name>.schema.json
```

---

## Semantic Versioning Rules

| Increment | When to use | Examples |
|-----------|-------------|---------|
| **PATCH** (`x.y.Z`) | Backward-compatible fixes: correcting descriptions, tightening regex patterns, fixing typos. | `1.0.0` → `1.0.1` |
| **MINOR** (`x.Y.z`) | Backward-compatible additions: new **optional** properties, relaxing an existing constraint, adding new permitted `enum` values. | `1.0.1` → `1.1.0` |
| **MAJOR** (`X.y.z`) | Breaking changes: removing or renaming required properties, adding new required properties, narrowing existing constraints, changing a property's type. | `1.1.0` → `2.0.0` |

---

## Change Process

1. **Propose** – Open a GitHub issue describing the schema change and its justification.
2. **Draft** – Submit a pull request with the schema changes and an updated `schema_version` value in all affected `.schema.json` files.
3. **Review** – At least one maintainer must approve the PR. Breaking changes (MAJOR) require sign-off from two maintainers.
4. **Changelog** – Update `CHANGELOG.md` (if present) with a summary under the new version heading.
5. **Merge & Tag** – Merge to `main` and create a Git tag following the pattern `v<MAJOR>.<MINOR>.<PATCH>` (e.g. `v2.0.0`).

---

## Compatibility Guarantees

- **Within a MAJOR version**, all schema changes are backward-compatible.
- **Across MAJOR versions**, no compatibility is guaranteed. Consumer applications must explicitly migrate to the new MAJOR version.
- Schema files will be retained for **at least 24 months** after a new MAJOR version is published to allow migration.

---

## Current Schema Versions

| Schema | Version | Status | Notes |
|--------|---------|--------|-------|
| `scripture_record` | 2.0.0 | ✅ Stable | Supersedes v1.0.0. Adds content_hash (required), attestations[], hermeneutics, AI metadata, stewardship_tags |
| `scripture_record` (v1 compat) | 1.0.0 | ⚠️ Deprecated | Retained until 2028-03 per 24-month policy. File: `scripture_record.v1.schema.json` |
| `tithe_transaction` | 1.0.0 | ✅ Stable | New in v2 suite |
| `farcaster_frame_event` | 1.0.0 | ✅ Stable | New in v2 suite |
| `agent_envelope` | 2.0.0 | ✅ Stable | Adds routing, security_context, farcaster_context, base_chain_context; extended payload_type enum |
| `church_record` | 1.0.0 | ✅ Stable | |
| `defi_strategy` | 1.0.0 | ✅ Stable | |
| `security_finding` | 1.0.0 | ✅ Stable | |
| `wallet_record` | 1.0.0 | ✅ Stable | |
| `user_profile` | 1.0.0 | ✅ Stable | |
| `agent_task` | 1.0.0 | ✅ Stable | Agentic pipeline |
| `agent_run_log` | 1.0.0 | ✅ Stable | Agentic pipeline |
| `scripture_seed_batch` | 1.0.0 | ✅ Stable | Agentic pipeline |
| `cross_language_validation` | 1.0.0 | ✅ Stable | Agentic pipeline |
| `theological_validation` | 1.0.0 | ✅ Stable | Agentic pipeline |

---

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

---

## Migration: agent_envelope v1.x → v2.0.0

### Non-Breaking Additions

All new fields in `agent_envelope` v2.0.0 are optional:
- `routing` — Message routing metadata (destination/source agent, priority, TTL, correlation_id)
- `security_context` — Classification, trust level, sandboxing
- `farcaster_context` — Farcaster Frame origin metadata
- `base_chain_context` — Base chain transaction context

### Updated payload_type Enum

The `payload_type` enum in v2.0.0 now includes all supported schemas:
`scripture_record`, `church_record`, `defi_strategy`, `security_finding`, `wallet_record`, `user_profile`, `tithe_transaction`, `farcaster_frame_event`, `agent_task`, `agent_run_log`, `scripture_seed_batch`, `cross_language_validation`, `theological_validation`, `generic`

---

## Deprecation Policy

Deprecated schemas are retained for **24 months** after their deprecation date. After that period, they may be removed.

| Schema | Deprecated | Removal Date |
|--------|-----------|--------------|
| `scripture_record` v1.0.0 | 2026-03 | 2028-03 |

---

## Pre-Release Versions

Pre-release schemas use the standard SemVer pre-release suffix (e.g. `1.0.0-alpha.1`, `1.0.0-rc.1`) and **must not** be used in production deployments.
