# WhyGraph Neo4j Plugin

This module contains the active Neo4j plugin source for WhyGraph.


## Layout

- `src/main/java/com/whyquery/neo4j/`: main `why.*` procedures and rewriting logic
- `src/main/java/org/neo4j/whygraph/`: compatibility `whygraph.*` procedures
- `src/main/resources/`: Neo4j service registration and optional LLM config
- `src/test/java/`: unit and integration-oriented tests
- `scripts/build.sh`: canonical build entrypoint
- `scripts/install.sh`: canonical Neo4j Desktop install entrypoint
- `docs/examples.md`: compact runnable examples
- `user_guide.md`: detailed procedure and option reference

## Canonical Commands

Build:

```bash
./scripts/build.sh
```

Install into a Neo4j Desktop DBMS:

```bash
./scripts/install.sh "/path/to/dbms-xxxx"
```

Restart Neo4j after installation, then verify that the procedures are loaded:

```cypher
SHOW PROCEDURES YIELD name
WHERE name STARTS WITH 'why.'
RETURN name
ORDER BY name;
```

Run tests:

```bash
mvn test
```

## Notes

- The `why.*` procedures are the primary public API used by the artifact and evaluation harness.
- The `whygraph.*` procedures are kept as compatibility wrappers for earlier iterations of the plugin.
- Native/C++ scaffolding and generated local build artifacts have been removed from the active module layout; `scripts/` and `src/` are the canonical entry points.

## V2 Procedures (`why2.*`)

The plugin now also exposes a parallel v2 namespace designed to evolve the formal core without breaking v1:

- `why2.empty(query, options)`
- `why2.few(query, target, options)`
- `why2.many(query, target, options)`
- `why2.not(query, target, options)`

`why2.not` accepts either `witness` (single) or `witnesses` (list):

```cypher
CALL why2.not(
  "MATCH (p:NRPerson)-[:ERwasBornIn]->(c:NRCity) WHERE p.name='This_Person_Does_Not_Exist' RETURN p,c",
  {
    witnesses: [
      {p:{properties:{name:'Diederik_Korteweg'}}},
      {p:{properties:{name:'Ada_Lovelace'}}}
    ]
  },
  {strict_core:true, use_schema:true, use_planner_stats:true}
)
YIELD status, payload, diagnostics, errorMessage
RETURN status, payload['witness_recovery'], diagnostics, errorMessage;
```
