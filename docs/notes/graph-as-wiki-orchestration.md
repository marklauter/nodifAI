# Graph-Based Agent Orchestration

## Core Concept

A multi-agent orchestration system where the graph — agent roles, message contracts, inter-agent prompts, conditions, fan-out patterns — is defined as markdown files with structured metadata. The runtime is a graph navigator that passes memos from node to node.

## Graph Representation

The graph topology can be represented in several ways, still under consideration:

- **YAML frontmatter in node files** — each node is a markdown file with YAML metadata defining edges, types, and memo schemas (similar to Anthropic skills patterns).
- **A standalone property graph in markdown** — a single file (or small set of files) encoding the full graph structure using a markdown-native notation for nodes, edges, and properties.
- **Hybrid** — node content lives in individual markdown files, graph topology lives in a dedicated property graph file.

Several markdown notations work well for property graphs (tables, definition lists, indented outlines, fenced blocks with lightweight graph DSLs). The right choice depends on what's most natural to read and edit.

## Design Principles

- **Markdown is the substrate.** Node definitions, graph topology, and agent prompts all live in markdown files, possibly with YAML metadata.
- **Progressive disclosure.** A top-level view shows the graph structure. Drill into individual node files for system prompts, tool access, output schemas.
- **Memos are the interface contract.** Each edge carries a typed memo. The receiving node declares what it expects. Mismatches are catchable statically by walking the graph before execution.
- **Human-readable, machine-executable.** The same files a developer reads to understand the system are the files the runtime reads to run it.

## Node Types

### Agent Nodes
System prompt and model config in the markdown body. Frontmatter defines input/output memo schemas. The node *is* the agent's role definition.

### Judgment Nodes
An LLM evaluates a condition. Outbound edges represent possible outcomes. The node body contains the evaluation prompt and criteria.

### Mechanism Nodes
Deterministic code execution — validation, transformation, API calls. Code is attached to the node (inline, co-located script, or referenced module).

### Router Nodes
Conditional branching based on memo fields. Edges are labeled with the conditions under which they fire.

### Fan-out / Fan-in Nodes
Spawn parallel branches across a collection or set of concerns. A corresponding fan-in node collects and merges results.

## Memo Flow

Memos are the messages passed between nodes. Open questions:

- **Accumulation vs. transformation:** Does the memo grow as it passes through the graph (shared state), or does each node produce a clean output memo? Accumulation is flexible but gets noisy. Clean transforms are composable but require explicit memory nodes for context that must persist.
- **Schema:** Frontmatter on each node can declare expected input/output memo shapes. This enables static validation of the graph before runtime.

## Execution Semantics

- **Cycles:** Supported via back-edges. Termination conditions live in the node or on the edge.
- **Fan-out:** A node can emit multiple memos to multiple targets in parallel.
- **Fan-in:** A collector node waits for all (or a quorum of) upstream branches before continuing.
- **Deterministic code:** Mechanism nodes can run arbitrary code at graph traversal time.

## Observability

Every node crossing is a discrete event with the memo state at that point. Tracing comes nearly for free — the graph structure *is* the trace topology.

## Influences

- **LangGraph** — stateful graph-based agent orchestration with cycles, shared state, checkpointing.
- **Basic Memory** — local-first markdown knowledge graph, entity-observation-relation model, MCP integration.
- **Anthropic skills** — markdown + YAML frontmatter as a pattern for defining agent capabilities.

## Status

Early concept. Key open questions: graph representation format, property graph notation, how code attaches to mechanism nodes, memo accumulation strategy.
