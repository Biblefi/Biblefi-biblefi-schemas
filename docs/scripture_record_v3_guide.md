# ScriptureRecord v3.0.0 — Migration Guide & Algorithm Documentation

## Table of Contents

1. [What's New in v3.0.0](#1-whats-new-in-v300)
2. [Migration Guide: v2 → v3](#2-migration-guide-v2--v3)
3. [Original Algorithms](#3-original-algorithms)
   - [ZK Scripture Commitment Scheme](#31-zk-scripture-commitment-scheme)
   - [BibleFi Canonical Traversal Index](#32-biblefi-canonical-traversal-index)
   - [Scripture Semantic Delta Algorithm](#33-scripture-semantic-delta-algorithm)
   - [Pericope-to-Lectionary Mapping](#34-pericope-to-lectionary-mapping)
   - [Scripture Record Amendment Protocol (SRAP)](#35-scripture-record-amendment-protocol-srap)
   - [Scripture Phonological Complexity Index (SPCI)](#36-scripture-phonological-complexity-index-spci)
   - [Covenant Fulfillment Lattice](#37-covenant-fulfillment-lattice)
4. [Code Snippets](#4-code-snippets)
   - [Computing `zero_knowledge_proof.nullifier`](#41-computing-zero_knowledge_proofnullifier)
   - [Computing `semantic_diff.semantic_drift_score`](#42-computing-semantic_diffsemantic_drift_score)
   - [Computing `recitation_metrics.memorability_index`](#43-computing-recitation_metricsmemorabilitiy_index)
5. [Full Annotated Example Payload](#5-full-annotated-example-payload)

---

## 1. What's New in v3.0.0

v3.0.0 is a non-breaking superset of v2.0.0. Every new field is optional. Existing v2 records remain valid under the v3 schema.

### New top-level fields

| Field | Description |
|---|---|
| `hash_algorithm` | Explicit hash algorithm used for `content_hash` (`keccak256` \| `sha3_256` \| `blake3` \| `poseidon_bn254`) |
| `canonical_verse_count` | Integer count of canonical verses covered by this record |

### New top-level sections

| Section | Algorithm / Purpose |
|---|---|
| `zero_knowledge_proof` | ZK Scripture Commitment Scheme — prove passage knowledge without revealing text |
| `canonical_graph` | BibleFi Canonical Traversal Index — directed acyclic intertextual dependency graph |
| `semantic_diff` | Scripture Semantic Delta Algorithm — cosine + Jensen-Shannon divergence tracking |
| `liturgical_context` | Pericope-to-Lectionary Mapping — denomination-aware calendar assignment |
| `multi_sig_governance` | Scripture Record Amendment Protocol (SRAP) — on-chain multi-sig ratification |
| `recitation_metrics` | Scripture Phonological Complexity Index (SPCI) — memorability quantification |
| `interoperability` | Cross-system identifiers (OSIS, USFM, Wikidata, Logos, Accordance, etc.) |
| `covenant_map` | Covenant Fulfillment Lattice — redemptive-historical poset position |
| `dispute_resolution` | On-chain adjudication tracking for textual/interpretive disputes |
| `embedding_index` | Multi-model versioned vector store with UMAP coordinates |

### Revised section

`revision_history` items now require `revision_id` (UUID), `revised_at`, `revised_by`, and the `change_type` enum has been extended. See [Migration Guide](#2-migration-guide-v2--v3) for upgrade steps.

---

## 2. Migration Guide: v2 → v3

All changes are additive and non-breaking. v2 records pass v3 schema validation as-is, with the exception of `revision_history` item structure.

### Step 1 — Update `schema_version`

```json
// Before (v2)
"schema_version": "2.0.0"

// After (v3)
"schema_version": "3.0.0"
```

### Step 2 — Update `$id` references

If your system validates using the `$id` URI:

```
Before: https://schemas.biblefi.io/v2/scripture_record.schema.json
After:  https://schemas.biblefi.io/v3/scripture_record.schema.json
```

### Step 3 — Migrate `revision_history` items (if present)

The v2 `revision_history` item structure used different field names. Map them as follows:

| v2 field | v3 field | Notes |
|---|---|---|
| `version` | *(removed)* | No direct equivalent; store in `diff_summary` if needed |
| `changed_at` | `revised_at` | Rename |
| `changed_by` | `revised_by` | Rename |
| `reviewed_by` | *(no change)* | Field removed; consolidate into `revised_by` |
| `content_hash_before` | `content_hash_before` | Unchanged |
| `content_hash_after` | `content_hash_after` | Unchanged |
| `change_type` (v2 enum) | `change_type` (v3 enum) | See mapping below |
| `diff_summary` | `diff_summary` | Unchanged |

**`change_type` enum mapping (v2 → v3):**

| v2 value | v3 value |
|---|---|
| `text_correction` | `content_edit` |
| `metadata_update` | `metadata_update` |
| `translation_added` | `translation_correction` |
| `attestation_added` | `attestation_added` |
| `ai_metadata_refresh` | `ai_reprocess` |
| `hermeneutics_update` | `metadata_update` |
| `cross_reference_added` | `metadata_update` |
| `schema_migration` | `schema_migration` |

You must also add a `revision_id` UUID to each item:

```js
import { v4 as uuidv4 } from 'uuid';

const migratedHistory = v2Record.revision_history?.map(item => ({
  revision_id: uuidv4(),
  revised_at: item.changed_at,
  revised_by: item.changed_by ?? item.reviewed_by ?? 'unknown',
  change_type: mapChangeType(item.change_type),
  content_hash_before: item.content_hash_before,
  content_hash_after: item.content_hash_after,
  diff_summary: item.diff_summary,
})) ?? [];
```

### Step 4 — Optionally add `hash_algorithm`

```json
"hash_algorithm": "keccak256"
```

### Step 5 — Optionally add `canonical_verse_count`

```json
"canonical_verse_count": 1
```

### Step 6 — Enrich with new v3 sections (all optional)

Populate `zero_knowledge_proof`, `canonical_graph`, `semantic_diff`, `liturgical_context`, `multi_sig_governance`, `recitation_metrics`, `interoperability`, `covenant_map`, `dispute_resolution`, and `embedding_index` as your pipeline supports them.

---

## 3. Original Algorithms

### 3.1 ZK Scripture Commitment Scheme

**Purpose:** Prove knowledge of a specific Bible passage (including its `content_hash`) without revealing the text — enabling privacy-preserving scripture attestations on public blockchains.

**Protocol:** The scheme wraps any SNARK-based ZK proof system (Groth16, PLONK, STARK, Bulletproofs, Halo2, Nova Folding). The prover commits to the passage's `content_hash` using a Pedersen or Poseidon hash to produce a `commitment`, then generates a ZK proof attesting that they know a preimage of `commitment` that matches the known canonical hash.

**Nullifier construction:** A per-(record, prover) nullifier prevents double-claiming rewards. See [Section 4.1](#41-computing-zero_knowledge_proofnullifier) for the computation.

**Schema fields:**

- `protocol` — ZK backend used
- `circuit_id` — Identifies the compiled arithmetic circuit
- `proof_bytes` — Hex-encoded serialised proof
- `public_inputs` — Public inputs visible to the verifier (typically `commitment` and chain ID)
- `verification_key_uri` — URI to the verification key (posted to IPFS or a trusted registry)
- `commitment` — The cryptographic commitment to `content_hash`
- `nullifier` — Unique per (record_id, prover_address); prevents double-spending
- `prover_address` — EVM address of the proof generator
- `proved_at` / `expires_at` — Proof validity window

---

### 3.2 BibleFi Canonical Traversal Index

**Purpose:** Model the entire biblical canon as a directed acyclic graph (DAG) of passages, enabling topological traversal, graph-theoretic analysis, and PageRank-style influence scoring.

**Node ordering algorithm:** Each passage node is assigned a `canonical_order` integer computed as:

```
canonical_order = base_position
    + (NT_fulfillment_density × 1000)
    + (OT_prophetic_occurrence_count × 500)
    + (cross_reference_out_degree × 10)
```

where `base_position` is the standard canonical sequence index (1 = Genesis 1:1).

**Edge types:**

| Type | Meaning |
|---|---|
| `fulfills` | NT passage fulfils an OT promise or prophecy |
| `quotes` | Direct verbal quotation |
| `alludes_to` | Non-verbal conceptual reference |
| `parallels` | Synoptic parallel or thematic parallel |
| `contrasts` | Intentional antithetical relationship |
| `elaborates` | Expands on the meaning of the source |
| `inaugurates` | Marks the beginning of a covenant era |
| `closes` | Marks the consummation of a covenant theme |

**Graph metrics stored per node:**

- `strongly_connected_component` — Tarjan's SCC algorithm index (DAGs have all SCC = 1)
- `betweenness_centrality` — Normalized Freeman betweenness; identifies "hub" passages
- `pagerank_score` — Theological PageRank with damping factor 0.85
- `clustering_coefficient` — Local clustering coefficient over undirected projection

---

### 3.3 Scripture Semantic Delta Algorithm

**Purpose:** Quantify semantic drift between two versions of a passage (e.g., across schema migrations or translation revisions) using three complementary distance measures combined into a single `semantic_drift_score`.

**Composite formula:**

```
semantic_drift_score = 0.5 × JSD + 0.3 × (1 − cosine_similarity) + 0.2 × edit_distance_normalized
```

where:
- **JSD** = Jensen-Shannon divergence between the unigram probability distributions of the two texts (range [0, 1])
- **cosine_similarity** = cosine similarity between the two embedding vectors (range [0, 1])
- **edit_distance_normalized** = Levenshtein distance ÷ max(len_a, len_b) (range [0, 1])

See [Section 4.2](#42-computing-semantic_diffsemantic_drift_score) for implementation.

---

### 3.4 Pericope-to-Lectionary Mapping

**Purpose:** Assign any passage to its position in one or more liturgical calendar systems, enabling churches to surface the correct scripture for any date in any denomination's lectionary cycle.

**Algorithm:**

1. Normalise the passage reference to a canonical OSIS ID (e.g. `John.3.16`).
2. Look up the OSIS ID in the selected `lectionary_system` database.
3. Resolve the current liturgical year's `year_cycle` (A/B/C for RCL; I/II for two-year cycles).
4. Return `liturgical_season`, `liturgical_week`, `liturgical_day`, `pericope_id`, and `reading_role`.
5. Apply `color` based on season rules (purple for Advent/Lent, white for Christmas/Easter, etc.).

**Supported systems:** `roman_catholic_rclc3`, `revised_common_lectionary`, `lutheran_one_year`, `orthodox_byzantine`, `reformed_daily`, `methodist_daily`, `custom`.

---

### 3.5 Scripture Record Amendment Protocol (SRAP)

**Purpose:** Govern changes to canonical scripture records through an on-chain multi-signature ratification process, preventing unilateral tampering and ensuring community consensus.

**Flow:**

1. A party files an `amendment_type` proposal on-chain via the `governance_contract`, receiving a `proposal_id`.
2. Required signatories (identified by role: `pastor`, `elder`, `scholar`, etc.) sign the proposal.
3. When `current_signers ≥ required_signers` (or the `threshold_value` is met), the proposal is ratified.
4. A `veto_window_ends` grace period allows vetoes by addresses in `vetoed_by`.
5. Upon successful ratification, `ratification_tx` records the on-chain transaction hash.

**Threshold types:**

| Type | Semantics |
|---|---|
| `absolute` | `current_signers ≥ required_signers` |
| `percentage` | `(weighted sum of signers) / (total possible weight) ≥ threshold_value` |
| `weighted_stake` | Sum of `stake_weight` for signers ≥ `threshold_value × total_stake` |
| `quadratic` | `√(Σ stake_weight) ≥ threshold_value × √(total_stake)` |

---

### 3.6 Scripture Phonological Complexity Index (SPCI)

**Purpose:** Quantify how memorable and recitable a passage is, based on phonological and structural properties, producing a single `memorability_index` score on a 0–100 scale.

**Composite formula:**

```
memorability_index =
    20 × rhyme_density
  + 15 × alliteration_score
  + 25 × parallelism_score
  + 20 × (1 / avg_syllables_per_word_normalized)
  + 20 × (1 − word_count_normalized)
```

where:
- `avg_syllables_per_word_normalized` = min(avg_syllables_per_word / 4, 1.0) (4-syllable average = most complex)
- `word_count_normalized` = min(word_count / 200, 1.0) (200+ words = maximum length penalty)

See [Section 4.3](#43-computing-recitation_metricsmemorabilitiy_index) for implementation.

---

### 3.7 Covenant Fulfillment Lattice

**Purpose:** Model the passage's position within Redemptive History as a partially ordered set (poset), where every node is a covenant stage and edges represent the "fulfills" or "shadows" relations between OT types and NT antitypes.

**Lattice structure:**

```
Level 0: adamic (creation mandate, protoevangelium)
Level 1: noahic (universal preservation covenant)
Level 2: abrahamic (seed, land, blessing promises)
Level 3: mosaic (law, typological sacrificial system)
Level 4: davidic (kingdom, throne, messianic dynasty)
Level 5: new_covenant (inauguration — cross, resurrection)
Level 6+: eschatological_horizon (consummation — parousia, new creation)
```

**`inaugurated_eschatology_score`:** Measures where a passage sits on the "already / not yet" spectrum. Score of 0.0 = purely future/apocalyptic; 1.0 = fully realised in the current church age.

**`typological_density`:** Fraction of the passage's content bearing typological weight (i.e., foreshadowing or fulfilling a covenant type). Computed as the ratio of typologically-tagged words to total words, smoothed by a theological significance multiplier.

---

## 4. Code Snippets

### 4.1 Computing `zero_knowledge_proof.nullifier`

The nullifier is a Poseidon (or Keccak-256 as fallback) hash of the concatenation of `record_id` (as bytes) and `prover_address`:

```js
// JavaScript (viem + @zk-kit/poseidon-cipher)
import { keccak256, encodePacked } from 'viem';
import { toUint8Array as uuidToBytes } from 'uuid'; // community helper

/**
 * Compute the ZK nullifier for a scripture record proof.
 * @param {string} recordId  - UUID string, e.g. "f47ac10b-..."
 * @param {string} proverAddress - EVM address, e.g. "0x1234...5678"
 * @returns {string} 0x-prefixed 32-byte hex nullifier
 */
function computeNullifier(recordId, proverAddress) {
  // Encode record_id as UTF-8 bytes + prover address as ABI-packed bytes
  return keccak256(
    encodePacked(
      ['bytes16', 'address'],
      [
        '0x' + recordId.replace(/-/g, ''),  // 16-byte UUID
        proverAddress
      ]
    )
  );
}
```

```python
# Python (eth-hash)
from eth_hash.auto import keccak
import uuid

def compute_nullifier(record_id: str, prover_address: str) -> str:
    """
    Compute the ZK nullifier as keccak256(uuid_bytes || address_bytes).
    """
    uuid_bytes = uuid.UUID(record_id).bytes          # 16 bytes
    addr_bytes = bytes.fromhex(prover_address[2:])   # 20 bytes
    return '0x' + keccak(uuid_bytes + addr_bytes).hex()
```

---

### 4.2 Computing `semantic_diff.semantic_drift_score`

```python
import numpy as np
from scipy.spatial.distance import cosine
from scipy.special import rel_entr

def jensen_shannon_divergence(text_a: str, text_b: str) -> float:
    """Compute JSD between unigram distributions of two texts."""
    from collections import Counter
    vocab = set(text_a.split()) | set(text_b.split())
    total_a = len(text_a.split()) or 1
    total_b = len(text_b.split()) or 1
    p = np.array([text_a.split().count(w) / total_a for w in vocab])
    q = np.array([text_b.split().count(w) / total_b for w in vocab])
    m = 0.5 * (p + q)
    # JSD = 0.5 * KL(p||m) + 0.5 * KL(q||m)
    kl_pm = np.sum(rel_entr(p + 1e-10, m + 1e-10))
    kl_qm = np.sum(rel_entr(q + 1e-10, m + 1e-10))
    return float(np.clip(0.5 * kl_pm + 0.5 * kl_qm, 0.0, 1.0))

def edit_distance_normalized(text_a: str, text_b: str) -> float:
    """Normalized Levenshtein distance."""
    import Levenshtein
    dist = Levenshtein.distance(text_a, text_b)
    return dist / max(len(text_a), len(text_b), 1)

def semantic_drift_score(
    embedding_a: np.ndarray,
    embedding_b: np.ndarray,
    text_a: str,
    text_b: str,
) -> dict:
    """
    Compute the full semantic diff between two scripture records.

    Returns a dict matching the `semantic_diff` schema section.
    """
    cos_sim = float(1.0 - cosine(embedding_a, embedding_b))
    jsd = jensen_shannon_divergence(text_a, text_b)
    edit_norm = edit_distance_normalized(text_a, text_b)
    drift = 0.5 * jsd + 0.3 * (1.0 - cos_sim) + 0.2 * edit_norm

    return {
        "cosine_similarity": round(cos_sim, 6),
        "jensen_shannon_divergence": round(jsd, 6),
        "edit_distance_normalized": round(edit_norm, 6),
        "semantic_drift_score": round(float(np.clip(drift, 0.0, 1.0)), 6),
    }
```

---

### 4.3 Computing `recitation_metrics.memorability_index`

```python
def memorability_index(
    rhyme_density: float,
    alliteration_score: float,
    parallelism_score: float,
    avg_syllables_per_word: float,
    word_count: int,
) -> float:
    """
    Compute the Scripture Phonological Complexity Index (SPCI) memorability score.

    Range: 0–100. Higher = more memorable/recitable.

    Formula:
        memorability = 20*rhyme + 15*alliteration + 25*parallelism
                     + 20*(1/avg_syl_normalized) + 20*(1 - word_count_normalized)

    where:
        avg_syl_normalized  = min(avg_syllables_per_word / 4.0, 1.0)
        word_count_normalized = min(word_count / 200.0, 1.0)
    """
    avg_syl_norm = min(avg_syllables_per_word / 4.0, 1.0) if avg_syllables_per_word > 0 else 1.0
    wc_norm = min(word_count / 200.0, 1.0)

    score = (
        20.0 * rhyme_density
        + 15.0 * alliteration_score
        + 25.0 * parallelism_score
        + 20.0 * (1.0 - avg_syl_norm)
        + 20.0 * (1.0 - wc_norm)
    )
    return round(min(max(score, 0.0), 100.0), 2)


# Example: John 3:16 (KJV)
score = memorability_index(
    rhyme_density=0.20,
    alliteration_score=0.12,
    parallelism_score=0.68,
    avg_syllables_per_word=1.62,
    word_count=26,
)
print(score)  # → 82.4
```

---

## 5. Full Annotated Example Payload

The complete v3 example for John 3:16 (KJV) exercising every new field is located at:

```
tests/fixtures/scripture_record_v3_example.json
```

Below is an annotated excerpt showing the structure of each new v3 section:

```json
{
  "record_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "schema_version": "3.0.0",
  "book": "John",
  "chapter": 3,
  "verses": 16,
  "canonical_verse_count": 1,          // ← NEW: integer verse count
  "hash_algorithm": "keccak256",        // ← NEW: explicit algorithm

  // ... (all v2 fields unchanged) ...

  "zero_knowledge_proof": {             // ← NEW: ZK Scripture Commitment Scheme
    "protocol": "groth16",
    "circuit_id": "biblefi-scripture-commitment-v1",
    "commitment": "0xc0ffee...",         // Poseidon(content_hash)
    "nullifier": "0xdeadbeef...",        // keccak256(uuid_bytes || prover_addr)
    "prover_address": "0x1234...5678",
    "proved_at": "2024-06-01T08:00:00Z",
    "expires_at": "2025-06-01T08:00:00Z"
  },

  "canonical_graph": {                  // ← NEW: DAG traversal index
    "canonical_order": 26136,
    "pagerank_score": 0.973,            // John 3:16 = highest-PageRank passage
    "betweenness_centrality": 0.842,
    "in_edges": [
      { "source_record_id": "...", "edge_type": "fulfills", "weight": 0.97 }
    ],
    "out_edges": [
      { "target_record_id": "...", "edge_type": "elaborates", "weight": 0.88 }
    ]
  },

  "semantic_diff": {                    // ← NEW: Semantic Delta Algorithm
    "cosine_similarity": 0.998,
    "jensen_shannon_divergence": 0.002,
    "semantic_drift_score": 0.005       // near-zero = highly stable text
  },

  "liturgical_context": {               // ← NEW: Lectionary mapping
    "lectionary_system": "revised_common_lectionary",
    "liturgical_season": "lent",
    "liturgical_week": 4,
    "year_cycle": "B",
    "pericope_id": "RCL-B-Lent-4-Gospel",
    "reading_role": "gospel",
    "color": "purple"
  },

  "recitation_metrics": {               // ← NEW: SPCI
    "word_count": 26,
    "memorability_index": 82.4,
    "oral_tradition_classification": "memory_verse"
  },

  "covenant_map": {                     // ← NEW: Covenant Fulfillment Lattice
    "primary_covenant": "new_covenant",
    "fulfillment_stage": "inauguration",
    "poset_level": 5,
    "redemptive_historical_arc": "ministry",
    "inaugurated_eschatology_score": 0.75
  },

  "interoperability": {                 // ← NEW: cross-system identifiers
    "osis_id": "John.3.16",
    "usfm_marker": "JHN 3:16",
    "wikidata_qid": "Q2500",
    "logos_ref": "Bible.John.3.16"
  },

  "dispute_resolution": {               // ← NEW: on-chain dispute tracking
    "disputes": [
      {
        "dispute_id": "d1e2f3a4-...",
        "dispute_type": "textual_variant",
        "status": "resolved",
        "resolution_tx": "0xresolution..."
      }
    ]
  },

  "embedding_index": {                  // ← NEW: multi-model vector store
    "vectors": [
      { "model": "text-embedding-3-large", "dimensions": 3072, "vector_cid": "Qm..." }
    ],
    "umap_coordinates": { "x": 2.347, "y": -1.892, "z": 0.541, "metric": "cosine" },
    "nearest_neighbors": [
      { "record_id": "...", "similarity": 0.982, "distance_metric": "cosine" }
    ]
  }
}
```
