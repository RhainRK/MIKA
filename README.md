# MIKA

**Local evidence intelligence and AI assurance for provenance-aware analysis, hybrid retrieval and auditable investigations.**

MIKA is a local-first evidence analysis system built for working with operational records, structured datasets and investigation material while keeping retrieval, permissions, provenance and audit controls outside the language model.

It combines deterministic information retrieval with optional local AI-assisted analysis. Evidence is ingested with source provenance, indexed for lexical and semantic search, checked for suspicious prompt-injection content and made available through a CLI or browser-based analyst console.

The language model is optional. Core functions such as ingestion, retrieval, authorization, case management, provenance tracking, evidence filtering and audit verification run independently of model generation.

Current release: **v1.4.0**

---

## Overview

MIKA is designed around a simple principle:

> The language model should not be the trusted component of an analytical system.

Instead of giving an LLM unrestricted access to files, tools and external services, MIKA separates deterministic system controls from generated analysis.

MIKA handles:

* evidence ingestion and provenance;
* hybrid lexical and semantic retrieval;
* source and evidence filtering;
* case management;
* citation tracking;
* entity resolution;
* relationship analysis;
* structured-data profiling;
* constrained tool execution;
* prompt-injection quarantine;
* tamper-evident audit logging;
* retrieval evaluation and regression testing;
* optional local-LLM synthesis.

This makes MIKA useful as both an evidence-analysis application and an experimental platform for building safer retrieval-augmented AI systems.

---

## Design

MIKA is built around several independent layers.

```text
                         +---------------------+
                         |     Source Data     |
                         +----------+----------+
                                    |
                                    v
                         +---------------------+
                         |      Ingestion      |
                         | provenance + hashes |
                         +----------+----------+
                                    |
                                    v
                       +-------------------------+
                       | Injection / Trust Check |
                       +-----------+-------------+
                                   |
                 +-----------------+-----------------+
                 |                                   |
                 v                                   v
         Trusted Evidence                    Quarantined Evidence
                 |                          excluded by default
                 |
                 v
       +----------------------+
       | SQLite Chunk Store   |
       | FTS5 + Provenance    |
       +----------+-----------+
                  |
        +---------+---------+
        |                   |
        v                   v
+---------------+   +----------------+
| Lexical Search|   | Semantic Search|
| FTS5 / BM25   |   | embeddings     |
+-------+-------+   +--------+-------+
        |                    |
        +---------+----------+
                  |
                  v
       +----------------------+
       | Reciprocal Rank      |
       | Fusion + ID Boosting |
       +----------+-----------+
                  |
                  v
       +----------------------+
       | Evidence Selection   |
       | sources/cases/chunks |
       +----------+-----------+
                  |
          +-------+-------+
          |               |
          v               v
   Case Analysis      Local LLM
   Findings           optional
   Evidence           grounded output
   Reports                  |
          |                 v
          |         Citation Validation
          |                 |
          +--------+--------+
                   |
                   v
             Analyst Output

Tool request
     |
     v
Policy + typed schema
     |
     v
Authorized execution
     |
     v
Audit chain
```

The detailed architecture is documented in [`docs/architecture.md`](docs/architecture.md).

---

## Core Features

### Provenance-aware ingestion

MIKA records where evidence came from rather than treating imported text as an anonymous knowledge base.

Sources and chunks are associated with:

* source identifiers;
* SHA-256 fingerprints;
* chunk identifiers;
* import metadata;
* quarantine state;
* source-level inventory information.

Unchanged evidence can be imported again without replacing existing chunk IDs.

This is important because cases, findings and citations may already reference those chunks. Stable identifiers prevent routine re-imports from silently breaking historical evidence references.

---

### Source register

v1.4.0 adds a dedicated source register for inspecting the material currently stored in MIKA.

The register exposes information including:

* source identity;
* source fingerprint;
* number of chunks;
* import count;
* quarantine state;
* source metadata.

Absolute workspace paths are not exposed through the browser interface.

CLI examples:

```bash
mika sources
mika source 1
```

---

### Hybrid retrieval

MIKA combines lexical and semantic retrieval.

Lexical search uses:

* SQLite FTS5;
* BM25 ranking;
* exact structured-identifier handling.

Semantic retrieval uses local embeddings and vector similarity.

The two result sets are combined using **Reciprocal Rank Fusion (RRF)**.

This allows MIKA to handle both:

* exact identifiers such as `CASE-1042`, account numbers or technical terms;
* natural-language queries where wording differs from the source material.

Exact structured identifiers receive additional ranking treatment so a semantically similar document does not displace an exact record match.

---

### Selected-evidence search

Search does not always need to run across the entire evidence store.

MIKA v1.4.0 can restrict retrieval to selected:

* sources;
* chunk IDs;
* case evidence.

Examples:

```bash
mika search "reconciliation anomaly" --source 1
```

```bash
mika search "payment discrepancy" --case case_xxxxxxxxxxxx
```

This is particularly useful during investigations where an analyst wants the retrieval system to reason only over an approved evidence set.

---

### Bounded-memory semantic retrieval

Earlier MIKA builds could assemble larger intermediate embedding structures during retrieval.

v1.4.0 changes semantic search to operate in bounded batches and maintain only the current best candidates required for top-k selection.

The new path also detects and repairs invalid embedding records when possible, including:

* corrupt vectors;
* stale dimensions;
* non-finite values;
* missing embeddings.

In a synthetic 12,000-chunk comparison used during the v1.4.0 release review, measured peak Python allocation fell from approximately **49.3 MiB to 1.51 MiB**.

That is roughly a **96.9% reduction** in that specific test environment.

The same comparison measured query time at approximately:

* v1.3.2 path: `0.277 s`
* v1.4.0 path: `0.209 s`

These numbers are regression measurements from one build environment, not general production-performance guarantees.

---

## Case Management

MIKA includes an evidence-backed case workflow.

Cases can contain:

* an investigation title;
* an objective;
* registered evidence;
* findings;
* confidence values;
* alternatives;
* limitations.

Create a case:

```bash
mika case-create "Reconciliation Investigation" --objective "Identify the cause of the reported ledger mismatch"
```

List cases:

```bash
mika cases
```

Attach evidence:

```bash
mika case-attach case_xxxxxxxxxxxx 12 13 18
```

Record a finding:

```bash
mika case-finding case_xxxxxxxxxxxx \
  "The discrepancy is associated with the settlement batch." \
  --confidence 0.86 \
  --evidence 12 18
```

Export a report:

```bash
mika case-report case_xxxxxxxxxxxx
```

The analyst dashboard provides the same workflow through a browser interface.

---

## Evidence-backed findings

Findings can reference specific evidence chunks.

In v1.4.0, evidence cited by a finding is automatically registered with the case. This keeps the case's evidence register synchronized with the analytical record.

A finding can also retain:

* alternative explanations;
* limitations;
* confidence estimates.

The goal is not to turn a model response into a fact automatically. The case structure keeps the conclusion, supporting evidence and uncertainty separately reviewable.

---

## Citation Validation

When the optional local model generates an answer, MIKA checks the citation labels returned by the model against the evidence supplied to it.

The response includes citation-validation metadata and can identify citation labels that were invented by the model rather than supplied by retrieval.

This catches a common failure mode in retrieval-augmented generation where an answer appears sourced but references evidence that was never actually retrieved.

Citation validation does **not** prove that every generated claim is correct. It verifies the relationship between emitted citation labels and the evidence provided to the model.

Human review is still required.

---

## Entity Resolution

MIKA includes deterministic entity-resolution tools for matching inconsistent names in structured data.

The resolver uses techniques including:

* Unicode normalization;
* alias handling;
* token blocking;
* weighted fuzzy similarity;
* explainable match scoring.

Example:

```bash
mika entity-resolve samples/entities.csv "Northstar Comp"
```

The output is intended to help an analyst investigate possible matches rather than automatically declare identity.

---

## Relationship Analysis

MIKA can analyse relationship data represented as graph edges.

The graph layer uses adjacency lists and bounded breadth-first search to trace relationships between entities.

Example:

```bash
mika graph-path samples/relationships.csv "Maya Chen" "Helios Research Group"
```

This can be used to explore:

* organizational relationships;
* transaction networks;
* account ownership;
* operational dependencies;
* other structured link data.

The implementation is deliberately bounded so a malformed or unexpectedly large graph cannot trigger unlimited traversal.

---

## Data Quality

CSV datasets can be profiled before analysis.

Example:

```bash
mika profile samples/operations_incidents.csv
```

The profiler provides a deterministic first look at structured evidence and can help identify malformed, incomplete or inconsistent records before they are fed into further analysis.

---

## Tool Security

Tools are not executed simply because a language model asks for them.

MIKA places a deterministic authorization layer between the model and execution.

A tool must be:

1. registered;
2. allowed by the active policy;
3. supplied with arguments that pass its typed schema.

Unknown tools are rejected.

Arguments are validated with Pydantic before the handler executes.

This creates a default-deny model:

```text
LLM proposes action
        |
        v
Is tool registered?
        |
       no -> reject
        |
       yes
        |
        v
Is tool permitted?
        |
       no -> reject
        |
       yes
        |
        v
Validate arguments
        |
     invalid -> reject
        |
       valid
        |
        v
Execute
```

The model itself cannot grant additional permissions.

---

## Workspace Confinement

File-backed tools are restricted to the configured workspace.

Path traversal attempts outside that workspace are rejected.

MIKA also includes a bounded read-only SQLite tool. It accepts controlled `SELECT` / `WITH` queries while rejecting operations such as:

* writes;
* schema modification;
* database attachment;
* unsafe pragmas.

This is intended to make local analysis useful without turning the model into an unrestricted database or filesystem interface.

---

## Prompt-Injection Handling

Imported evidence is treated as **untrusted data**, not as part of MIKA's system instructions.

Evidence can be scanned for suspicious prompt-injection content and quarantined.

Quarantined chunks are excluded from normal retrieval unless deliberately inspected.

This is defense in depth rather than the main security boundary.

A prompt-injection classifier can miss malicious content. MIKA therefore keeps authorization and tool permissions deterministic even if suspicious text reaches the language model.

---

## Audit Logging

MIKA records security-relevant and analytical events in a JSONL audit log.

Audit entries contain:

* event data;
* the previous record hash;
* the current record's SHA-256 digest.

This creates a chained log in which modifying or reordering an existing record breaks subsequent verification.

Check the chain with:

```bash
mika audit-verify
```

The audit system is **tamper-evident**, not cryptographically immutable. An attacker able to replace the complete log and recompute every hash could create a new internally consistent chain.

Production systems would normally add external integrity anchoring or centralized append-only logging.

---

## Database Backups

v1.4.0 introduces an integrated database backup command.

```bash
mika backup
```

By default, backups are written under:

```text
data/backups/
```

MIKA verifies the SQLite snapshot before replacing the backup destination.

The command can also be invoked as:

```bash
mika database-backup
```

---

## Schema Migrations

Database schema versions are now tracked.

When an older supported MIKA database is opened by v1.4.0, the required migration runs automatically.

The v1.4 migration adds source-register tracking while preserving existing:

* evidence;
* cases;
* findings.

Back up the database before upgrading between releases.

---

## Analyst Console

MIKA includes a FastAPI backend and TypeScript analyst interface.

The v1.4.0 console supports:

* hybrid evidence search;
* source browsing;
* source-filtered retrieval;
* case-evidence-filtered retrieval;
* case review;
* finding management;
* Markdown case-report downloads;
* evaluation information.

Start it with:

```bash
mika serve --host 127.0.0.1 --port 8000
```

Then open:

```text
http://127.0.0.1:8000
```

MIKA generates a new browser password when the server starts.

Username:

```text
mika
```

The password is printed in the terminal.

Stopping the process invalidates that session password.

---

## Local-only Web Security Model

The analyst console is designed as a **single-user local application**.

All HTTP routes require authentication.

The service also implements controls including:

* loopback-only binding;
* peer-address checks;
* Host validation;
* Origin validation;
* Fetch Metadata validation;
* restricted HTTP methods;
* request-size limits;
* rate/resource budgets;
* disabled public API documentation;
* disabled proxy-header trust;
* disabled HTTP access logging;
* Content Security Policy;
* frame denial;
* `no-store` caching policy;
* referrer restrictions;
* browser permissions policy;
* same-origin resource policy.

Do **not** expose the MIKA service through:

* router port forwarding;
* public interfaces;
* reverse proxies;
* public tunnels;
* internet-facing hosts.

It is not a multi-user web authentication platform.

---

## Optional Local LLM

MIKA does not require an LLM for retrieval, cases, evaluation or the analyst interface.

To enable generated analysis:

```bash
pip install -e ".[llm]"
```

MIKA currently supports a local Ollama workflow.

For example:

```bash
ollama pull qwen3.5:4b
```

Then:

```bash
mika ask "What evidence explains the reconciliation anomaly?"
```

MIKA restricts configured plaintext model endpoints to literal loopback addresses such as:

```text
http://127.0.0.1:11434
```

The MIKA client also disables inherited HTTP proxy configuration for model requests.

MIKA cannot control the behaviour, telemetry or network configuration of a separately installed model server. Review that software independently.

---

## Embeddings

The default embedding implementation is deterministic and offline.

No trained embedding model is required for the basic installation.

For local semantic models:

```bash
pip install -e ".[semantic]"
```

Set `MIKA_EMBEDDING_MODEL` to a trusted **local model path**.

Semantic embeddings are optional. FTS5/BM25 retrieval remains available without them.

---

## Public Data Connector

MIKA includes an allow-listed connector for the official UK FCDO sanctions dataset.

```bash
mika fetch-uk-sanctions
```

The connector is intentionally narrow. Host and endpoint validation prevent it from becoming a general-purpose HTTP client.

MIKA does not make sanctions, compliance or legal decisions automatically. Results should be reviewed by a human analyst.

---

## Installation

### Requirements

* Python **3.11+**
* Node.js/npm if rebuilding the analyst console
* Ollama only if using local generated analysis

Clone or extract the project and enter the repository:

```bash
cd MIKA_v1.4.0
```

Create a virtual environment:

```bash
python -m venv .venv
```

Linux/macOS:

```bash
source .venv/bin/activate
```

Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install MIKA:

```bash
pip install --upgrade pip setuptools
pip install -e ".[server]"
```

Check the installation:

```bash
mika --version
mika doctor
```

Expected version:

```text
MIKA 1.4.0
```

A complete Windows installation guide is available in [`WINDOWS_SETUP.md`](WINDOWS_SETUP.md).

---

## Build the Dashboard

Install the locked frontend dependencies:

```bash
cd dashboard
npm ci --ignore-scripts
npm run typecheck
npm run build
cd ..
```

Start MIKA:

```bash
mika serve --host 127.0.0.1 --port 8000
```

---

## Basic Workflow

Ingest evidence:

```bash
mika ingest samples/benchmark_evidence.txt
```

```bash
mika ingest samples/operations_incidents.csv
```

Inspect registered sources:

```bash
mika sources
```

Search:

```bash
mika search "reconciliation anomaly"
```

Inspect a source:

```bash
mika source 1
```

Create and work with a case:

```bash
mika case-create "Operations Investigation"
mika cases
```

Verify the audit log:

```bash
mika audit-verify
```

Back up the database:

```bash
mika backup
```

Run the evaluation suite:

```bash
mika evaluate --repeats 20
```

---

## Evaluation

MIKA includes deterministic retrieval and security regression tests.

Retrieval metrics include:

* Precision@k;
* Recall@k;
* Mean Reciprocal Rank;
* nDCG;
* bootstrap confidence intervals;
* retrieval latency percentiles.

The repository includes a synthetic 120-case operational retrieval benchmark.

Current v1.4.0 baseline:

| Check                       | Result      |
| --------------------------- | ----------- |
| Python tests                | 121 passing |
| Python branch coverage      | 88%         |
| Operations benchmark        | 120 cases   |
| Recall@5                    | 1.000       |
| MRR                         | 1.000       |
| nDCG@5                      | 1.000       |
| Default-deny authorization  | Pass        |
| Unknown-tool rejection      | Pass        |
| Prompt-injection quarantine | Pass        |
| Audit-chain verification    | Pass        |
| TypeScript typecheck        | Pass        |
| TypeScript build            | Pass        |
| Release publication check   | Pass        |

The bundled benchmark is synthetic and exists to detect engineering regressions. It should not be interpreted as proof of real-world accuracy.

Evaluation output and fingerprints are stored in:

```text
reports/evaluation-results.json
```

---

## Development

Install development dependencies:

```bash
pip install -e ".[dev,server]"
```

Run the Python test suite:

```bash
pytest -q
```

Run coverage:

```bash
pytest --cov=mika --cov-branch --cov-fail-under=82 -q
```

Run Ruff:

```bash
ruff check .
ruff format --check .
```

Run mypy:

```bash
mypy mika
```

Run Bandit:

```bash
bandit -r mika
```

Run the release checker:

```bash
python scripts/release_check.py
```

Frontend:

```bash
npm run --prefix dashboard typecheck
npm run --prefix dashboard build
```

Dependency audits:

```bash
pip-audit
```

```bash
npm audit --prefix dashboard
```

CI runs the relevant formatting, typing, security, test and build gates automatically.

Dependabot is configured for Python, npm and GitHub Actions dependencies.

---

## What's New in v1.4.0

v1.4.0 expands MIKA from a retrieval-focused assurance system into a more complete evidence workspace.

### Provenance and sources

* Added a dedicated source register.
* Added source metadata and chunk inventory inspection.
* Added import-count tracking.
* Repeat ingestion of unchanged evidence now preserves chunk IDs.
* Existing case and finding references remain valid across unchanged re-imports.

### Retrieval

* Added source-filtered search.
* Added chunk-filtered search.
* Added case-evidence-filtered search.
* Reworked semantic retrieval around bounded batches.
* Added bounded top-k selection.
* Added recovery for corrupt, stale and non-finite embeddings.
* Improved numeric and structured-identifier retrieval behaviour.

### Cases

* Expanded case review workflows.
* Findings automatically register cited evidence with their case.
* Added downloadable Markdown case reports.
* Added stronger evidence selection throughout the analyst workflow.

### Local-model analysis

* Added citation-validation metadata.
* Added detection of invented citation labels.
* Preserved deterministic controls outside model generation.

### Storage

* Added tracked database schema migrations.
* Added integrity-checked database backups.
* Added migration and backup regression coverage.

### Analyst console

* Added source browsing.
* Added source-filtered retrieval controls.
* Added case-evidence search.
* Expanded case review.
* Added report downloads.

### Reliability and testing

* Expanded the regression suite to **121 tests**.
* Added migration tests.
* Added database-backup tests.
* Added repeat-import provenance tests.
* Added corrupt-embedding recovery tests.
* Added quarantine regression tests.
* Added selected-evidence tests.
* Added citation-hallucination tests.
* Added large-corpus bounded-memory retrieval testing.

---

## Security

MIKA is designed around:

* least privilege;
* complete mediation;
* explicit authorization;
* typed boundaries;
* provenance;
* reviewability;
* local-first operation.

The project has regression coverage for threats including:

* implicit permission regression;
* unregistered tool invocation;
* malformed tool arguments;
* filesystem traversal;
* SQLite modification attempts;
* prompt-injection content;
* quarantine bypass;
* audit-log modification;
* connector host substitution;
* unauthorized HTTP requests.

See [`SECURITY.md`](SECURITY.md) and [`SECURITY_REVIEW.md`](SECURITY_REVIEW.md) for the complete security model and release review.

---

## Security Boundaries

MIKA should not be treated as an internet-facing production service.

Important limitations include:

* browser authentication is designed for local single-user operation;
* evidence databases are not application-encrypted;
* a compromised operating-system account can access local files;
* an administrator can access the application state;
* malicious browser extensions may access browser content;
* audit chaining is tamper-evident rather than externally signed;
* prompt-injection detection is heuristic;
* generated analysis still requires human evidence review;
* CLI role selection is not OS-level identity authentication;
* MIKA cannot control the telemetry or network behaviour of external software such as Ollama;
* dependency scans do not replace OS, Python runtime or native-library security maintenance.

Do not place highly sensitive information into MIKA without first assessing whether the host system and storage controls are appropriate for that data.

---

## Project Structure

```text
MIKA_v1.4.0/
├── dashboard/
│   ├── src/
│   ├── dist/
│   └── package.json
├── data/
├── docs/
│   └── adr/
├── mika/
│   ├── connectors/
│   ├── evaluation/
│   ├── tools/
│   ├── analyst.py
│   ├── api.py
│   ├── app.py
│   ├── audit.py
│   ├── cases.py
│   ├── embeddings.py
│   ├── entity_resolution.py
│   ├── policy.py
│   ├── provenance.py
│   ├── relationships.py
│   ├── retrieval.py
│   ├── security.py
│   ├── storage.py
│   └── telemetry.py
├── benchmarks/
├── reports/
├── samples/
├── scripts/
├── tests/
├── CHANGELOG.md
├── CONTRIBUTING.md
├── SECURITY.md
├── SECURITY_REVIEW.md
├── WINDOWS_SETUP.md
├── WINDOWS_UPGRADE.md
└── pyproject.toml
```

---

## Documentation

Additional documentation:

* [`docs/architecture.md`](docs/architecture.md) — system architecture
* [`docs/evaluation.md`](docs/evaluation.md) — evaluation methodology
* [`docs/case-study.md`](docs/case-study.md) — example operational workflow
* [`docs/public-data.md`](docs/public-data.md) — public-data connector
* [`docs/references.md`](docs/references.md) — design references
* [`docs/regression-baseline.md`](docs/regression-baseline.md) — benchmark baseline
* [`SECURITY.md`](SECURITY.md) — security architecture
* [`SECURITY_REVIEW.md`](SECURITY_REVIEW.md) — v1.4.0 review
* [`WINDOWS_SETUP.md`](WINDOWS_SETUP.md) — Windows installation
* [`WINDOWS_UPGRADE.md`](WINDOWS_UPGRADE.md) — upgrade instructions
* [`CHANGELOG.md`](CHANGELOG.md) — release history
* [`CONTRIBUTING.md`](CONTRIBUTING.md) — development workflow

---

## Roadmap

Areas for future development include:

* larger-corpus vector indexing;
* stronger labelled entity-resolution evaluation;
* richer case timelines;
* evidence comparison and contradiction detection;
* additional deterministic analysis tools;
* improved document-format ingestion;
* expanded retrieval evaluation datasets;
* stronger external audit anchoring;
* optional multi-user architecture separated from the current local runtime;
* further adversarial testing of retrieval and model-grounding behaviour.

Any future network or multi-user deployment should be treated as a separate security architecture rather than an extension of the current local server configuration.

---

## License

MIKA is released under the **MIT License**.

See [`LICENSE`](LICENSE).

---

## Disclaimer

MIKA is an engineering and research project for evidence analysis and AI assurance.

It does not replace professional legal, compliance, financial, security or investigative judgement. Generated analysis and entity matches should be reviewed against the underlying evidence before decisions are made.
