# MIKA

**MIKA** is a local-first evidence-analysis and AI-assurance system. It turns documents and structured records into searchable, source-linked evidence while testing whether retrieval, tool access and audit controls behave as intended.

It is designed for analytical workflows where an answer should be traceable to evidence, permissions should be explicit, and evaluation should be reproducible.

## What MIKA does

MIKA supports a complete local analysis workflow:

1. **Ingest evidence** from text files, CSVs and approved data connectors.
2. **Preserve provenance** including source, metadata and ingestion context.
3. **Retrieve relevant evidence** using hybrid lexical and vector-style ranking.
4. **Analyse structured data** through profiling, entity matching and relationship graphs.
5. **Control tool use** through typed, allow-listed tools with default-deny behaviour.
6. **Detect suspicious content** and quarantine likely prompt-injection attempts.
7. **Record actions** in a tamper-evident chained audit trail.
8. **Evaluate the system** against retrieval, authorisation, quarantine and latency checks.
9. **Expose results locally** through a CLI, API and optional browser-based analyst console.

MIKA can also use a local Ollama model to synthesise answers from retrieved evidence. Generated answers should still be reviewed against the cited source material.

## Core principles

- **Evidence before assertion:** findings should link back to source passages.
- **Local-first operation:** data and services can remain on the local machine.
- **Default-deny controls:** a tool must be explicitly advertised and authorised before use.
- **Reproducible evaluation:** benchmark definitions, source fingerprints and evaluation outputs can be retained for comparison.
- **Analyst review:** automated outputs support investigation; they do not replace judgement.

## Features

### Provenance-aware ingestion

MIKA ingests evidence while retaining source and metadata information. This makes it possible to inspect where a result came from and distinguish source content from a model-generated interpretation.

### Hybrid retrieval

Search combines lexical matching with embedding-based similarity where available. The system can run with deterministic hash embeddings offline or use `sentence-transformers` for semantic retrieval.

Retrieval results include the evidence passages required for an analyst to verify the result.

### Entity and relationship analysis

MIKA can resolve possible entity matches across imperfect records using:

- Alias handling
- Unicode normalisation
- Token blocking
- Fuzzy matching
- Explainable match signals

It can also traverse relationship data to find bounded paths between entities, helping investigators understand connections across structured records.

### Data-quality profiling

CSV profiling identifies useful quality signals, including missing values, duplicates and column-level characteristics. This provides a quick check before using a dataset in analysis.

### Controlled tools and audit records

Tool calls are typed and controlled through an allow-list. Unknown, unadvertised or unauthorised tools are denied by default.

Audit events are chained so later modifications can be detected during verification. MIKA records the important actions taken during an investigation or evaluation run.

### Prompt-injection quarantine

MIKA scans ingested content for suspicious instruction-like patterns, such as attempts to override system behaviour or request secrets. Flagged content is quarantined and excluded from ordinary retrieval by default.

This is defence in depth, not a complete security boundary.

### Evaluation harness

MIKA includes a reproducible evaluation suite covering:

- Retrieval quality, including recall, precision, MRR and NDCG
- Tool-authorisation behaviour
- Default-deny behaviour for unknown tools
- Prompt-injection quarantine checks
- Audit-chain integrity
- Local retrieval-latency percentiles

The included operations benchmark contains 120 synthetic retrieval cases. It is intended to detect regressions between changes; it is not evidence of production-domain accuracy.

## Architecture

```text
Evidence files / CSVs / approved connectors
                  │
                  ▼
      Ingestion + provenance records
                  │
                  ▼
 Retrieval index + entity/relationship data
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
     Search    Analysis    Evaluation
        │         │          │
        └─────────┴──────────┘
                  │
                  ▼
       CLI, local API and analyst console
