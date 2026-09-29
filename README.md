<div align="center">

# MIKA

### Model Integrity & Knowledge Analysis

**Local evidence intelligence, investigation tooling and AI assurance**

`v1.6.0`

Python · FastAPI · SQLite · TypeScript · NumPy · Ollama

[GitHub Repository](https://github.com/RhainRK/MIKA)

</div>

<br>

> **The model should help with analysis. It should never be the thing you have to blindly trust.**

MIKA is a local evidence analysis system I built for working with documents, structured datasets and investigation material.

The main idea is pretty simple.

You should be able to search evidence, build cases, trace where information came from and optionally use AI to help analyse it without giving the model control over the system.

MIKA keeps retrieval, permissions, provenance, tool access and audit controls outside the language model.

The AI layer is optional.

The evidence system is not.

## What MIKA actually does

Think of MIKA as an investigation workspace with an AI layer attached to it.

You can give it evidence such as documents or structured data and then use it to:

* search across evidence
* find exact identifiers
* find semantically related information
* trace results back to their original source
* organise evidence into cases
* record findings and supporting evidence
* compare possible explanations
* map relationships between entities
* profile structured datasets
* inspect suspicious imported content
* use a local model to analyse selected evidence
* check whether generated citations actually exist
* control which tools the model is allowed to use
* keep an audit trail of important actions
* measure retrieval quality and latency

MIKA can work without a language model at all.

Retrieval, ingestion, cases, provenance, authorization, evaluation and the analyst interface all work independently.

## Why I built it

A lot of AI systems work roughly like this:

```text
Documents
    ↓
Language Model
    ↓
Answer
```

That is useful until you need to answer questions like:

```text
Where did this claim come from?

Was this actually in the evidence?

Did the model invent that citation?

Why was this result ranked first?

What files did the model have access to?

Was that tool even allowed to run?

Can I reproduce the same analysis later?
```

MIKA is built around making those questions easier to answer.

The system looks more like this:

```text
                    SOURCE DATA
                         │
                         ▼
                  ┌─────────────┐
                  │  INGESTION  │
                  └──────┬──────┘
                         │
                provenance + hashes
                         │
                         ▼
                  ┌─────────────┐
                  │ TRUST CHECK │
                  └──────┬──────┘
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼

       TRUSTED EVIDENCE        QUARANTINED DATA
             │
             ▼
       ┌───────────────┐
       │ SQLITE STORE  │
       │ FTS5 + chunks │
       └───────┬───────┘
               │
        ┌──────┴──────┐
        │             │
        ▼             ▼

   LEXICAL SEARCH   SEMANTIC SEARCH
     FTS5 / BM25       embeddings

        │             │
        └──────┬──────┘
               ▼

        RECIPROCAL RANK
             FUSION
               │
               ▼
       EVIDENCE SELECTION
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼

   CASE WORKFLOW     LOCAL MODEL
                         optional
       │                │
       │                ▼
       │          citation checking
       │                │
       └────────┬───────┘
                ▼

          ANALYST OUTPUT
```

The language model sits near the end of the pipeline.

Not at the centre of the trust model.

## Core system

### Evidence ingestion

MIKA stores provenance alongside imported evidence instead of turning everything into an anonymous text collection.

Evidence can retain information including:

* source identifiers
* SHA 256 fingerprints
* chunk identifiers
* import metadata
* quarantine state
* source inventory information

Unchanged evidence can be imported again while preserving existing chunk references.

That matters because cases and findings may already reference those chunks.

A normal reimport should not silently break previous analysis.

## Source register

MIKA keeps a register of imported sources.

You can inspect information such as:

* source identity
* source fingerprint
* chunk count
* import count
* quarantine state
* metadata

```bash
mika sources
```

Inspect a source:

```bash
mika source 1
```

The browser interface does not expose absolute workspace paths.

## Hybrid retrieval

MIKA uses two different approaches to search.

### Lexical retrieval

Useful when the exact wording matters.

Built around:

* SQLite FTS5
* BM25 ranking
* structured identifier handling

### Semantic retrieval

Useful when the query and source mean similar things but use different wording.

Semantic retrieval uses local embeddings and vector similarity.

The two result sets are combined using **Reciprocal Rank Fusion**.

```text
Exact wording               Similar meaning
     │                            │
     ▼                            ▼
 FTS5 / BM25                 embeddings
     │                            │
     └────────────┬───────────────┘
                  ▼
                 RRF
                  │
                  ▼
             final ranking
```

This means MIKA can deal with both natural language questions and exact identifiers such as:

```text
CASE 1042
ACC 00831
INV 9214
```

Exact identifiers receive additional ranking treatment so a vaguely similar result should not replace an exact record match.

## Search only what matters

Sometimes searching every document is the wrong thing to do.

MIKA can limit retrieval to selected:

* sources
* evidence chunks
* case evidence

For example:

```bash
mika search "reconciliation anomaly" --source 1
```

or:

```bash
mika search "payment discrepancy" --case case_xxxxxxxxxxxx
```

This is useful when analysis should only use a deliberately approved evidence set.

## Case workspace

MIKA includes a case workflow for turning retrieved evidence into something more structured.

A case can hold:

```text
Objective
   │
   ├── Evidence
   │
   ├── Findings
   │
   ├── Confidence
   │
   ├── Alternatives
   │
   └── Limitations
```

Create one:

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

Add a finding:

```bash
mika case-finding case_xxxxxxxxxxxx \
"The discrepancy is associated with the settlement batch." \
--confidence 0.86 \
--evidence 12 18
```

Export the case:

```bash
mika case-report case_xxxxxxxxxxxx
```

The browser interface supports the same general workflow.

## Findings stay attached to evidence

A finding can include:

* supporting evidence
* confidence
* alternative explanations
* limitations

This is intentional.

The point is not to convert an AI answer into a fact.

MIKA keeps the conclusion and the evidence behind it separately reviewable.

## Citation validation

If the optional local model produces an answer, MIKA checks the citation labels it returns against the evidence that was actually supplied.

That means MIKA can detect cases where a model produces something that looks like:

```text
According to [SOURCE 14] ...
```

when `SOURCE 14` was never part of the retrieved evidence.

Citation validation does not prove that every sentence is correct.

It checks whether the citation actually belongs to the evidence set the model received.

There is still a human at the end of the process.

## Entity resolution

Real datasets are messy.

The same entity might appear as:

```text
Northstar Computing Ltd
Northstar Computing
Northstar Comp
NORTHSTAR COMPUTING LTD.
```

MIKA includes deterministic record matching using techniques such as:

* Unicode normalization
* alias handling
* token blocking
* weighted fuzzy similarity
* explainable scoring

Example:

```bash
mika entity-resolve samples/entities.csv "Northstar Comp"
```

The resolver gives the analyst possible matches.

It does not automatically decide that two records are definitely the same entity.

## Relationship analysis

MIKA can also work with relationship data.

The graph layer uses adjacency lists and bounded breadth first search to trace paths between entities.

```bash
mika graph-path samples/relationships.csv "Maya Chen" "Helios Research Group"
```

This can help explore things like:

* organisational links
* transaction relationships
* ownership
* operational dependencies
* account relationships

The traversal is bounded so unexpectedly large or malformed graphs cannot trigger unlimited exploration.

## Data quality

Before analysing a dataset, sometimes the most useful thing is simply figuring out whether the data is any good.

MIKA can profile CSV evidence:

```bash
mika profile samples/operations_incidents.csv
```

This provides a deterministic first look at structured evidence before it reaches later stages of analysis.

## AI is optional

MIKA does not require a model for its core functionality.

With no LLM enabled you still have:

```text
Ingestion
Retrieval
Cases
Provenance
Source tracking
Entity resolution
Relationship analysis
Evaluation
Tool authorization
Audit verification
Analyst console
```

If generated analysis is useful, a local Ollama model can be added.

```bash
pip install -e ".[llm]"
```

Example:

```bash
ollama pull qwen3.5:4b
```

Then:

```bash
mika ask "What evidence explains the reconciliation anomaly?"
```

MIKA only allows configured plaintext model endpoints on literal loopback addresses such as:

```text
http://127.0.0.1:11434
```

The MIKA client also disables inherited HTTP proxy configuration for model requests.

External model software is still external software.

MIKA cannot control its telemetry or network behaviour.

## Tool security

One of the parts I care about most in MIKA is that the model does not decide what it is allowed to do.

A requested tool action goes through a deterministic boundary first.

```text
MODEL REQUEST
      │
      ▼
Is the tool registered?
      │
   no │ yes
      │
 reject
      │
      ▼
Is it permitted?
      │
   no │ yes
      │
 reject
      │
      ▼
Validate arguments
      │
 invalid
      │
   reject
      │
      ▼
   EXECUTE
```

A tool must be:

1. registered
2. permitted by policy
3. called with arguments that pass its typed schema

Unknown tools are rejected.

Arguments are validated with Pydantic before execution.

The model cannot grant itself additional permissions.

## Workspace confinement

File tools are restricted to the configured workspace.

Path traversal attempts outside that workspace are rejected.

MIKA also contains a bounded read only SQLite capability.

It accepts controlled `SELECT` and `WITH` queries while rejecting operations involving:

* writes
* schema modification
* database attachment
* unsafe pragmas

Useful analysis should not require turning the model into an unrestricted filesystem or database user.

## Prompt injection handling

Imported evidence is data.

It is not trusted instruction text.

MIKA can scan evidence for suspicious prompt injection content and quarantine it.

Quarantined chunks are excluded from normal retrieval unless deliberately inspected.

The detection layer is not treated as perfect.

Even if malicious text gets through it, the real security boundary remains the deterministic authorization system.

## Audit trail

MIKA records analytical and security relevant activity in a JSONL audit log.

Each entry includes:

* event data
* the previous record hash
* its own SHA 256 digest

That creates a chain where editing or reordering an existing record breaks verification further down the log.

Verify it with:

```bash
mika audit-verify
```

This is tamper evident.

It is not magically immutable.

Someone capable of replacing the complete audit log and recomputing the full chain could create a new internally valid log.

A production deployment would normally use external integrity anchoring or central append only logging.

## Local analyst console

MIKA includes a FastAPI backend with a TypeScript browser interface.

The console supports:

* evidence search
* source browsing
* source filtered search
* case evidence search
* case review
* finding management
* Markdown case reports
* evaluation information

Start it:

```bash
mika serve --host 127.0.0.1 --port 8000
```

Then open:

```text
http://127.0.0.1:8000
```

MIKA creates a fresh browser password when the server starts.

Username:

```text
mika
```

The generated password is printed in the terminal.

Stopping the process invalidates that session password.

## Local web security

The browser console is designed as a **single user local application**.

It is not an internet facing web platform.

Controls include:

* loopback only binding
* peer address checks
* Host validation
* Origin validation
* Fetch Metadata checks
* restricted HTTP methods
* request size limits
* rate and resource budgets
* disabled public API documentation
* disabled proxy header trust
* disabled HTTP access logging
* Content Security Policy
* frame denial
* no store caching
* referrer restrictions
* browser permissions policy
* same origin resource policy

Do not expose it using:

```text
Router port forwarding
Public network interfaces
Reverse proxies
Public tunnels
Internet facing hosts
```

If MIKA ever becomes a multi user or remotely hosted system, that should be treated as a separate security architecture.

Not as a switch you turn on.

## Public data connector

MIKA includes a deliberately narrow connector for the official UK FCDO sanctions dataset.

```bash
mika fetch-uk-sanctions
```

The connector uses host and endpoint restrictions so it cannot quietly become a general purpose HTTP client.

MIKA does not make sanctions, legal or compliance decisions.

It gives an analyst evidence to review.

## Evaluation

MIKA includes a synthetic operational retrieval benchmark and regression suite.

Metrics include:

* Precision at k
* Recall at k
* Mean Reciprocal Rank
* nDCG
* bootstrap confidence intervals
* latency percentiles

### Published regression baseline

| Check | Result |
|:--|--:|
| Python tests | **121 passing** |
| Python branch coverage | **88%** |
| Retrieval benchmark | **120 cases** |
| Recall@5 | **1.000** |
| MRR | **1.000** |
| nDCG@5 | **1.000** |
| Default deny authorization | **Pass** |
| Unknown tool rejection | **Pass** |
| Prompt injection quarantine | **Pass** |
| Audit chain verification | **Pass** |
| TypeScript typecheck | **Pass** |
| TypeScript build | **Pass** |

These figures are regression measurements from the bundled synthetic benchmark.

They are there to catch engineering regressions.

They are not a claim that MIKA achieves perfect accuracy on arbitrary real world investigations.

Evaluation results and dataset fingerprints are stored in:

```text
reports/evaluation-results.json
```

## A simple MIKA workflow

```text
01  IMPORT
        │
        ▼
    evidence enters MIKA

02  VERIFY
        │
        ▼
    provenance and trust checks

03  SEARCH
        │
        ▼
    lexical + semantic retrieval

04  NARROW
        │
        ▼
    choose approved evidence

05  INVESTIGATE
        │
        ▼
    entities, relationships, data quality

06  BUILD CASE
        │
        ▼
    evidence + findings + uncertainty

07  ANALYSE
        │
        ▼
    optional local model

08  CHECK
        │
        ▼
    citations + audit trail

09  EXPORT
        │
        ▼
    reviewable case report
```

## Installation

### Requirements

* Python 3.11+
* Node.js and npm if rebuilding the browser interface
* Ollama only if using local generated analysis

Clone the project:

```bash
git clone https://github.com/RhainRK/MIKA.git
cd MIKA
```

Create a virtual environment:

```bash
python -m venv .venv
```

### Windows PowerShell

```powershell
.\.venv\Scripts\Activate.ps1
```

### Linux and macOS

```bash
source .venv/bin/activate
```

Install MIKA:

```bash
python -m pip install --upgrade pip setuptools
pip install -e ".[server]"
```

Check it:

```bash
mika --version
mika doctor
```

Expected release:

```text
MIKA 1.6.0
```

## Build the dashboard

```bash
cd dashboard
npm ci --ignore-scripts
npm run typecheck
npm run build
cd ..
```

Run MIKA:

```bash
mika serve --host 127.0.0.1 --port 8000
```

## First five commands

If you just cloned MIKA and want to see what it does:

### 1. Import evidence

```bash
mika ingest samples/benchmark_evidence.txt
```

### 2. See registered sources

```bash
mika sources
```

### 3. Search

```bash
mika search "reconciliation anomaly"
```

### 4. Create a case

```bash
mika case-create "Operations Investigation"
```

### 5. Check the audit chain

```bash
mika audit-verify
```

That already gives you most of the core MIKA workflow.

## Database backups

Create a verified SQLite backup:

```bash
mika backup
```

Default location:

```text
data/backups/
```

MIKA verifies the SQLite snapshot before replacing the backup destination.

## Database migrations

MIKA tracks database schema versions.

Supported older databases can be migrated while preserving existing:

* evidence
* cases
* findings

Back up the database before upgrading between releases.

## Embeddings

The basic MIKA install includes deterministic offline embedding behaviour.

A trained semantic model is optional.

For local semantic models:

```bash
pip install -e ".[semantic]"
```

Then point:

```text
MIKA_EMBEDDING_MODEL
```

to a trusted local model path.

FTS5 and BM25 remain available without semantic embeddings.

## Development

Install development dependencies:

```bash
pip install -e ".[dev,server]"
```

Run tests:

```bash
pytest -q
```

Coverage:

```bash
pytest --cov=mika --cov-branch --cov-fail-under=82 -q
```

Linting:

```bash
ruff check .
ruff format --check .
```

Typing:

```bash
mypy mika
```

Security checks:

```bash
bandit -r mika
pip-audit
npm audit --prefix dashboard
```

Release checks:

```bash
python scripts/release_check.py
```

Frontend:

```bash
npm run --prefix dashboard typecheck
npm run --prefix dashboard build
```

CI runs the relevant formatting, typing, security, test and build checks automatically.

Dependabot is configured for Python, npm and GitHub Actions dependencies.

## Security philosophy

MIKA is built around a few rules I try not to compromise on.

```text
Least privilege

Explicit authorization

Typed boundaries

Evidence provenance

Reviewable decisions

Local operation by default

The model is not the security boundary
```

Regression coverage includes scenarios involving:

* implicit permission changes
* unknown tool invocation
* malformed tool arguments
* filesystem traversal
* SQLite modification attempts
* prompt injection
* quarantine bypass
* audit log modification
* connector host substitution
* unauthorized HTTP requests

More detail lives in:

```text
SECURITY.md
SECURITY_REVIEW.md
```

## Known boundaries

MIKA is still an engineering and research project.

Things worth being clear about:

* the browser authentication model is for local single user use
* evidence databases are not application encrypted
* someone with access to the operating system account may access local files
* an administrator can access application state
* malicious browser extensions can potentially access browser content
* the audit chain is tamper evident rather than externally signed
* prompt injection detection is heuristic
* generated analysis still needs human review
* CLI role selection is not operating system identity authentication
* MIKA cannot control external software such as Ollama
* dependency scanning does not replace keeping the operating system and runtime secure

Do not put highly sensitive data into MIKA without first deciding whether the host machine and storage environment are appropriate for it.

## Project structure

```text
MIKA/
│
├── dashboard/
│   ├── src/
│   ├── dist/
│   └── package.json
│
├── data/
│
├── docs/
│   └── adr/
│
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
│
├── benchmarks/
├── reports/
├── samples/
├── scripts/
├── tests/
│
├── CHANGELOG.md
├── CONTRIBUTING.md
├── SECURITY.md
├── SECURITY_REVIEW.md
├── WINDOWS_SETUP.md
├── WINDOWS_UPGRADE.md
└── pyproject.toml
```

## Documentation

More detail is split into the project docs so this README does not have to explain every internal decision.

| Document | What it covers |
|:--|:--|
| `docs/architecture.md` | System architecture |
| `docs/evaluation.md` | Evaluation methodology |
| `docs/case-study.md` | Example investigation workflow |
| `docs/public-data.md` | Public data connector |
| `docs/references.md` | Design references |
| `docs/regression-baseline.md` | Benchmark baseline |
| `SECURITY.md` | Security architecture |
| `SECURITY_REVIEW.md` | Security review |
| `WINDOWS_SETUP.md` | Windows installation |
| `WINDOWS_UPGRADE.md` | Upgrade process |
| `CHANGELOG.md` | Release history |
| `CONTRIBUTING.md` | Development workflow |

## Where I want to take it

MIKA is already useful as a local investigation and assurance environment, but there is a lot more I want to explore.

Some directions I am interested in:

```text
larger evidence collections

better document ingestion

richer investigation timelines

contradiction detection

evidence comparison

stronger entity evaluation

more deterministic analysis tools

better retrieval benchmarks

external audit anchoring

stronger adversarial testing

better analyst visualisation
```

Longer term I am also interested in what a separate multi user architecture could look like.

That would need a proper security design of its own.

I do not want to turn the local server into an internet service by slowly removing the restrictions that currently make it safe.

## Design principle

If I had to reduce MIKA to one idea, it would be this:

```text
Use AI for the part AI is good at.

Do not make AI responsible for the parts that need to be trusted.
```

Retrieval should be measurable.

Permissions should be explicit.

Evidence should have provenance.

Important actions should be auditable.

Generated analysis should be reviewable.

And the user should always be able to get back to the source.

## License

MIKA is released under the MIT License.

See `LICENSE`.

## Disclaimer

MIKA is an engineering and research project for evidence analysis and AI assurance.

It does not replace professional legal, compliance, financial, security or investigative judgement.

Generated analysis, entity matches and automated findings should always be checked against the underlying evidence before decisions are made.

<div align="center">

### MIKA v1.6.0

**Evidence first. Models second.**

[github.com/RhainRK/MIKA](https://github.com/RhainRK/MIKA)

</div>
