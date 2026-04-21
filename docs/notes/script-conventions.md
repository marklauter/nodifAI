# Script Conventions

How shell scripts in this repo are written. Applies to anything under `.scripts/` or similar — helper tools, CI glue, local automation.

## Shell and Safety

- **`#!/usr/bin/env bash`** as the shebang. Finds bash on non-standard paths (macOS, nix, Windows Git Bash).
- **`set -euo pipefail`** at the top of every script. `-e` exits on error, `-u` errors on unset variables, `-o pipefail` catches failures mid-pipeline instead of silently swallowing them.
- **No `set -x` in committed scripts.** Gate verbose mode behind `[[ "${DEBUG:-}" ]] && set -x` so it's opt-in.

## Variables and Expansion

- **Always quote expansions** — `"$var"`, `"${array[@]}"`. Unquoted expansions word-split and glob, which is a classic source of silent corruption.
- **Default-substitute unset vars** — `"${2:-}"` instead of `"$2"` under `set -u`.
- **Required args fail fast** — `"${1:?error: title required}"` aborts with a readable message if the caller forgot it.
- **`[[ ... ]]` over `[ ... ]`** — no word-splitting on the left, supports `&&`, `||`, `=~`, and the unary tests.
- **`$(...)` over backticks** for command substitution — nestable and readable.

## Paths and Working Directory

- **Resolve the script's own directory** — `SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"`. Never assume the caller's cwd.
- **Use absolute paths internally.** Scripts that `cd` then run relative paths are fragile.
- **Never parse `ls`.** Use globs with `shopt -s nullglob` for empty-safe iteration, or `find -print0 | xargs -0`.

## Input, Output, Errors

- **Errors go to stderr** — `echo "error: ..." >&2`. stdout is for the script's result; stderr is for diagnostics. This lets callers pipe real output without swallowing error messages.
- **Non-zero exit on failure.** Use distinct codes if callers need to distinguish cases.
- **Uniform error prefixes** — `error:`, `warn:`. Cheap to grep for.
- **Read multiline input from stdin**, not from the command line. Avoids quoting hell. The caller uses a heredoc.

## Temp Files

- **`mktemp` for temp files and directories.** Never `/tmp/myscript.$$` — race-prone.
- **`trap cleanup EXIT`** to remove them. Set the trap immediately after creating the resource.
- **One trap, one cleanup function.** Multiple `trap` calls on the same signal overwrite each other.

## Arrays

- **Use arrays for argument lists**, not space-joined strings. `LABEL_FLAGS+=(--label "$x")` then `"${LABEL_FLAGS[@]}"` handles spaces and empty elements correctly.
- **Check element count with `${#arr[@]}`.** Test the first element exists with `[[ -f "${arr[0]:-}" ]]`.

## Structure

- **Header comment** with purpose, usage, and an example invocation. First thing a reader needs.
- **Tiny functions over long scripts.** A single-purpose function named `create_issue` makes retry trivial and tests readable.
- **Declare `local` inside functions** so variables don't leak into the caller's scope.
- **Sectional dividers** — `# --- config ---`, `# --- labels ---` — when the script has phases. Cheap structure, no tooling required.

## External Tools and Robustness

- **Check prerequisites early** — `command -v gh >/dev/null || { echo "error: gh not installed" >&2; exit 1; }`. Fail before side effects, not halfway through.
- **Validate inputs before side effects.** Don't half-apply a change and bail.
- **Retry idempotent network calls once or twice** with the underlying error visible if all attempts fail. Bounded retries only — no sleep loops.
- **`jq` over `grep | sed | awk`** for JSON. **`yq` over improvised YAML parsers** — ad-hoc `grep '^key:' | sed ...` works for simple files but breaks on anything with quoting, nesting, or multiline values.

## Anti-patterns

- **No `curl | bash`** in scripts we ship. Download, inspect, then run.
- **No silent `|| true`** to swallow errors. If you genuinely don't care, comment why.
- **No `eval`** on anything that touches user input.
- **Don't parse filenames with newlines implicitly.** If it could happen, use `-print0` / `-0`.

## Linting

Run ShellCheck on every script. It catches most of the above automatically. Disable a specific rule with `# shellcheck disable=SCxxxx` and a one-line reason when the deviation is deliberate.
