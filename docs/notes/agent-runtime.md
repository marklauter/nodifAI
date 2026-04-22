# Agent runtime

How agent and judgment nodes execute the LLM-plus-tools work that their memo-in/memo-out contract implies. Orthogonal to `runtime-semantics.md`, which covers the graph engine above this layer.

## Problem

An agent node wraps an LLM call with tool use. A planner-and-developer team needs to read files, write patches, run tests, and grep the codebase — the full software-engineering tool surface. nodifAI picks a runtime for that work inside each node. The choice drives cost, build effort, control, and language.

## Options

### Claude Client SDK

Anthropic's Messages API bindings. Send prompts, receive responses, implement the tool loop yourself. Official languages: Python, TypeScript, Go, Java, Ruby, PHP, and C# (beta).

- Upside: maximum control, minimum per-execution overhead, and widest language support.
- Downside: you build Read/Write/Edit/Bash and the agent loop from scratch. Substantial work.

### Claude Agent SDK

Claude Code's agent loop and tool infrastructure as a library. Built-in Read/Write/Edit/Bash/Glob/Grep/WebSearch/WebFetch, MCP support, hooks, subagents, and session management. Ships in Python and TypeScript.

- Upside: the tool stack is pre-built and Claude handles the tool loop autonomously. Fastest path to a working agent node.
- Downside: Python or TypeScript only.

### Claude Code CLI via subprocess

Orchestrate `claude -p "prompt"` invocations from the harness, one per agent node. `--bare` is the scripted-invocation mode: it skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md, so context is whatever the harness injects explicitly.

- Upside: zero runtime to build, and each node gets OS-process isolation for free — crash containment, memory isolation, and any container or seccomp wrapper the harness wants to apply.
- Downside: subprocess overhead per node, and the server-side prompt cache is keyed on byte-identical prefixes with a five-minute sliding TTL — across short-lived invocations with drifting prefixes, hit rates fall off.

## Tradeoffs

Cost per execution, low to high: Agent SDK, Client SDK with heavy prompt caching, Claude Code CLI. The Agent SDK wins because it exposes `cache_control` breakpoints as a first-class knob: a shared constitutional preamble stays cached across every plan, developer, and review invocation. CLI caching is per-session and opaque, so fan-out across short-lived role processes mostly misses.

Build effort, low to high: Claude Code CLI, Agent SDK, Client SDK.

Language support: Client SDK covers seven languages including C# (beta). Agent SDK is Python and TypeScript. CLI subprocess works from any language that can spawn processes.

Tool surface: CLI and Agent SDK share the full palette. Client SDK gives you nothing until you build it.

## Chosen path

Build the agent-node runtime on the Agent SDK, in Python. The graph-orchestration layer (node dispatch, memo routing, and topology) lives in nodifAI proper; the Agent SDK handles the inside of each agent node.

Two constraints drive the pick. First, Anthropic's February 2026 clarification requires API-key authentication for Agent SDK and unattended agentic workloads; nodifAI's multi-hour runs would hit the subscription's rolling rate cap mid-cycle and corrupt a run, so metered API is the only predictable billing model regardless of runtime. Second, a graph with a shared constitutional preamble across plan, developer, and review nodes pays that preamble's tokens on every node call; explicit `cache_control` breakpoints, which the Agent SDK exposes and the CLI does not, let the preamble stay cached across roles within the sliding TTL.

Mechanism and judgment nodes use whichever runtime is lightest. A mechanism node runs deterministic code with no LLM — no SDK needed. A judgment node can use the Client SDK directly for a single structured call without tools.

Claude Code stays as the dev environment for designing, testing, and iterating on nodes. Production execution runs on the Agent SDK. Anthropic's docs name this pattern: CLI for dev, SDK for production.

## Language implications

Python is the chosen implementation language. The Agent SDK constraint (Python or TypeScript only) drove the pick; TypeScript remains a fallback if a specific component doesn't suit Python.

C# would have forced one of:

- Shell out to a Python or TypeScript runtime for agent nodes.
- Use the Client SDK in C# and rebuild the tool loop, returning to the Client SDK option's build cost.

The Claude Code CLI subprocess option is language-agnostic — any language that can spawn processes works — and remains available if a single node ever benefits from full Claude Code capabilities over the Agent SDK's narrower surface.

## Open questions

- Whether mechanism and judgment nodes justify a separate runtime, or share the Agent SDK process to amortize startup.
- How to layer `cache_control` breakpoints — shared constitution, per-role charter, per-node tool schema — to maximize hit rate without starving node-local context of cache budget. Constraints: up to four breakpoints per request, a roughly twenty-block prefix lookback from each breakpoint, a minimum cacheable-prefix size (1024 tokens on Sonnet and Opus, 2048 on Haiku), and a five-minute sliding TTL by default with a one-hour beta tier at higher write cost. No documented cap on the number of distinct cached prefixes an organization can hold warm, so each role's constitution can coexist as its own cache entry — the question is which ones fire often enough inside the TTL to pay back their 1.25× write cost.

## References

- [Agent SDK overview](https://docs.anthropic.com/en/docs/claude-code/sdk) and [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) — Agent SDK requires API-key authentication; the CLI-for-dev, SDK-for-production pattern.
- [Best practices for Claude Code](https://www.anthropic.com/engineering/claude-code-best-practices) — `--bare` skips hooks, LSP, plugin sync, skill directory walks, and auto-memory, and requires `ANTHROPIC_API_KEY` or an `apiKeyHelper`.
- [Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) and [Pricing](https://docs.anthropic.com/en/docs/about-claude/pricing) — `cache_control: {type: "ephemeral"}` with `ttl` of `5m` (default) or `1h` (beta); writes bill at 1.25× base input, reads at 0.1× base input; TTL refreshes on each hit.
