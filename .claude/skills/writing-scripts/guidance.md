# Script conventions

How shell scripts in this repo are written. Applies to anything under `.scripts/` or similar — helper tools, CI glue, and local automation.

## Shell and safety

- `#!/usr/bin/env bash` as the shebang. Finds bash on non-standard paths (macOS, nix, and Windows Git Bash). Plain `#!/bin/bash` fails on macOS, where bash lives elsewhere.
- `set -euo pipefail` at the top of every script. `-e` exits on error, `-u` errors on unset variables, and `-o pipefail` catches failures mid-pipeline instead of silently swallowing them.
- `IFS=$'\n\t'` alongside `set -euo pipefail`. Strips space-splitting from the default IFS so iterating over file lists and command output stops misbehaving when filenames contain spaces.
- No `set -x` in committed scripts. Gate verbose mode behind `[[ "${DEBUG:-}" ]] && set -x` so it's opt-in.

## Variables and expansion

- Always quote expansions — `"$var"`, `"${array[@]}"`. Unquoted expansions word-split and glob.
- Default-substitute unset vars — `"${2:-}"` instead of `"$2"` under `set -u`.
- Required args fail fast — `"${1:?error: title required}"` aborts with a readable message if the caller forgot it.
- `readonly CONST=value` for values that should not change after first assignment.
- `[[ ... ]]` over `[ ... ]` — no word-splitting on the left, supports `&&`, `||`, `=~`, and the unary tests.
- `$(...)` over backticks for command substitution — nestable and readable.

## Arguments

Argument complexity drives tool choice.

- One positional arg: read `$1` directly.
- Two: `$1` and `$2`.
- More than two, or any flags: use `getopts`.
- Beyond what `getopts` handles cleanly (subcommands, long flags, or optional values): reach for Python.

```bash
while getopts "c:v" flag; do
  case "$flag" in
    c) CONFIG=$OPTARG ;;
    v) VERBOSE=1 ;;
    *) echo "usage: $0 [-c config] [-v]" >&2; exit 2 ;;
  esac
done
shift $((OPTIND-1))
```

## Paths and working directory

- Resolve the script's own directory — `SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"`. Never assume the caller's cwd.
- Use absolute paths internally. Scripts that `cd` then run relative paths are fragile.
- Forward slashes in every path, even on Windows under Git Bash. Never hard-code `\`.
- Never parse `ls`. Use globs with `shopt -s nullglob` for empty-safe iteration, or `find -print0 | xargs -0`.

## Input, output, errors

- stdout carries the script's result, nothing else. Logs, progress, and errors go to stderr — `echo "error: ..." >&2`. A caller should be able to `$(script.sh)` and get only the payload.
- Uniform error prefixes — `error:` and `warn:`. Cheap to grep for.
- Non-zero exit on failure. Use `1` for generic failure, `2` for misuse or bad arguments. Reserve distinct codes for cases callers need to branch on; beyond that, `man sysexits` defines 64–78 by convention.
- Read multiline input from stdin via heredoc, not from the command line.
- Quote the heredoc delimiter — `<<'EOF'` — to prevent `$var` and backtick expansion in the body. Use `<<<` for single-line input, and `<<-EOF` when leading tabs should be stripped.

## Temp files

- `mktemp` for temp files and directories. Never `/tmp/myscript.$$` — race-prone.
- `trap cleanup EXIT INT TERM` to remove them. EXIT alone misses Ctrl-C and `kill`. Set the trap immediately after creating the resource.
- One trap, one cleanup function. Multiple `trap` calls on the same signal overwrite each other.

## Arrays

- Use arrays for argument lists, not space-joined strings. `LABEL_FLAGS+=(--label "$x")` then `"${LABEL_FLAGS[@]}"` handles spaces and empty elements correctly.
- Check element count with `${#arr[@]}`. Test the first element exists with `[[ -f "${arr[0]:-}" ]]`.

## Structure

- Header comment with purpose, usage, example invocation, and a `test:` line.
- Tiny functions over long scripts. A single-purpose function named `create_issue` makes retry trivial and tests readable.
- Declare `local` inside functions so variables don't leak into the caller's scope.
- Sectional dividers — `# --- config ---`, `# --- labels ---` — when the script has phases.
- Prefer `while read ... done < <(cmd)` over `cmd | while read ...`. The pipe form runs the loop body in a subshell, so variable assignments inside the loop die when the loop exits.

## External tools and robustness

- Declare every external dependency at the top with a helper that fails fast:

```bash
require_cmd() {
  command -v "$1" >/dev/null || { echo "error: $1 required" >&2; exit 1; }
}
require_cmd gh jq yq
```

- Validate inputs before side effects.
- Bound every subprocess call that can hang. Wrap network calls and external-tool invocations with `timeout`:

```bash
timeout 30 curl --fail "$url"
```

- Retry idempotent network calls once or twice with the underlying error visible if all attempts fail. Bounded retries only — no sleep loops.
- `jq` over `grep | sed | awk` for JSON. `yq` over improvised YAML parsers; ad-hoc grep/sed breaks on quoting, nesting, or multiline values.

## Cross-platform

The development machine is Windows; the deployment target is Linux. Scripts run on both.

- Target GNU-flavored tool behavior. `sed`, `date`, `readlink`, and `stat` have different flags on BSD (macOS) and GNU (Linux). Git Bash on Windows ships GNU-flavored tools via MSYS, which happens to match Linux.
- Check for portable alternatives when reaching for a GNU-only flag. `sed -i` works on GNU; on BSD it needs an empty backup suffix. Safer pattern: write to a temp file and `mv` over the original.
- Scripts use LF line endings. `.gitattributes` should enforce this. Git already warns on add when CRLF would be converted.

## Anti-patterns

- No `curl | bash` in scripts we ship. Download, inspect, and then run.
- No silent `|| true` to swallow errors. If you genuinely don't care, comment why.
- No `eval` on anything that touches user input.
- No implicit parsing of filenames with newlines. If it could happen, use `-print0` / `-0`.

## Verification

- The header comment includes a `test:` line that exercises the golden path. Run it after any edit; check exit code and output.
- Destructive or network-calling scripts ship with a `--dry-run` flag that logs what would happen without doing it.
- Non-trivial logic gets a companion `.bats` file next to the script — `file-issue.sh` and `file-issue.bats`.
- Every script ships tested.

## Linting

Run ShellCheck on every script. It catches most of the above automatically. Disable a specific rule with `# shellcheck disable=SCxxxx` and a one-line reason when the deviation is deliberate.

## Template

```bash
#!/usr/bin/env bash
# Short description of what this script does.
#
# Usage:   .scripts/my-script.sh <arg> [-n]
# Example: .scripts/my-script.sh widget-42 -n
# Test:    .scripts/my-script.sh --dry-run example

set -euo pipefail
IFS=$'\n\t'

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly SCRIPT_DIR

require_cmd() {
  command -v "$1" >/dev/null || { echo "error: $1 required" >&2; exit 1; }
}
require_cmd gh jq

# --- args ---
DRY_RUN=0
while getopts "n" flag; do
  case "$flag" in
    n) DRY_RUN=1 ;;
    *) echo "usage: $0 [-n] <arg>" >&2; exit 2 ;;
  esac
done
shift $((OPTIND-1))

ARG="${1:?error: arg required}"

# --- cleanup ---
TMP=$(mktemp)
cleanup() { rm -f "$TMP"; }
trap cleanup EXIT INT TERM

# --- main ---
main() {
  echo "processing $ARG" >&2
  if [[ $DRY_RUN -eq 1 ]]; then
    echo "would process $ARG (dry run)" >&2
    return 0
  fi
  # real work
  echo "ok"
}

main "$@"
```
