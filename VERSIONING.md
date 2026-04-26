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

New schemas at v1.0.0 use a `/v1/` prefixed URI:

```
https://schemas.biblefi.io/v1/<schema-name>.schema.json
```

> **Consumer guidance:** Consumers MUST verify that the major version in a payload's `schema_version` field matches the major version in the `$id` they are validating against. The pattern `^\d+\.\d+\.\d+$` does not enforce this constraint itself.

---

## Semantic Versioning Rules

| Increment | When to use | Examples |
|-----------|-------------|---------|
| **PATCH** (`x.y.Z`) | Backward-compatible fixes: correcting descriptions, tightening regex patterns that were already implicitly required, fixing typos. | `1.0.0` → `1.0.1` |
| **MINOR** (`x.Y.z`) | Backward-compatible additions: new **optional** properties, relaxing an existing constraint, adding new permitted `enum` values. | `1.0.1` → `1.1.0` |
| **MAJOR** (`X.y.z`) | Breaking changes: removing or renaming required properties, adding new required properties, narrowing existing constraints, changing a property's type, restructuring nested objects. | `1.1.0` → `2.0.0` |

---

## Change Process

1. **Propose** – Open a GitHub issue describing the schema change and its justification.
2. **Draft** – Submit a pull request with the schema changes and an updated `schema_version` value in all affected `.schema.json` files.
3. **Review** – At least one maintainer must approve the PR. Breaking changes (MAJOR) require sign-off from two maintainers.
4. **Changelog** – Update `CHANGELOG.md` (if present) with a summary under the new version heading.
5. **Merge & Tag** – Merge to `main` and create a Git tag following the pattern `v<MAJOR>.<MINOR>.<PATCH>` (e.g. `v2.0.0`).

---

## Compatibility Guarantees

- **Within a MAJOR version**, all schema changes are backward-compatible. A consumer validating with schema `1.0.0` will also successfully validate documents produced against `1.3.2`.
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
| `$id` | `https://schemas.biblefi.io/v1/scripture_record.schema.json` | `https://schemas.biblefi.io/v2/scripture_record.schema.json` |

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

### Breaking Changes

| Change | v1.x | v2.0.0 |
|--------|------|--------|
| `correlation_id` (top-level) | Optional free string | **Removed**. Use `routing.correlation_id` (UUID v4) instead. |
| `$id` | `https://schemas.biblefi.io/agent_envelope.schema.json` | `https://schemas.biblefi.io/v2/agent_envelope.schema.json` |

> **IMPORTANT:** Because `agent_envelope` v2.0.0 sets `additionalProperties: false`, any v1 payload that includes a top-level `correlation_id` field will **fail** v2 validation. Producers must move `correlation_id` into the `routing` object before upgrading to v2.

### Migration Steps for `correlation_id`

```javascript
// Before (v1.x)
const envelope = {
  correlation_id: "my-workflow-id",   // top-level, free string
  // ...other fields
};

// After (v2.0.0)
const envelope = {
  routing: {
    correlation_id: "550e8400-e29b-41d4-a716-446655440000",  // UUID v4 required
    // ...other routing fields
  },
  // ...other fields
};
```

### Non-Breaking Additions in v2.0.0

All other new fields in `agent_envelope` v2.0.0 are optional and non-breaking:
- `routing` — Message routing metadata (destination/source agent, priority, TTL, correlation_id)
- `security_context` — Classification, trust level, sandboxing metadata
- `farcaster_context` — Farcaster Frame origin metadata
- `base_chain_context` — Base chain transaction context
- `sender.fid` — Farcaster ID for the sender
- `signature.signed_at` — Timestamp when the signature was created

### Updated `payload_type` Enum

The `payload_type` enum in v2.0.0 now includes all supported schemas:
`scripture_record`, `church_record`, `defi_strategy`, `security_finding`, `wallet_record`, `user_profile`, `tithe_transaction`, `farcaster_frame_event`, `agent_task`, `agent_run_log`, `scripture_seed_batch`, `cross_language_validation`, `theological_validation`, `generic`

If your producer sets `payload_type` to a value not in this enum, update to one of the listed values.

---

## Security Notes

### `security_context` Self-Assertion Warning

The `security_context` object in `agent_envelope` v2.0.0 carries classification and trust metadata. **All fields are self-asserted by the sender** and are not cryptographically enforced by the schema.

Consumers **MUST NOT** use `security_context` fields for:
- Access-control or authorization decisions
- Privilege escalation
- Bypassing sandboxing policies

Instead, verify the envelope's `signature` against a trusted key registry and enforce trust boundaries using server-side policy engines. See `TRUST_BOUNDARIES.md` and `SANDBOXING_POLICY.md` in `Biblefi/BibleFi`.

### `format` Keyword Enforcement

BibleFi schemas use `format: uuid`, `format: uri`, `format: date-time`, etc. The JSON Schema specification does not require validators to assert `format` constraints by default. Enforcement depends on the validator and its configuration:

| Validator | Default behavior | To enable assertions |
|-----------|-----------------|---------------------|
| Ajv | Annotation only | Use `ajv-formats` with `mode: "full"` |
| jsonschema (Python) | Annotation only | Use `format_checker=FormatChecker()` |

If strict format enforcement is required (e.g. for UUID validation), add a `pattern` constraint alongside `format`. Example for UUID:
```json
{
  "type": "string",
  "format": "uuid",
  "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$"
}
```

### `signed_at` Replay Prevention

The `signature.signed_at` field is a self-reported, client-side timestamp. Recipients **MUST NOT** use `signed_at` alone for replay detection. Use server-side time validation combined with `expires_at` and nonce tracking.

---

## Deprecation Policy

Deprecated schemas are retained for **24 months** after their deprecation date. After that period, they may be removed.

| Schema | Deprecated | Removal Date |
|--------|-----------|--------------|
| `scripture_record` v1.0.0 | 2026-03 | 2028-03 |

---

## Pre-Release Versions

Pre-release schemas use the standard SemVer pre-release suffix (e.g. `1.0.0-alpha.1`, `1.0.0-rc.1`) and **must not** be used in production deployments. They are published for review and integration testing only.
