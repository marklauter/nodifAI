# Memo contracts

How memos — the messages passed between nodes — are structured and validated.

## What is a memo

A memo is the unit of data that flows along an edge from one node to the next. It is the only way nodes communicate. Nodes do not share state — they share memos.

## Schema declaration

Each node declares its expected input and output memo shapes in frontmatter. This enables static validation: before running the graph, the engine can walk all edges and verify that every node's output schema is compatible with its downstream node's input schema.

```yaml
---
input:
  required: [outline, plan]
  optional: [prior_findings]
output:
  required: [plan, rationale]
---
```

## Open questions

### Accumulation vs. transformation

Two strategies for how memos evolve as they traverse the graph:

Accumulation — the memo grows. Each node adds fields but prior fields persist. The memo becomes a running record of everything that has happened.

- Pro: downstream nodes have full context.
- Con: memos get large and noisy. Nodes depend on fields they shouldn't. Tight coupling.

Transformation — each node receives an input memo and produces a clean output memo. Only declared output fields carry forward.

- Pro: explicit contracts. Nodes are composable and independent.
- Con: context that must persist across multiple hops needs explicit pass-through or a separate context mechanism.

Likely answer: transformation as the default. Accumulation fields (e.g., original outline and cycle count) are explicitly declared as pass-through in the schema rather than implicitly carried.

### Memo format

What format are memos in? Options:

- Structured (JSON/YAML) — machine-parseable and schema-validatable.
- Markdown with frontmatter — human-readable, structured metadata in YAML, and prose content in the body.
- Free text — maximum flexibility for LLM-to-LLM communication, minimum structure for mechanical nodes.

Mechanical nodes (mechanism, router, fan-out, and fan-in) need structured fields. Agent and judgment nodes produce natural language. The format needs to accommodate both — structured envelope with a prose body.

### Validation strictness

How strict is schema enforcement?

- Strict — missing required fields halt the graph.
- Warn — missing fields logged, execution continues.
- Per-edge — strictness is declared on the edge, not globally.
