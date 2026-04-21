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

Orchestrate `claude -p "prompt"` invocations from the harness, one per agent node. Inherits every capability Claude Code ships — tools, skills, hooks, MCP, and permissions.

- Upside: zero runtime to build.
- Downside: subprocess overhead per node, less control over context and caching, and Claude Code's general-purpose system prompt comes along whether you want it or not.

## Tradeoffs

Cost per execution, low to high: Agent SDK, Client SDK with heavy prompt caching, Claude Code CLI.

Build effort, low to high: Claude Code CLI, Agent SDK, Client SDK.

Language support: Client SDK covers seven languages including C# (beta). Agent SDK is Python and TypeScript. CLI subprocess works from any language that can spawn processes.

Tool surface: CLI and Agent SDK share the full palette. Client SDK gives you nothing until you build it.

## Chosen path

Build the agent-node runtime on the Agent SDK, in Python. The graph-orchestration layer (node dispatch, memo routing, and topology) lives in nodifAI proper; the Agent SDK handles the inside of each agent node.

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
- Whether prompt caching warrants a fixed per-node system prompt that Anthropic can cache across invocations, or allows node-local context to dominate.
