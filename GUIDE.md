# Agent CLI Design Guide

10 principles for building CLI tools that AI agents can use reliably.

These principles apply whether you're building a new CLI from scratch or adapting an existing one for agent consumption. They are ordered by architectural impact — start from the top.

---

## Principle 1: Noun-Verb Command Structure

Use `noun verb` hierarchy, not `verb-noun` or flat commands.

```bash
# Good: noun-verb (tree-searchable)
mytool user create
mytool user list
mytool project deploy
mytool billing invoice list

# Bad: verb-noun (flat, undiscoverable)
mytool create-user
mytool list-users
mytool deploy-project
```

**Why:** Agents discover commands through tree search. They run `mytool --help` → see resource nouns → run `mytool user --help` → see available verbs. This is a deterministic, layer-by-layer narrowing process.

Flat command lists force agents to scan a large unstructured list with no hierarchy to guide exploration.

**Rules:**
- Top-level commands are nouns (resources): `user`, `project`, `billing`
- Second-level commands are verbs (actions): `create`, `list`, `delete`, `update`
- Use the same verbs across different nouns for consistency (`list`, `get`, `create`, `delete`, `update`)
- Avoid near-synonyms that create confusion (don't have both `update` and `upgrade`)
- Don't implement catch-all subcommands — they block adding new commands later
- Don't allow prefix abbreviation (`mytool u` for `mytool user`) — it blocks future commands starting with `u`

---

## Principle 2: Long Flags First

Every flag must have a long form (`--verbose`). Short forms (`-v`) are optional human conveniences.

```bash
# Good: self-describing
mytool deploy --environment production --dry-run --format json

# Risky for agents: ambiguous short flags
mytool deploy -e production -n -f json
```

**Why:** Long flags are self-describing. The semantic binding between `--dry-run` and "preview without executing" is reinforced across billions of training samples. Short flags like `-n` could mean `--dry-run`, `--line-number`, `--numeric`, or `--name` depending on the tool.

Case-sensitive short flags are especially dangerous:

| Flag | Meaning |
|------|---------|
| `grep -i` | Ignore case |
| `grep -I` | Ignore binary files |
| `ssh -v` | Verbose |
| `ssh -V` | Version |
| `tar -c` | Create archive |
| `tar -C` | Change directory |

**Rules:**
- All flags have a long form; short form is optional
- Long flags use lowercase words joined by hyphens: `--dry-run`, `--no-interactive`
- Boolean negation uses `--no-` prefix: `--color` / `--no-color`
- Never accept passwords via flags (visible in process list and shell history) — use `--password-file` or stdin
- Follow GNU standard names where applicable: `--help`, `--version`, `--verbose`, `--quiet`, `--output`, `--force`, `--recursive`, `--all`

---

## Principle 3: Structured Output as API Contract

JSON to stdout, everything else to stderr. Treat structured output as a versioned API.

```bash
# stdout: structured data only
mytool user list --format json

# stderr: progress, warnings, logs (never mixed into stdout)
```

**Why:** Agents need to pipe and parse output reliably. Any non-JSON content in stdout (spinners, progress bars, color codes, warning text) will break downstream parsing.

**Rules:**
- Support `--format json` on every command that produces output
- Default to JSON when stdout is not a TTY (see Principle 4)
- Support additional formats as needed: `table` (human), `csv`, `ndjson` (streaming)
- stdout = data, stderr = messages — no exceptions
- Adding a new optional field to JSON output is safe (non-breaking)
- Removing or renaming a field is a breaking change — requires major version bump
- Include pagination support: `--page-size`, `--page-all`, or use NDJSON for streaming
- Flat JSON structures are preferred over deeply nested ones
- Use consistent field names and types across all commands

---

## Principle 4: TTY-Aware Behavior

Detect whether the CLI is running in a terminal (TTY) or a pipe, and adjust behavior automatically.

```bash
# Terminal (TTY): colors, tables, interactive prompts, progress bars
mytool status

# Pipe (non-TTY): JSON, no colors, no prompts, no pagination
mytool status | jq '.'
```

**Why:** Agents almost always invoke CLIs in non-TTY contexts. Interactive prompts, pagers, and ANSI color codes will hang or corrupt agent workflows.

The AWS CLI v2 pager incident is a canonical example: changing the default pager to `less` broke thousands of CI jobs worldwide because automated consumers couldn't respond to the interactive pager.

**Rules:**
- In non-TTY mode: default to JSON output, disable colors, disable interactive prompts, disable pagers
- Provide explicit override flags: `--no-interactive`, `--no-color`, `--yes` (skip confirmations)
- Respect standard environment variables: `NO_COLOR`, `TERM=dumb`
- The `--yes` flag should skip ALL confirmation prompts, not just some
- If input is required but no TTY is available and no flag provides the value, fail explicitly with a clear error — never hang waiting for input

---

## Principle 5: Dry-Run by Default for Side Effects

Every command that creates, modifies, or deletes resources must support `--dry-run`.

```bash
# Preview what would happen
mytool user delete --name john --dry-run

# Output: structured diff of what would change
{
  "action": "delete",
  "resource": "user",
  "target": {"name": "john", "id": "u_123"},
  "reversible": false
}
```

**Why:** Dry-run provides agents with a zero-cost exploration-verification loop. Instead of gambling on the first attempt, the agent can preview consequences, verify correctness, then commit.

**Rules:**
- `--dry-run` output must be structured (JSON), not just a text message saying "this is a dry run"
- Show exactly what would be created, modified, or deleted
- Use a dedicated exit code for dry-run success (e.g., exit 10) to distinguish from actual execution success
- Destructive commands (`delete`, `drop`, `reset`) should additionally require `--force` or confirmation input when in TTY mode

---

## Principle 6: Semantic Exit Codes

Define and document exit codes beyond just 0 and 1.

| Exit Code | Meaning | Agent Response |
|-----------|---------|----------------|
| 0 | Success | Continue pipeline |
| 1 | General error | Read stderr for diagnosis |
| 2 | Invalid arguments / usage error | Fix arguments and retry |
| 3 | Resource not found | Skip or create |
| 4 | Permission denied | Prompt user for authorization |
| 5 | Conflict / already exists | Skip or update |
| 10 | Dry-run passed | Safe to execute for real |

**Why:** Exit codes are the first signal an agent sees after command execution. They determine control flow: retry, skip, escalate, or continue. With only 0/1, the agent must parse stderr text to understand what happened.

**Rules:**
- Document all exit codes in `--help` and in project documentation
- Exit codes must be stable across versions — changing semantics is a breaking change
- Distinguish transient errors (worth retrying: network timeout, rate limit) from permanent errors (wrong arguments, permission denied)
- Use exit code 2 for usage errors (POSIX convention)

---

## Principle 7: Input Validation and Hallucination Defense

Validate all inputs strictly. Don't trust that the caller — human or agent — knows what they're doing.

**Why:** LLMs can hallucinate non-existent flags, fabricate parameter values, or construct dangerous inputs. Input validation is the last line of defense, just like SQL injection prevention in web applications. This matters regardless of how capable the calling model is — your CLI will be used by models of varying quality.

**Rules:**
- Use enums for constrained parameters: `--format json|table|csv` not `--format <string>`
- Validate URLs (reject `javascript:`, `file:` protocols, embedded credentials)
- Validate file paths (reject traversal into sensitive directories like `.ssh/`, `.gnupg/`)
- Validate domains (reject path separators, shell metacharacters)
- Provide a schema introspection command:
  ```bash
  mytool schema --all           # Full command tree as JSON
  mytool schema user create     # Single command's parameter definitions
  ```
- Schema introspection should return: command name, description, flags with types/defaults/enums, required vs. optional, and examples
- Let agents query schema on-demand rather than injecting all schemas into context upfront

---

## Principle 8: Idempotent Operations

Design commands to be safely repeatable. Prefer declarative over imperative.

```bash
# Imperative: fails if resource already exists
mytool user create --name "john"         # Error on second run

# Declarative: same result regardless of how many times it runs
mytool user ensure --name "john"         # Success every time
# or
mytool user create --name "john" --if-not-exists
```

**Why:** Agents retry. Network timeouts, ambiguous results, interrupted tasks — all lead to repeated execution. Non-idempotent commands cause duplicate resources, duplicate messages, or duplicate charges.

**Rules:**
- Provide `--if-not-exists` or `--if-exists` flags for create/delete operations
- Use declarative verbs where possible: `ensure`, `apply`, `sync`
- Return exit code 5 (conflict) when a create operation finds the resource already exists, rather than a generic error
- Support idempotency keys (`--idempotency-key <uuid>`) for operations that can't be naturally idempotent (e.g., sending messages)
- `kubectl apply` is the gold standard: define desired state, let the system reconcile

---

## Principle 9: Actionable Error Messages

Errors must be machine-readable, specific, and include recovery instructions.

```json
{
  "error": "permission_denied",
  "message": "Missing calendar:read scope",
  "suggestion": "Run: mytool auth login --scope calendar:read",
  "retryable": false
}
```

**Why:** When an agent hits an error, the error message is its only source of information for recovery. Vague errors ("something went wrong") leave the agent stuck or looping.

**Rules:**
- Include a machine-readable error type/code (e.g., `"permission_denied"`, `"rate_limited"`, `"not_found"`)
- Include a human-readable description of what went wrong
- Include a specific suggestion for how to fix it (ideally a command the agent can run)
- Include a `retryable` boolean — network timeouts are retryable, permission errors are not
- Echo the failing input in the error so the agent can see what it sent wrong
- In non-TTY mode, output errors as JSON to stderr

---

## Principle 10: Help Text is the Agent's Brain

`--help` quality directly determines how accurately an agent will use your CLI.

Anthropic's tool use documentation states: **"Descriptions are by far the most important factor"** in tool use accuracy. They dramatically improved accuracy on benchmarks simply by optimizing tool descriptions. This maps directly to CLI `--help` text.

**Rules:**
- Lead with 2-3 realistic usage examples — agents (and humans) look at examples first
- Clearly mark required vs. optional flags: `--id <required>`, `--format <optional, default: json>`
- Show the value domain for constrained flags: `--format json|table|csv`
- List subcommands with one-line descriptions when showing group help
- Keep help text concise — aim for under 50 lines per command. Overly long help text can reduce agent accuracy by diluting the signal
- Respond to `-h`, `--help`, and `help <subcommand>`
- Include the `--json` flag explicitly in help text so agents discover it

---

## References

- [POSIX Utility Conventions](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html)
- [GNU Coding Standards — CLI](https://www.gnu.org/prep/standards/html_node/Command_002dLine-Interfaces.html)
- [Command Line Interface Guidelines (clig.dev)](https://clig.dev/)
- [12 Factor CLI Apps](https://medium.com/@jdxcode/12-factor-cli-apps-dd3c227a0e46)
- [Anthropic — Define Tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Anthropic — Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Lightning Labs — 10 Agent-CLI Design Axes](https://github.com/lightninglabs/lnget/pull/14)
- [InfoQ — Patterns for AI Agent Driven CLIs](https://www.infoq.com/articles/ai-agent-cli/)
- [Berkeley Function Calling Leaderboard V4](https://gorilla.cs.berkeley.edu/leaderboard.html)
