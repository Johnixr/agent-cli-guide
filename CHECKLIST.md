# Agent CLI Design Checklist

A quick-reference checklist for building agent-friendly CLI tools. See [GUIDE.md](GUIDE.md) for detailed rationale and examples.

## Command Structure
- [ ] Commands use noun-verb hierarchy (`mytool user create`, not `mytool create-user`)
- [ ] Same verbs are reused across resource types (`list`, `get`, `create`, `delete`, `update`)
- [ ] No catch-all subcommands or prefix abbreviation

## Flags & Parameters
- [ ] Every flag has a long form (`--verbose`); short form (`-v`) is optional
- [ ] Constrained parameters use enums: `--format json|table|csv`
- [ ] No passwords accepted via flags — use `--password-file` or stdin
- [ ] GNU standard names followed: `--help`, `--version`, `--verbose`, `--quiet`, `--output`, `--force`

## Output
- [ ] `--format json` supported on every command that produces output
- [ ] JSON goes to stdout; progress/warnings/logs go to stderr
- [ ] Adding new optional JSON fields is non-breaking; removing/renaming fields is breaking
- [ ] Pagination support: `--page-size`, `--page-all`, or NDJSON streaming

## TTY Awareness
- [ ] Non-TTY mode: JSON output, no colors, no prompts, no pagers
- [ ] `--yes` / `--no-interactive` skips ALL confirmation prompts
- [ ] `NO_COLOR` and `TERM=dumb` environment variables respected
- [ ] Missing required input in non-TTY mode fails explicitly (never hangs)

## Safety & Dry-Run
- [ ] All side-effect commands support `--dry-run`
- [ ] `--dry-run` output is structured JSON (not just a text message)
- [ ] Destructive commands require `--force` in TTY mode

## Exit Codes
- [ ] Documented exit codes beyond 0/1: usage error (2), not found (3), permission denied (4), conflict (5)
- [ ] Dry-run success uses a dedicated exit code (e.g., 10)
- [ ] Exit code semantics are stable across versions

## Input Validation
- [ ] URLs validated (reject `javascript:`, `file:`, embedded credentials)
- [ ] File paths validated (reject traversal into `.ssh/`, `.gnupg/`, etc.)
- [ ] Schema introspection available: `mytool schema [command]`

## Idempotency
- [ ] Create operations support `--if-not-exists`
- [ ] Conflict returns exit code 5 (not generic error)
- [ ] Idempotency keys supported for non-idempotent operations (`--idempotency-key`)

## Error Messages
- [ ] Errors include: machine-readable type, description, recovery suggestion, retryable flag
- [ ] Non-TTY errors output as JSON to stderr
- [ ] Failing input is echoed in the error

## Help Text
- [ ] `--help` leads with 2-3 realistic usage examples
- [ ] Required vs. optional flags clearly marked
- [ ] Value domains shown: `--format json|table|csv`
- [ ] Help text under 50 lines per command
- [ ] Responds to `-h`, `--help`, and `help <subcommand>`
