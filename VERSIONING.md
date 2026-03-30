# BibleFi Schema Versioning Policy

All BibleFi schemas follow **semantic versioning** (`MAJOR.MINOR.PATCH`) with the following rules:

| Change Type | Version Bump | Example |
|---|---|---|
| Breaking change (required field added, type changed) | MAJOR | 1.0.0 → 2.0.0 |
| Non-breaking addition (new optional field) | MINOR | 1.0.0 → 1.1.0 |
| Bug fix or documentation only | PATCH | 1.0.0 → 1.0.1 |

---

## Deprecation Policy

- Deprecated schemas are retained for **24 months** after the deprecation date.
- Deprecated schemas carry a `DEPRECATED` notice in their `description` field with the expiry date.
- Consumers must migrate within the 24-month window.

---

## Current Schema Versions

| Schema | Current Version | Previous Version | Deprecated Until |
|---|---|---|---|
| `agent_envelope` | 2.0.0 | — | — |
| `scripture_record` | 2.0.0 | 1.0.0 | 2028-03 |
| `defi_strategy` | 2.0.0 | — | — |
| `church_record` | 2.0.0 | — | — |
| `tithe_transaction` | 1.0.0 | — | — |
| `farcaster_frame_event` | 1.0.0 | — | — |
| `wallet_record` | 1.0.0 | — | — |
| `user_profile` | 1.0.0 | — | — |
| `security_finding` | 1.0.0 | — | — |

---

## v2.0.0 Migration Warnings

### `scripture_record` v1.0.0 → v2.0.0

⚠️ **Breaking changes:**

1. **`content_hash` is now required.** Compute: `keccak256(normalize_nfc(text.trim()))`. See `docs/biblefi_schema_guide.md` for algorithm details and code examples.
2. **`attestation` (singular) replaced by `attestations` (array).** Update any consumer reading `record.attestation` to `record.attestations[0]`.
3. **`translation.manuscript_tradition` added** (optional but recommended).

**New optional fields:** `ai_metadata`, `hermeneutics`, `cross_references`, `translation_graph`, `provenance`, `stewardship_tags`, `revision_history`, `tags`.

### `defi_strategy` v1.x → v2.0.0

⚠️ **Breaking changes:**

1. **`content_hash` is now required.** Compute Keccak-256 of canonical strategy JSON.

**New optional fields:** `biblefi_vault_integration`, `performance_tracking`, `risk_parameters`, `farcaster_frame_uri`, `attestations`, `approved_by`, `tags`, `created_at`, `updated_at`.

### `church_record` v1.x → v2.0.0

No breaking changes. All new fields are optional: `farcaster_identity`, `governance`, `treasury_stats`, `ministry_programs`, `attestations`, `social_proof`, `tags`, `created_at`, `updated_at`.

### `agent_envelope` v1.x → v2.0.0

No breaking changes. New `payload_type` enum values added: `wallet_record`, `user_profile`, `tithe_transaction`, `governance_vote`, `farcaster_frame_event`. New optional fields: `routing`, `security_context`, `farcaster_context`, `base_chain_context`.

---

## $id URI Convention

- Current schemas: `https://schemas.biblefi.io/<name>.schema.json` (for v1 schemas)
- v2 schemas: `https://schemas.biblefi.io/v2/<name>.schema.json`
- v1 deprecated: `https://schemas.biblefi.io/v1/<name>.schema.json`
