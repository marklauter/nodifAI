# Node model

The types of nodes in the graph and what defines each one.

## Common structure

Every node is a markdown file with YAML frontmatter. The frontmatter declares the node's type, input/output memo schemas, and configuration. The markdown body contains the node's content (system prompt, evaluation criteria, code reference, etc.).

```yaml
---
type: agent | judgment | mechanism | router | fan-out | fan-in
input: <memo schema>
output: <memo schema>
---
```

## Node types

### Agent

An LLM-powered node. The markdown body is the system prompt (the agent's role definition). The model receives the inbound memo as the user message and produces an output memo.

- Config: model, temperature, tool access, and max turns.
- Body: system prompt / role definition.
- Input: memo from upstream node.
- Output: structured memo matching the declared output schema.

### Judgment

An LLM evaluates a condition and produces a verdict. Similar to an agent node but the output is a decision, not work product. Outbound edges are labeled with possible outcomes.

- Body: evaluation prompt and criteria.
- Input: memo containing the material to evaluate.
- Output: verdict (maps to an outbound edge label).

### Mechanism

Deterministic code execution. No LLM involved. Runs a script, command, or function.

- Body: description of what the mechanism does (for human readers).
- Config: script path, command, or inline code reference.
- Input: memo fields the mechanism reads.
- Output: memo fields the mechanism writes.

Examples: run build, run tests, compute coverage delta, hash protected files, and validate schema.

### Router

Conditional branching based on memo field values. No LLM, no code execution — pure routing logic.

- Config: condition expressions mapped to outbound edge labels.
- Input: memo to inspect.
- Output: same memo, routed to the matching edge.

## Fan-out and fan-in

Fan-out and fan-in are not node types — they are properties of graph topology (multiple outbound or inbound edges). See runtime-semantics.md for activation strategy (open question: where this is declared).
