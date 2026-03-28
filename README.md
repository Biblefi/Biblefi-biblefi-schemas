# BibleFi Schemas

Canonical [JSON Schema](https://json-schema.org/) definitions for the BibleFi protocol. Every message exchanged between agents, every record stored on-chain, and every payload surfaced by BibleFi APIs must conform to one of these schemas.

---

## Repository Layout

```
schemas/
  agent_envelope.schema.json    # Universal wrapper for all BibleFi payloads
  scripture_record.schema.json  # Canonicalised Bible passage records
  church_record.schema.json     # Church / faith-organisation profiles
  defi_strategy.schema.json     # DeFi yield & staking strategy configurations
  security_finding.schema.json  # Security audit & scan findings
VERSIONING.md                   # Versioning policy and change process
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
from jsonschema import validate, RefResolver
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

Current schema versions are all at **`1.0.0`**.

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
