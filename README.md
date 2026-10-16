# WhyGraph Neo4j Plugin User Guide

## Goals and scope
WhyGraph explains why a Cypher query is empty, returns too few results, or
returns too many results. It does this by analyzing the query AST, schema
snapshot, and statistics cache, then proposing rewrites. By default, it
operates in estimate-only mode (no data scans or query execution).

Core use cases:
- Why-empty: explain why a query returns no results and suggest a minimal fix.
- Why-so-few: explain why a query returns fewer results than expected.
- Why-so-many: explain why a query returns more results than expected.
- Why-not: explain why a specific tuple/entity is missing from results.

## Build and install
From repo root:

```bash
./why-query-neo4j/scripts/build.sh
```

Install into a Neo4j Desktop DBMS:

```bash
./why-query-neo4j/scripts/install.sh "/path/to/dbms-xxxx"
```

Restart Neo4j, then verify the procedures are available:

```cypher
CALL dbms.procedures() YIELD name
WHERE name STARTS WITH 'why'
RETURN name;
```

### Neo4j config (recommended)
In `neo4j.conf` add:

```
dbms.security.procedures.unrestricted=why.*
dbms.security.procedures.allowlist=why.*
```

## Cache and snapshot management
WhyGraph uses two snapshots stored next to the plugin jar:
- Schema snapshot: `why_schema_cache.json`
- Statistics snapshot: `whygraph-statistics.json`

Override the schema snapshot path if needed:
- Environment variable: `WHY_SCHEMA_SNAPSHOT_PATH`
- Java system property: `why.schema.snapshot.path`

Inspect cache state:

```cypher
CALL why.cache_info();
```

Reload snapshots from disk (no data scans):

```cypher
CALL why.refresh_schema_cache();
CALL why.refresh_statistics_cache();
```

Rebuild snapshots from live data (explicit opt-in required):

```cypher
CALL why.rebuild_schema_snapshot({allow_snapshot_build: true});
CALL why.rebuild_statistics_snapshot({allow_snapshot_build: true});
```

You can also use `{allow_data_queries:true}` instead of
`allow_snapshot_build:true`.

## Core procedures
All core procedures accept an optional `options` map as the final argument.

### Why-empty
```cypher
CALL why.empty(
  'MATCH (p:Person {name:"NonExistent"})-[:LIVES_IN]->(c) RETURN p, c',
  {use_schema:true}
);
```

### Why-so-many
```cypher
CALL why.so_many(
  'MATCH (p:Person)-[:KNOWS]->(f) RETURN p, f',
  1000,
  {allow_add_limit:true}
);
```

### Why-so-few
```cypher
CALL why.so_few(
  'MATCH (p:Person {country:"Iceland"}) RETURN p',
  100,
  {use_schema:true}
);
```

### Why-not (missing example)
`missing` maps variables to labels/properties.

```cypher
CALL why.not(
  'MATCH (p:Person)-[:WORKS_AT]->(c:Company) RETURN p, c',
  {
    p: {labels: ["Person"], properties: {name: "Alice"}},
    c: {labels: ["Company"], properties: {name: "Acme"}}
  },
  {}
);
```

### Cardinality estimation (function)
```cypher
RETURN why.estimate_cardinality(
  'MATCH (p:Person)-[:LIVES_IN]->(c:City) RETURN p'
) AS estimatedCount;
```

## Rewriting strategies
Use `rewriting_strategy` to choose the algorithm:
- `astar` (default): schema/stats-guided A* search (estimate-only).
- `weighted-astar` (or `wastar`): faster, bounded-suboptimal A*.
- `greedy`: lightweight greedy selection.
- `vasilyeva`: subgraph relaxation baseline (data-driven if allowed).
- `song-schema`: schema-guided baseline inspired by Song et al. (estimate-only).
- `song-data`: data-driven Song baseline (uses live data if allowed).

Example:
```cypher
CALL why.empty(
  'MATCH (p:Person)-[:WORKS_AT]->(c:Company {name:"X"}) RETURN p, c',
  {rewriting_strategy: "weighted-astar", weighted_astar_weight: 1.5}
);
```

## Options reference
Common options (defaults in parentheses):
- `use_schema` (true): use schema snapshot for validation/pruning.
- `use_formal_bounds` (true): require a fresh safe ceiling for metadata-only why-many goals. This does not make the practical heuristic admissible.
- `use_calibrated_bounds` (false): calibrate bounds using workload scaling.
- `use_soft_upper_bounds` (false): allow softer upper bounds.
- `use_planner_stats` (false): use EXPLAIN estimates.
- `allow_planner_calls` (false): must be true if using `use_planner_stats`.
- `use_rewriting` (true): enable rewrite generation.
- `force_astar_rewrite` (false): force rewrite even if prechecks pass.
- `force_rewrite_after_precheck` (false): continue after schema/value checks.
- `rewriting_strategy` ("astar"): choose strategy (see above).
- `weighted_astar_weight` (1.5 when using weighted A*): w > 1.
- `admissible_heuristic` (false): enforce admissible heuristic.
- `allow_unbounded_ops` (true): permit unbounded ops (e.g., delete hubs).
- `unbounded_penalty` (5.0): extra cost for unbounded ops.
- `allow_add_limit` (true): allow LIMIT rewrite for why-so-many.
- `allow_relationship_repairs` (true): allow relationship repairs.
- `allowed_ops` ([]): list of allowed operations by enum name.
- `enforce_connected` (true): keep patterns connected.
- `modification_threshold` (5.0): max cost threshold for rewrites.
- `max_iterations` (-1): cap A* expansions (use -1 for no cap).
- `max_alternatives` (-1): cap number of rewrite alternatives.
- `timeout_seconds` (300): time budget for rewrite search.
- `refresh_stats` (false): refresh caches before analysis
  (requires `allow_data_queries:true`).
- `allow_data_queries` (false): allow live data queries for counts/validation.
- `allow_snapshot_build` (false): allow snapshot rebuild procedures.

Allowed operations use `ModificationOperation.Type` names, e.g.:
`DELETE_NODE`, `DELETE_RELATIONSHIP`, `DELETE_PREDICATE`, `RELAX_LABEL`,
`MAKE_OPTIONAL`, `ADD_PREDICATE`, `ADD_LABEL`, `ADD_LIMIT`,
`RESTRICT_PROPERTY`, `REPLACE_RELATIONSHIP_TYPE`, `FLIP_RELATIONSHIP_DIRECTION`.

### User preferences (optional)
You can bias rewrites via `user_preferences`:
```cypher
CALL why.empty(
  'MATCH (p:Person)-[:WORKS_AT]->(c:Company) RETURN p, c',
  {
    user_preferences: {
      operation_penalties: {DELETE_NODE: 20.0, DELETE_PREDICATE: 1.0},
      preferred_labels: ["Person"],
      avoided_labels: ["Organization"],
      preferred_relationship_types: ["WORKS_AT"],
      avoided_relationship_types: ["KNOWS"],
      critical_properties: ["name"],
      relaxable_properties: ["age"],
      prefer_optional_match: true,
      preserve_order_by: true,
      preserve_limit: true
    }
  }
);
```

## Estimate-only vs data-driven
By default, WhyGraph does not execute candidate queries. It relies on schema
and statistics snapshots plus optional EXPLAIN estimates. To validate rewrites
against actual data (or to use data-driven baselines), set:

```cypher
{allow_data_queries: true}
```

Use this sparingly on large graphs, and consider `timeout_seconds`.

## LLM explanations (optional)
LLM support is off by default. When enabled, it can produce:
- Natural language explanations of why a query is empty/too few/too many.
- LLM-suggested rewrites (schema-validated before surfacing).

Enable by providing an LLM config file:
- Place `whygraph-llm-config.json` (or `llm-config.json`) next to the plugin jar, or
- Set the system property `why.llm.config=/path/to/config.json`.

Then call:
```cypher
CALL why.empty('MATCH ...', {enable_llm:true, llm_mode:"both"});
```

LLM-related options:
- `enable_llm` (false): master toggle.
- `llm_mode` ("both"): `"both"`, `"explain"`, `"rewrite"`, or `"none"`.
- `enable_llm_explanations` (true when LLM on): explicitly enable explanations.
- `enable_llm_rewrite_suggestions` (true when LLM on): enable rewrite suggestions.

Example: explanations only, no LLM rewrites
```cypher
CALL why.so_few(
  'MATCH (p:Person) WHERE p.age > 200 RETURN p',
  10,
  {enable_llm:true, llm_mode:"explain"}
);
```

## Troubleshooting
Common issues and fixes:
- Procedure not found: verify plugin jar in `plugins/`, restart Neo4j, check
  allowlist/unrestricted config.
- Schema cache empty: run `CALL why.rebuild_schema_snapshot({allow_snapshot_build:true});`.
- Statistics stale: run `CALL why.rebuild_statistics_snapshot({allow_snapshot_build:true});`.
- EXPLAIN not used: set both `allow_planner_calls:true` and `use_planner_stats:true`.
- LLM not running: check config file placement and logs.

## Certificate evidence in `why.*`

For why-many, `upper_target_certified=true` certifies only the reported ceiling.
`target_certified=false` and `target_validation_required=true` indicate that
independent validation must still establish `1 <= actual_count <= target`.
An empty rewrite is not a successful repair. When candidate counts are enabled,
why-many goals require at least one row, including in the greedy path; a caller's
higher requested floor is preserved.

The ceiling counts physical relationship choices, including parallel edges,
cycles and self-loops. A unique child node can have several incident
relationships; its key does not cap edge multiplicity. The snapshot-product path rejects
undirected atoms, edge-bearing disconnected patterns, nonconjunctive filters,
optional patterns, aggregation and unsupported subquery semantics. Edge-free
Cartesian products use a product of population ceilings. The input guard is
conservative because the simplified parser does not preserve every construct.

Regeneration preserves separate `MATCH` clauses and their intervening filters.
Cypher's relationship uniqueness applies within each clause; a relationship
may be reused across separate clauses. Named relationship variables and inline
relationship properties are retained, including when a rewrite adds a filter.

Certificates require schema coverage of the queried runtime typed namespace,
exact snapshot counts/extrema, a fresh durable commit identifier and a read-only
certificate transaction with no pending own writes. Any later write commit disables
proof fields until a quiescent rebuild. Freshness does not verify global schema
conformance. Point estimates and planner signals remain ranking/goal heuristics.

`frontier_order` accepts `astar` (default), `uniform` or `greedy` for practical
queue priority. It changes queue priority only; edit costs and the admission
cutoff remain fixed. Strict theorem mode forces its own A* contract.

In practical executed why-many search, the bounded key prepass reads one matching
valuation and chooses key constants from that same row. It then validates each
restriction with bounded COUNT, including the requested nonempty lower floor.
This read is reported separately as `witnessQueryCalls` and `witnessQueryTimeMs`.
Snapshot-only mode performs no such read; strict mode omits the prepass.

Malformed comparisons are rejected before search goal testing. Unsupported
formal-bound inputs also suppress the current-query impossibility flag. The
bounded rho-decrease check accepts threshold strengthening only: adding a key
equality can reduce a population by more than any fixed rho.

`formal_certificate_available` describes availability of the fresh metadata backend; it is not a certificate that a particular unsupported query meets its target. Use `upper_target_certified` and the mode-specific `target_certified`/`target_validation_required` fields for that decision. Snapshot representatives may contain a property value linked to an observed positive degree; these are used to choose constants, not to claim a general predicate-sensitive lower bound.
