# BibleFi Schemas

Canonical [JSON Schema](https://json-schema.org/) definitions for the BibleFi protocol. Every message exchanged between agents, every record stored on-chain, and every payload surfaced by BibleFi APIs must conform to one of these schemas.

---

## Repository Layout

```
schemas/
  agent_envelope.schema.json          # Universal wrapper for all BibleFi payloads
  scripture_record.schema.json        # Canonicalised Bible passage records
  church_record.schema.json           # Church / faith-organisation profiles
  defi_strategy.schema.json           # DeFi yield & staking strategy configurations
  security_finding.schema.json        # Security audit & scan findings
  wallet_record.schema.json           # On-chain wallet identity records
  user_profile.schema.json            # BibleFi user profiles
  agent_task.schema.json              # Agentic pipeline task specifications
  agent_run_log.schema.json           # Hourly execution cycle run logs
  scripture_seed_batch.schema.json    # Batches of validated scriptures seeded to the dApp
  cross_language_validation.schema.json # Hebrew / Greek / Aramaic cross-reference results
  theological_validation.schema.json  # Concordance & dictionary validation results
ARCHITECTURE.md                       # Agentic pipeline architecture & scalability guide
VERSIONING.md                         # Versioning policy and change process
LICENSE
README.md
```

---

## Schemas

### `agent_envelope`
The **universal wrapper** schema. Every payload transmitted between BibleFi agents or stored on-chain must be wrapped in an `AgentEnvelope`. The envelope carries the payload type discriminator, sender identity, an optional cryptographic signature, and routing metadata.

**Key fields:** `envelope_id`, `schema_version`, `created_at`, `payload_type`, `payload`, `sender`, `signature`

---

### `scripture_record`
A canonicalised reference to a Bible passage. Includes the verbatim text, translation metadata (abbreviation, language, year), and an optional on-chain attestation linking the record to a smart contract or NFT.

**Key fields:** `record_id`, `book`, `chapter`, `verses`, `translation`, `text`, `topics`, `attestation`

---

### `church_record`
A profile for a church or faith-based organisation participating in the BibleFi ecosystem. Captures physical location, contact details, on-chain treasury identity (wallet address, ENS name, multisig), and verification status.

**Key fields:** `record_id`, `name`, `denomination`, `location`, `contact`, `on_chain_identity`, `verified`

---

### `defi_strategy`
Describes a DeFi yield or staking strategy managed by the BibleFi protocol on behalf of a church or user. Includes the target protocol and asset, allocation rules, risk tier, expected APY range, optional automatic tithe distribution, and rebalancing policy.

**Key fields:** `strategy_id`, `protocol`, `asset`, `allocation`, `risk_tier`, `expected_apy`, `tithe_allocation`, `rebalance_policy`

---

### `security_finding`
Tracks a security vulnerability or audit finding from discovery through remediation. Supports findings from manual audits, automated scans, and bug bounties. Includes CVSS scoring, CWE classification, proof-of-concept, and resolution details.

**Key fields:** `finding_id`, `title`, `severity`, `status`, `target`, `cwe_ids`, `cvss`, `recommendation`, `resolution`

---

### `wallet_record`
An on-chain wallet identity for a user or church participant in the BibleFi ecosystem. Captures the EVM address, supported chain IDs, wallet type (EOA, multisig, smart wallet, or hardware), optional balance snapshot, and treasury status.

**Key fields:** `wallet_id`, `schema_version`, `address`, `chain_ids`, `wallet_type`, `ens_name`, `owner_id`, `church_id`, `balance_snapshot`, `is_treasury`

---

### `user_profile`
A BibleFi user profile representing an individual participant, church administrator, auditor, developer, or observer. Links a user to their associated church, wallet addresses, preferred Bible translation, and notification preferences.

**Key fields:** `user_id`, `schema_version`, `display_name`, `role`, `email`, `church_id`, `wallet_addresses`, `preferred_translation`, `notification_preferences`

---

## Agentic Pipeline Schemas

These schemas support the BibleFi hourly scripture-seeding pipeline. See [ARCHITECTURE.md](./ARCHITECTURE.md) for a full description of the multi-agent framework.

### `agent_task`
Specification for a single unit of work assigned to a BibleFi agent or subagent. Covers all five agent types in the pipeline: `master`, `scripture_search`, `language_validator`, `theological_validator`, and `dapp_seeder`. Each task references its parent run via `run_id` and its sandbox environment via `sandbox_id`.

**Key fields:** `task_id`, `schema_version`, `agent_type`, `agent_id`, `parent_task_id`, `run_id`, `sandbox_id`, `status`, `input`, `output`, `error`

---

### `agent_run_log`
Records the outcome of a single hourly execution cycle. The master agent creates one `AgentRunLog` per run, updating it as each pipeline stage completes. Includes aggregate metrics such as `verses_scanned`, `scriptures_found`, `scriptures_seeded`, and stage-level status.

**Key fields:** `run_id`, `schema_version`, `triggered_at`, `trigger_type`, `status`, `pipeline_stages`, `metrics`, `seed_batch_id`

---

### `scripture_seed_batch`
Represents the batch of financially-themed, fully-validated scripture records pushed to the BibleFi dApp by the `dapp_seeder` agent in a single cycle. Includes a reference to the upstream `AgentRunLog`, the list of `ScriptureRecord` UUIDs, and dApp endpoint response details.

**Key fields:** `batch_id`, `schema_version`, `run_id`, `status`, `scripture_record_ids`, `financial_themes`, `translation`, `target_dapp`, `records_seeded`

---

### `cross_language_validation`
Result of cross-referencing a KJV passage with its original Hebrew, Greek, and/or Aramaic source texts. For each language, records the source text, transliteration, Strong's-referenced key terms, alignment score, and whether the financial theme is confirmed in the original language.

**Key fields:** `validation_id`, `schema_version`, `scripture_record_id`, `english_reference`, `language_results`, `overall_status`, `validated_by_agent`, `validated_at`

---

### `theological_validation`
Result of consulting Biblical concordances (Strong's, Nave's, Young's) and dictionaries (Vine's, BDB, TDNT, Baker's) to confirm the theological accuracy and financial thematic alignment of a scripture. Assigns each passage a `dapp_category` for display in the BibleFi interface.

**Key fields:** `validation_id`, `schema_version`, `scripture_record_id`, `cross_language_validation_id`, `concordance_references`, `dictionary_definitions`, `thematic_alignment`, `overall_status`

---

## Usage

### Validation Example (Node.js / Ajv)

```js
import Ajv from "ajv";
import addFormats from "ajv-formats";
import envelope from "./schemas/agent_envelope.schema.json" assert { type: "json" };
import scriptureRecord from "./schemas/scripture_record.schema.json" assert { type: "json" };

const ajv = new Ajv({ strict: true });
addFormats(ajv);

ajv.addSchema(scriptureRecord);
const validate = ajv.compile(envelope);

const payload = {
  envelope_id: "550e8400-e29b-41d4-a716-446655440000",
  schema_version: "1.0.0",
  created_at: "2024-01-01T00:00:00Z",
  payload_type: "scripture_record",
  payload: {
    record_id: "660e8400-e29b-41d4-a716-446655440001",
    schema_version: "1.0.0",
    book: "John",
    chapter: 3,
    verses: 16,
    translation: { abbreviation: "KJV", full_name: "King James Version", language: "en" },
    text: "For God so loved the world, that he gave his only begotten Son..."
  }
};

const valid = validate(payload);
if (!valid) console.error(validate.errors);
```

### Validation Example (Python / jsonschema)

```python
import json
from jsonschema import validate
from pathlib import Path

schema_dir = Path("schemas")
envelope = json.loads((schema_dir / "agent_envelope.schema.json").read_text())

payload = {
    "envelope_id": "550e8400-e29b-41d4-a716-446655440000",
    "schema_version": "1.0.0",
    "created_at": "2024-01-01T00:00:00Z",
    "payload_type": "scripture_record",
    "payload": {}
}

validate(instance=payload, schema=envelope)
```

---

## Versioning

See [VERSIONING.md](./VERSIONING.md) for the full versioning policy. In summary:

| Change type | Version bump |
|-------------|-------------|
| Bug-fix / description update | PATCH (`1.0.x`) |
| New optional field | MINOR (`1.x.0`) |
| Breaking change | MAJOR (`x.0.0`) |

Current schema versions are all at **`1.0.0`** for unchanged schemas; `agent_envelope` and `scripture_record` have been bumped to **`1.1.0`** with the addition of agentic pipeline support. The five new agentic pipeline schemas (`agent_task`, `agent_run_log`, `scripture_seed_batch`, `cross_language_validation`, `theological_validation`) are at **`1.0.0`**. See [VERSIONING.md](./VERSIONING.md) for the full version table.

---

## Contributing

1. Fork the repository and create a feature branch.
2. Edit the relevant `.schema.json` file(s) under `schemas/`.
3. Bump the `schema_version` field in the modified schemas according to [VERSIONING.md](./VERSIONING.md).
4. Open a pull request with a clear description of the change and its rationale.
5. Ensure the PR description references any related issues or BIPs (BibleFi Improvement Proposals).

---

## License

[MIT](./LICENSE)
