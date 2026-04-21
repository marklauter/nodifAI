# Runtime semantics

How the graph engine walks the graph.

## Push-based flow

The harness drives all flow. Nodes do not pull work — the harness pushes memos into downstream nodes. On completion, every node (agent or mechanical) produces a memo. Nodes also produce artifacts (files, diffs, patches, etc.) which are referenced within the memo, not carried inline.

## Graph shape

The graph is a pipeline, analogous to a UML activity diagram: one start/initial node, one final node. However, unlike an activity diagram, fan-out does not require a corresponding join. A fan-out node is more like a message bus — it can feed parallel, unrelated activities that may terminate independently.

## Activation

A node activates when the harness pushes a memo into it. For nodes expecting input from multiple sources (join/fan-in), activation requires all (or a quorum of) inbound memos before the node fires.

## Edges as control structures

Edges are the graph's control structures. A node's outbound edge declaration is a routing expression with 1..N targets and optional guards. Guards are simple predicates on memo fields — not processing. If a condition requires judgment or code, it belongs in an upstream node.

## Edge taxonomy

Four edge types:

- Sequential — 1 target, no guards.
- Gate — 1 target, 1 guard.
- Ternary — 2 targets, 1 guard (plus implicit else).
- Switch — 2+ targets, 1 guard per target.

```
# Sequential
planner -> developer

# Gate — memo passes through or doesn't
planner -> developer : when approved

# Ternary — one condition, two targets, second is implicit else
foreman -> planner : when findings > 0 || complete

# Switch — multi-way branch, one target fires
classifier -> bug-fix : when type == "bug" 
           || feature : when type == "feature" 
           || refactor : when type == "refactor"
```

Fan-out is not an edge type. Fan-out is a property of a node having multiple outbound edges. Each edge in the collection is independently typed — sequential, gate, ternary, or switch.

```
# Fan-out: three sequential edges from one node
developer -> eval-security
developer -> eval-architecture
developer -> eval-coverage

# Fan-out with mixed edge types
developer -> eval-security
developer -> eval-architecture : when lang == "csharp"
developer -> eval-coverage : when tests.exist || skip-coverage
```

Guards are simple predicates on memo fields. Complex conditions requiring code or LLM judgment belong in an upstream node, not on an edge.

A gate that doesn't open is a halt — a controlled stop, not an error. The graph author placed it intentionally.

## Fan-out

A fan-out node publishes memos to multiple downstream nodes. Branches execute independently with no shared state between them. There is no requirement that branches reconverge — they may terminate independently, like subscribers on a message bus.

## Fan-in and activation

Fan-in, like fan-out, is not a special node type. It's a property of a node having multiple inbound edges.

Two activation strategies are needed:

- `each` — message bus pattern. The node fires once per arriving memo and manages its own accumulation.
- `all` — `Task.WaitAll` pattern. The engine collects all inbound memos and delivers them as a batch. The node fires once with the full collection.

Open question: where does the activation strategy live? It's not a property of the node (the node just processes memos), and it's not a property of the edges (they just deliver). It's an orchestration concern — the harness deciding *when* to fire a node at a convergence point. Could be harness-level default, graph-level declaration, or per-convergence annotation. TBD.

## Cycles

Cycles are emergent, not a built-in concept. A cycle is just an edge whose target is upstream in the graph. It uses the same edge taxonomy — a ternary or switch edge pointing backward will naturally terminate when the guard routes to the non-looping target. The engine just follows edges; it doesn't need to know it's in a cycle.

A sequential edge pointing backward would be an infinite loop — either a graph design bug or requires an external kill (max iterations, timeout).

## Failure

Every node returns a memo. Failure is a value, not an exception. A node doesn't throw — it returns a memo with a result context. The engine doesn't need try/catch, retry logic, or special failure modes. It just pushes memos and follows edges.

Two distinct levels of "failure":

1. Node result — the node ran successfully but reports a problem in the work product. "I ran the build and it failed." The node worked; the code is broken.
2. Node failure — the node itself couldn't complete. "I couldn't run the build — tool crashed, disk full, or credentials expired."

Both are values in the memo. Both route via edges. The graph author handles them with the edge taxonomy:

```
checks -> developer : when result == "fail"    # work product broken, cycle back
      || eval-team : when result == "pass"      # work product good, proceed
      || operator : when result == "error"       # node itself broke, escalate to human
```

An unhandled result (no edge matches) is a gate that didn't open — a halt.

## State

The engine itself is stateless. All state travels in the memo. The engine's only job is to route memos along edges, activate nodes, and enforce the graph topology. Harness-held invariants (checkpoints, snapshots) live outside the graph — they are not memo state, they are enforcement infrastructure.

## Resolved

- Multiple terminal nodes are allowed. Fan-out branches can terminate independently.
