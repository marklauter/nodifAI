# Hosting

Where the harness runs, where agent nodes run, and how to keep the mutable harness out of the agents' reach. Depends on `agent-runtime.md`, which picks the Python Agent SDK as the per-node runtime.

## Problem

An agent node edits source code. The harness also lives on disk. Running both on the same device under a single user account gives no OS-level guarantee that an agent's Write or Bash stays inside the workspace. Configuration — `cwd`, `allowed_tools`, permission hooks — enforces the boundary. Any config bug or prompt-driven pivot puts the harness in scope, and the same applies to the Agent SDK install and the `claude` CLI binary, which are user-writable files wherever pip or npm placed them.

Windows amplifies this. NTFS ACLs and separate user accounts do not give a developer workstation practical separation, and there is no native unprivileged process sandbox analogous to Linux namespaces. Docker Desktop on Windows is a WSL2 Linux VM under the hood, not a Windows primitive. The structural answer lives in Linux, not in Windows.

The goal is a structural boundary between harness and agent on a single device — not through nested containers, which is available but operationally heavier than needed, but through user separation inside a dev container: the harness installed as root, the agent running as a non-root dev user, unix permissions doing the work. That plus container-level blast-radius containment (cheap reset, host protection, reproducible runs) is the shape the Chosen path lands on.

## Options

### Dev container with harness as a devcontainer feature

A single Linux dev container (Ona, Gitpod, Codespaces, or a locally built devcontainer) holds everything: the harness, the Agent SDK, the CLI, and the agent's target repo. The harness ships as a devcontainer feature — `ghcr.io/<org>/nodifai:<version>` referenced from `devcontainer.json`. The feature's install script runs at image build time as root, landing the binary on PATH (`/usr/local/bin/nodifai`) and the Python package in root-owned site-packages. The agent's target repo mounts into the workspace at `/workspaces/<project>/`, and the harness is invoked from there the same way `claude code` is: the user (or the harness's own entry point) runs `nodifai ...` with the workspace as cwd. Agent nodes launched by the harness run as the same non-root dev user with `cwd` scoped to the workspace. The root-owned install is visible to agents but not writable by them without sudo, so an agent that escapes its `cwd` can read but cannot modify the harness.

### Per-node nested container

On top of a dev container, each node invocation spawns its own sub-container that mounts only the workspace. The harness dispatches via the container runtime rather than as an in-process SDK call. Structural separation between harness and node inside one physical host. Added operational complexity: docker-in-docker or a container-runtime socket, image management, and memo I/O plumbing. Warranted when agents handle untrusted input or the harness serves multiple tenants.

### Remote sandbox service

E2B, Modal, Daytona, Fly Machines. The service hosts the agent execution; the harness dispatches over an API. Strongest isolation, highest operational overhead. Appropriate for multi-tenant or public-facing deployments.

### Ephemeral cloud VM per run

Spin up a VM, run the team, tear down. Trivially isolated. Per-run provisioning overhead makes this heavier than containers; justifiable only when the workload needs a full VM for reasons other than isolation.

## Chosen path

Dev container as the baseline, with the harness shipped as a devcontainer feature that installs into root-owned image paths at build time. While the harness itself is under active development, the feature's install script can instead clone to `/opt/nodifai/` or `pip install` an editable build; the structural story is the same as long as the install is root-owned. The agent's target repo lives in the workspace at `/workspaces/<project>/`. Agent nodes run as the non-root dev user with tight `cwd` and `allowed_tools`.

Separation between harness and agent comes from unix permissions, not from the container boundary: the harness install is root-owned and read-only to the dev user, so an agent that escapes its `cwd` can read but cannot write it. The container itself provides blast-radius containment for everything else — agent mistakes inside the workspace roll back with `git`, anything worse rolls back with a container rebuild.

Two conditions have to hold for the permissions boundary to mean anything:

- The container runs as a non-root user. Codespaces and standard devcontainer base images set this by default; a custom image must declare `remoteUser` or `USER` explicitly, otherwise everything runs as root and the boundary collapses.
- Sudo is absent or narrowly scoped. Passwordless sudo for arbitrary commands lets an agent rewrite the harness in one step, so either omit sudo from the agent's runtime environment or restrict it to specific tools that do not include editors, package managers, or shells.

Per-node nested containers defer to a later phase. Add them when nodifAI runs for anyone other than the author, when agent inputs come from outside your control, or when the trust model tightens enough that unix permissions alone feel thin.

Windows is a dev and exploration surface only. WSL2 runs a local dev container for iteration. Any unattended or multi-hour run belongs in a real Linux environment — a hosted dev container like Ona, a Linux VM, or a server — not on the Windows host.

## Staging

While iterating on the harness, work inside a dev container locally (WSL2 Ubuntu with VS Code's `devcontainer.json`, or a hosted Ona workspace). Harness source at `/opt/nodifai/` or in a second workspace folder; target project at `/workspaces/<project>/`; agents scoped to the project. `git` is the rollback for agent mistakes in the workspace; `docker rebuild` is the rollback for harness mistakes.

When nodifAI is stable enough for unattended runs, the same dev container shape is the production unit — one container per run, torn down afterward. Per-node nesting or a sandbox service comes in only when the trust model changes.

## Open questions

- How the project checkout lifecycle spans a multi-node sequence. Plan reads the repo, developer mutates it, review reads the mutated version. Shared volume across nodes, per-node copy-on-write, or a memo-serialized diff applied between nodes.
- Whether the harness needs any footprint inside the container beyond the Agent SDK call itself — for structured memo I/O, logging, supervision. Each addition reintroduces harness mutability surface that the minimal model avoids.
- Whether a hosted dev-container platform (Ona, Codespaces) or self-managed Linux VMs is the better production substrate. Operational convenience versus control over image, network, and billing.
