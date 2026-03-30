# BibleFi Schema Versioning Policy

All BibleFi schemas follow **Semantic Versioning 2.0.0** ([semver.org](https://semver.org)).

---

## SemVer Policy

| Change Type | Version Bump | Example |
|-------------|-------------|---------|
| New optional fields, backwards-compatible fixes | **PATCH** | 1.0.0 → 1.0.1 |
| New optional sections, new enum values | **MINOR** | 1.0.0 → 1.1.0 |
| Removed fields, changed types, new required fields, renamed fields | **MAJOR** | 1.0.0 → 2.0.0 |

### Rules
1. **PATCH** changes are always safe to deploy without consumer updates.
2. **MINOR** changes are backwards-compatible. Consumers using `additionalProperties: false` must update their local schema copies.
3. **MAJOR** changes break backwards compatibility. A migration guide must be published in `docs/` before or simultaneously with the schema update.
4. The `$id` URI includes the **major version** (e.g. `https://schemas.biblefi.io/v3/scripture_record.schema.json`). Minor and patch bumps share the same `$id` base URI.

---

## Deprecation Policy

- Deprecated schemas are retained for a **minimum of 24 months** from the deprecation date.
- The deprecation date and removal target are noted in the schema's `description` field.
- Deprecated schemas receive security fixes but no new features.
- Consumers are notified via GitHub release notes and the deprecation table below.

---

## Current Schema Versions

| Schema File | Version | Status | Stable Since | Notes |
|-------------|---------|--------|--------------|-------|
| `agent_envelope.schema.json` | 1.0.0 | ✅ Stable | 2024-01 | Universal BibleFi payload envelope |
| `church_record.schema.json` | 1.0.0 | ✅ Stable | 2024-01 | Church / faith community record |
| `defi_strategy.schema.json` | 1.0.0 | ✅ Stable | 2024-01 | DeFi tithing and giving strategy |
| `scripture_record.schema.json` | 3.0.0 | ✅ Stable | 2026-03 | Centerpiece — AI + crypto + hermeneutics |
| `security_finding.schema.json` | 1.0.0 | ✅ Stable | 2024-01 | Smart contract audit findings |
| `user_profile.schema.json` | 1.0.0 | ✅ Stable | 2024-01 | BibleFi user profile |
| `wallet_record.schema.json` | 1.0.0 | ✅ Stable | 2024-01 | EVM wallet record |

---

## Deprecated Schemas

| Schema File | Version | Deprecated | Removal Target | Successor |
|-------------|---------|------------|----------------|-----------|
| `scripture_record.v1.schema.json` | 1.0.0 | 2026-03 | 2028-03 | `scripture_record.schema.json` v3.0.0 |

---

## Migration Guides

| Migration | Document |
|-----------|----------|
| `scripture_record` v2 → v3 | [`docs/scripture_record_v3_guide.md`](docs/scripture_record_v3_guide.md) |
