# BibleFi Agentic Scripture Pipeline — Architecture

This document describes the agentic framework responsible for discovering, validating, and hourly seeding financially-themed Bible scriptures into the BibleFi dApp.

---

## Table of Contents

1. [Overview](#overview)
2. [Agent Hierarchy](#agent-hierarchy)
3. [Execution Flow](#execution-flow)
4. [Schemas](#schemas)
5. [Security and Sandboxing](#security-and-sandboxing)
6. [Multi-Language Scalability](#multi-language-scalability)
7. [Schema Version History](#schema-version-history)

---

## Overview

The BibleFi Agentic Scripture Pipeline is a multi-agent system that runs on an hourly schedule. It:

1. Traverses the King James Version (KJV) Bible from Genesis to Revelation.
2. Identifies passages with financial themes (debt, money, wealth, stewardship, tithing, etc.).
3. Cross-references each identified passage with the corresponding original Hebrew, Greek, and/or Aramaic texts.
4. Validates every passage against Biblical concordances and dictionaries for theological accuracy.
5. Seeds the fully-validated scripture records into the BibleFi dApp for public display.

Every agent in the pipeline operates inside an **isolated sandbox** (separate CI job / container), communicates exclusively through **`AgentEnvelope`-wrapped messages**, and emits structured records that conform to the schemas in this repository.

---

## Agent Hierarchy

```
Master Agent
├── scripture_search subagent(s)
│   └── Traverses KJV, emits ScriptureRecord documents with financial_keywords
├── language_validator subagent(s)
│   └── Validates each ScriptureRecord against Hebrew / Greek / Aramaic source texts
│       └── Emits CrossLanguageValidation records
├── theological_validator subagent(s)
│   └── Consults concordances and dictionaries for thematic accuracy
│       └── Emits TheologicalValidation records
└── dapp_seeder subagent
    └── Bundles validated records into a ScriptureSeedBatch and pushes to BibleFi dApp
```

### Agent Types (defined in `agent_task.schema.json`)

| `agent_type` | Responsibility |
|---|---|
| `master` | Orchestrates the hourly pipeline; creates `AgentRunLog`; spawns and monitors all subagents |
| `scripture_search` | Scans the KJV for passages matching financial keywords; emits `ScriptureRecord` documents |
| `language_validator` | Compares KJV passages against Original Hebrew, Greek, and Aramaic; emits `CrossLanguageValidation` records |
| `theological_validator` | Checks concordances (Strong's, Nave's, Young's) and dictionaries (Vine's, BDB, TDNT) for thematic alignment; emits `TheologicalValidation` records |
| `dapp_seeder` | Formats and pushes validated records to the BibleFi dApp API; emits `ScriptureSeedBatch` records |

---

## Execution Flow

Each hourly execution cycle follows a strict, sequential pipeline:

```
[Cron: every hour at :00]
          │
          ▼
  ┌───────────────────┐
  │   Master Agent    │  Creates AgentRunLog (run_id, triggered_at, trigger_type)
  └────────┬──────────┘
           │  spawns
           ▼
  ┌─────────────────────────┐
  │  Scripture Search Agent │  Scans KJV Genesis → Revelation
  │  (Sandbox A)            │  Financial keywords: debt, money, wealth, tithe,
  │                         │  mammon, usury, treasure, silver, gold, stewardship…
  └────────────┬────────────┘
               │  ScriptureRecord[]
               ▼
  ┌────────────────────────────┐
  │  Language Validator Agent  │  Hebrew: Westminster Leningrad Codex
  │  (Sandbox B)               │  Greek:  Nestle-Aland 28th Edition
  │                            │  Aramaic: Peshitta
  │                            │  Emits CrossLanguageValidation per record
  └────────────┬───────────────┘
               │  CrossLanguageValidation[]
               ▼
  ┌──────────────────────────────┐
  │  Theological Validator Agent │  Concordances: Strong's, Nave's, Young's
  │  (Sandbox C)                 │  Dictionaries: Vine's, BDB, TDNT, Baker's
  │                              │  Emits TheologicalValidation per record
  └─────────────┬────────────────┘
                │  TheologicalValidation[]
                ▼
  ┌─────────────────────────┐
  │  dApp Seeder Agent      │  Bundles into ScriptureSeedBatch
  │  (Sandbox D)            │  Pushes to BibleFi dApp API
  │                         │  Updates AgentRunLog with metrics
  └─────────────────────────┘
```

### Chronological Scheduling

The pipeline is triggered by a GitHub Actions `schedule` event (`cron: "0 * * * *"`) — every hour at minute zero. The `concurrency` block prevents overlapping runs:

```yaml
concurrency:
  group: biblefi-agentic-pipeline
  cancel-in-progress: false
```

If a run takes longer than an hour, the next scheduled run queues and waits rather than cancelling the active run.

---

## Schemas

The agentic pipeline introduces five new schemas and extends two existing schemas:

### New Schemas

| Schema file | Title | Purpose |
|---|---|---|
| `agent_task.schema.json` | `AgentTask` | Spec for a single unit of work assigned to any agent or subagent |
| `agent_run_log.schema.json` | `AgentRunLog` | Log record for one hourly execution cycle; tracks all stage statuses and metrics |
| `scripture_seed_batch.schema.json` | `ScriptureSeedBatch` | Batch of validated scriptures pushed to the dApp each cycle |
| `cross_language_validation.schema.json` | `CrossLanguageValidation` | Result of cross-referencing a KJV passage with Hebrew/Greek/Aramaic sources |
| `theological_validation.schema.json` | `TheologicalValidation` | Result of consulting concordances and dictionaries for thematic accuracy |

### Updated Schemas

| Schema file | Change | New Version |
|---|---|---|
| `agent_envelope.schema.json` | Added 5 new `payload_type` enum values for the agentic pipeline | `1.1.0` |
| `scripture_record.schema.json` | Added optional `financial_keywords`, `original_language_refs`, `cross_language_validation_id`, `theological_validation_id` fields | `1.1.0` |

### Schema Relationships

```
AgentEnvelope (wrapper)
  └── payload_type: "agent_run_log"
        └── AgentRunLog
              └── pipeline_stages[].task_ids → AgentTask[]
                    ├── agent_type: "scripture_search"  → ScriptureRecord (financial_keywords populated)
                    ├── agent_type: "language_validator" → CrossLanguageValidation
                    ├── agent_type: "theological_validator" → TheologicalValidation
                    └── agent_type: "dapp_seeder" → ScriptureSeedBatch
                                                         └── scripture_record_ids[] → ScriptureRecord[]
```

### Key Design Decisions

**`financial_keywords` on `ScriptureRecord`**
The scripture_search agent populates this field with the exact financial keyword(s) that caused the passage to be selected. This provides an audit trail: a consumer can always see *why* a record was flagged as financially relevant.

**`original_language_refs` on `ScriptureRecord`**
Stores the raw original-language text inline on the scripture record so the dApp can optionally display it alongside the English text. The `CrossLanguageValidation` record contains the full analysis; `original_language_refs` is a lightweight summary.

**`strongs_number` on `CrossLanguageValidation.language_results[].key_terms[]`**
Strong's numbers (e.g. `H3701` for Hebrew *keseph* = silver/money) are the universal reference system for original-language Biblical terms. Including them enables future integrations with Strong's APIs and lexicon databases.

**`dapp_category` on `TheologicalValidation.thematic_alignment`**
The theological_validator assigns each scripture to a dApp display category (e.g. `"Debt & Borrowing"`, `"Giving & Tithing"`, `"Stewardship"`). This lets the dApp render scriptures in organised thematic groups without additional classification logic.

---

## Security and Sandboxing

### Isolation Model

Each agent type runs in a **separate GitHub Actions job** (the `agent-scheduler.yml` workflow defines one job per stage). GitHub Actions guarantees that:

- Each job runs on a fresh, ephemeral Ubuntu runner.
- Jobs do not share filesystem state, environment variables, or network namespaces unless explicitly passed through `outputs`.
- The only data that flows between jobs is via typed `outputs` (e.g. `run_id`, `task_id` — both UUIDs), preventing injection of arbitrary payloads between sandboxes.

### Communication Protocol

All inter-agent messages must be wrapped in an `AgentEnvelope`. The envelope carries:
- A cryptographic `signature` (optional but recommended for production) to prove provenance.
- A `sender` block identifying the originating agent.
- A `correlation_id` linking messages back to the parent `run_id`.

Agents must **reject** any envelope whose `signature` cannot be verified or whose `payload_type` is not in the approved enum.

### Permissions

Each job in `agent-scheduler.yml` requests only `contents: read` — the minimum required to read schema files. No job is granted write access to the repository or any external service in the schema-validation phase. Production deployments should use GitHub OIDC short-lived tokens for any external API calls.

### Secrets Management

Secrets required by the `dapp_seeder` agent (e.g. `BIBLEFI_DAPP_API_KEY`) must be stored as [GitHub Actions encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) and accessed only within the `dapp-seeder` job scope. They must never be printed to logs or passed as job outputs.

---

## Multi-Language Scalability

The framework is designed for gradual expansion from English (KJV) to translations in every major world language:

### Phase 1 — English (current)
- Primary translation: KJV
- Original-language validation: Hebrew, Greek, Aramaic
- dApp language: `en`

### Phase 2 — Major European Languages
Extend `scripture_search` agent input to accept additional translation codes:

```json
{
  "input": {
    "translation": "RVR1960",
    "target_languages": ["hebrew", "greek", "aramaic"]
  }
}
```

The `translation` object on `ScriptureRecord` already supports BCP-47 language tags (`"es"`, `"pt-BR"`, `"fr"`, etc.) and a `language` field, so no schema changes are required for this phase.

### Phase 3 — Global Availability (all continents)

| Region | Target languages | Notes |
|---|---|---|
| Africa | Swahili (`sw`), Yoruba (`yo`), Amharic (`am`), Zulu (`zu`), Hausa (`ha`) | Many users prefer local-language Bibles |
| Asia & Pacific | Mandarin (`zh`), Hindi (`hi`), Tamil (`ta`), Indonesian (`id`), Japanese (`ja`), Korean (`ko`) | High-growth mobile-first markets |
| Americas | Spanish (`es`), Portuguese (`pt-BR`), French Creole (`ht`) | Largest diaspora communities |
| Europe | German (`de`), French (`fr`), Russian (`ru`), Ukrainian (`uk`) | Strong Christian populations |
| Middle East | Arabic (`ar`) | Critical for context with Biblical origins |

**Schema extensibility**: The `translation.language` field on `ScriptureRecord` already accepts any BCP-47 tag matching `^[a-z]{2,3}(-[A-Z]{2,3})?$`. Adding a new language requires only:
1. A new `scripture_search` subagent task with the desired translation code.
2. An additional `AgentTask` pointing to the appropriate multilingual Bible corpus API.
3. No schema changes (MINOR or MAJOR version bump) are needed.

### Adding New Agent Types

To add a new subagent specialisation (e.g. a `transliteration_agent` for phonetic rendering):
1. Add the new value to the `agent_type` enum in `agent_task.schema.json` (MINOR version bump).
2. Add a new job to `agent-scheduler.yml` in the appropriate pipeline stage.
3. Create a corresponding new schema if the agent produces a new record type.

---

## Schema Version History

| Schema | Version | Change |
|---|---|---|
| `agent_envelope` | `1.1.0` | Added `agent_task`, `agent_run_log`, `scripture_seed_batch`, `cross_language_validation`, `theological_validation` to `payload_type` enum |
| `scripture_record` | `1.1.0` | Added optional `financial_keywords`, `original_language_refs`, `cross_language_validation_id`, `theological_validation_id` fields |
| `agent_task` | `1.0.0` | New schema |
| `agent_run_log` | `1.0.0` | New schema |
| `scripture_seed_batch` | `1.0.0` | New schema |
| `cross_language_validation` | `1.0.0` | New schema |
| `theological_validation` | `1.0.0` | New schema |
| `church_record` | `1.0.0` | Unchanged |
| `defi_strategy` | `1.0.0` | Unchanged |
| `security_finding` | `1.0.0` | Unchanged |
| `wallet_record` | `1.0.0` | Unchanged |
| `user_profile` | `1.0.0` | Unchanged |
