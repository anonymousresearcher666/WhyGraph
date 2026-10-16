# WhyGraph: anonymous reproducibility artifact (SIGMOD 2027 submission)

Contents:

- `why-query-neo4j/`: the Neo4j procedure library (Java/Maven) evaluated in the paper, with its unit tests.
- Final rerun: not included in this historical-only package.
- `evaluation/historical/`: dated earlier evidence; do not present it as single-build final results.
- `evaluation/vldb/`: frozen workloads and profiles; `evaluation/scripts/`: generators, runners, scorers.
- `human_study/`: historical study materials.

Evidence groups:

- `budgeted_evidence_policy`: 8 files
- `cross_scale_graph_instance_holdout`: 7 files
- `extensions_and_thresholds`: 15 files
- `fragment_coverage`: 22 files
- `historical_baselines`: 4 files
- `independent_transfer`: 18 files
- `llm`: 4 files
- `organic_shapes`: 3 files
- `paired_hardened`: 11 files
- `planner_runtime_audit`: 1 files
- `query_anchor_holdout`: 9 files
- `resource_isolation`: 8 files
- `snapshot_cost_and_freshness`: 6 files
- `why_not_blocker_detection`: 8 files
- `worked_example`: 2 files

JSON/JSONL files larger than 1 MB are stored as deterministic gzip (`.jsonl.gz`). SHA-256 hashes of every file are in `MANIFEST.json`. No credentials, local paths, database files or author identifiers are included (`PRIVACY_AUDIT.json`). Supply Neo4j credentials via `NEO4J_PASSWORD` only.

The code is released under the MIT License (`LICENSE`).
