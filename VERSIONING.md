> ⚠️ `scripture_record` has been upgraded to v3.0.0 (non-breaking superset of v2). See `docs/scripture_record_v3_guide.md` for migration instructions. v1.0.0 is retained at `schemas/scripture_record.v1.schema.json` until 2028-03.

# Schema Versioning

## Current Schema Versions

| Schema | Version | Status |
|---|---|---|
| `scripture_record` | `3.0.0` | Stable |
| `agent_envelope` | `1.0.0` | Stable |
| `church_record` | `1.0.0` | Stable |
| `defi_strategy` | `1.0.0` | Stable |
| `security_finding` | `1.0.0` | Stable |

## Changelog

### v3.0.0 (2026-03-30)
- Added 10 new optional top-level sections: `zero_knowledge_proof`, `canonical_graph`, `semantic_diff`, `liturgical_context`, `multi_sig_governance`, `recitation_metrics`, `interoperability`, `covenant_map`, `dispute_resolution`, `embedding_index`.
- Added `hash_algorithm` field (sibling of `content_hash`).
- Added `canonical_verse_count` field (sibling of `verses`).
- Upgraded `revision_history` item structure: new required fields `revision_id`, `revised_at`, `revised_by`; extended `change_type` enum.
- `$id` updated to `https://schemas.biblefi.io/v3/scripture_record.schema.json`.
- All v2 records remain valid under v3 (non-breaking).

### v2.0.0
- Added `content_hash` (required), `ai_metadata`, `hermeneutics`, `cross_references`, `translation_graph`, `attestations` array, `revision_history`, `provenance`.
- `$id` updated to `https://schemas.biblefi.io/v2/scripture_record.schema.json`.

### v1.0.0
- Initial release.

## Deprecated Schemas

| Schema | Version | Retained Until | Replacement |
|---|---|---|---|
| `scripture_record` (v2) | `2.0.0` | 2029-03 | `scripture_record` v3.0.0 |
| `scripture_record` (v1) | `1.0.0` | 2028-03 | `scripture_record` v3.0.0 |
