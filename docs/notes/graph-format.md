# Graph Format

How the graph topology is represented on disk.

## Requirements

- Human-readable and human-editable.
- Machine-parseable — the engine reads the same files a developer reads.
- Progressive disclosure — a top-level view shows the graph structure; drill into nodes for detail.
- Supports edges with conditions, labels, and properties.
- Supports fan-out, fan-in, cycles (back-edges), and conditional branching.

## Options Under Consideration

### Option A: YAML frontmatter in node files

Each node is a markdown file. Edges are declared in the node's frontmatter as outbound connections.

```yaml
---
edges:
  - target: evaluator-security
    condition: always
  - target: evaluator-architecture
    condition: always
  - target: evaluator-coverage
    condition: always
fan: out
---
```

The graph is the sum of all node files. Topology is distributed — you reconstruct it by reading every node.

**Tradeoff:** Easy to edit one node at a time. Hard to see the full graph at a glance.

### Option B: Standalone graph file

A single file (e.g., `graph.md`) encodes the full topology. Nodes reference their detail files.

```
planner -> developer
developer -> evaluator-security, evaluator-architecture, evaluator-coverage [fan-out]
evaluator-security -> foreman [fan-in]
evaluator-architecture -> foreman [fan-in]
evaluator-coverage -> foreman [fan-in]
foreman -> planner [condition: findings]
foreman -> done [condition: pass]
```

**Tradeoff:** Full graph visible at a glance. Node detail lives elsewhere — two places to maintain.

### Option C: Hybrid

Topology in a graph file. Node content (system prompts, schemas, config) in individual node files. The graph file is the map; node files are the territory.

**Tradeoff:** Best of both — but the link between graph file and node files must be kept in sync.

## Edge Syntax

Edges are control structures. Five named types:

```
# Sequential — always fires
planner -> developer

# Gate — memo passes or doesn't
planner -> developer : when approved

# Ternary — one condition, second target is implicit else
foreman -> planner : when findings > 0 || complete

# Case — multi-way branch, one fires
classifier -> bug-fix : when type == "bug" 
           || feature : when type == "feature" 
           || refactor : when type == "refactor"

# Fan-out is not an edge type — it's multiple edges from one node
developer -> eval-security
developer -> eval-architecture
developer -> eval-coverage
```

Guards are simple predicates on memo fields (`when` clauses). Complex conditions requiring code or LLM judgment live in upstream nodes, not on edges.

## Open Questions

- Which notation for the graph file? Tables, DSL in a fenced block, indented outline, something else?
- Does the graph file reference node files by path, by name, or by convention (e.g., `nodes/{name}.md`)?
- Default edge behavior when no guard matches — halt? error edge? implicit fallthrough?
