# Versioning Policy

BibleFi schemas use [Semantic Versioning 2.0.0](https://semver.org/) (`MAJOR.MINOR.PATCH`) to communicate the scope and impact of every change.

---

## Version Fields

Every schema document contains a top-level `schema_version` field (type `string`, pattern `^\d+\.\d+\.\d+$`).  
Every JSON payload **must** declare the version of the schema it was validated against so consumers can apply the correct validation logic.

The `$id` URI of each schema file also encodes a stable identifier:

```
https://schemas.biblefi.io/<schema-name>.schema.json
```

When a MAJOR version bump occurs the URI will be updated to include the major version:

```
https://schemas.biblefi.io/v2/<schema-name>.schema.json
```

---

## Semantic Versioning Rules

| Increment | When to use | Examples |
|-----------|-------------|---------|
| **PATCH** (`x.y.Z`) | Backward-compatible fixes: correcting descriptions, tightening regex patterns that were already implicitly required, fixing typos in `enum` values that were never emitted in production. | `1.0.0` → `1.0.1` |
| **MINOR** (`x.Y.z`) | Backward-compatible additions: new **optional** properties, relaxing an existing constraint (e.g. changing `minLength` from 5 to 1), adding new permitted `enum` values. Existing valid documents remain valid. | `1.0.1` → `1.1.0` |
| **MAJOR** (`X.y.z`) | Breaking changes: removing or renaming required properties, adding new required properties, narrowing existing constraints, changing a property's type, restructuring nested objects. Existing producers and consumers must be updated. | `1.1.0` → `2.0.0` |

---

## Change Process

1. **Propose** – Open a GitHub issue describing the schema change and its justification.
2. **Draft** – Submit a pull request with the schema changes and an updated `schema_version` value in all affected `.schema.json` files.
3. **Review** – At least one maintainer must approve the PR. Breaking changes (MAJOR) require sign-off from two maintainers.
4. **Changelog** – Update `CHANGELOG.md` (if present) with a summary under the new version heading.
5. **Merge & Tag** – Merge to `main` and create a Git tag following the pattern `v<MAJOR>.<MINOR>.<PATCH>` (e.g. `v1.2.0`).
6. **Publish** – The CI pipeline automatically publishes the updated schema package to the configured registry.

---

## Compatibility Guarantees

- **Within a MAJOR version**, all schema changes are backward-compatible. A consumer validating with schema `1.0.0` will also successfully validate documents produced against `1.3.2`.
- **Across MAJOR versions**, no compatibility is guaranteed. Consumer applications must explicitly migrate to the new MAJOR version.
- Schema files will be retained for **at least 24 months** after a new MAJOR version is published to allow migration.

---

## Pre-Release Versions

Pre-release schemas use the standard SemVer pre-release suffix:

```
1.0.0-alpha.1
1.0.0-beta.3
1.0.0-rc.1
```

Pre-release schemas **must not** be used in production deployments. They are published for review and integration testing only.

---

## Current Schema Versions

| Schema | Current Version | Status |
|--------|-----------------|--------|
| `agent_envelope` | `1.0.0` | Stable |
| `scripture_record` | `1.0.0` | Stable |
| `church_record` | `1.0.0` | Stable |
| `defi_strategy` | `1.0.0` | Stable |
| `security_finding` | `1.0.0` | Stable |
| `wallet_record` | `1.0.0` | Stable |
| `user_profile` | `1.0.0` | Stable |
