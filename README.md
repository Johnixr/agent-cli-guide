# Agent CLI Guide

Design guidelines for building CLI tools that AI agents can use reliably.

## Why

CLI is becoming the default interface for AI agents to operate software. But most CLI design conventions were created for humans. Agents have different cognitive patterns and failure modes — they excel at structured output parsing but struggle with ambiguous short flags, interactive prompts, and unstructured text.

This guide provides **10 principles and a checklist** for designing CLI tools that work well for both humans and AI agents.

## Quick Start

If you're using an AI coding assistant (Claude Code, Codex, Cursor, etc.) to build a CLI, point it to this repo:

```
Reference https://github.com/Johnixr/agent-cli-guide for CLI design principles.
Follow the 10 principles and checklist in GUIDE.md.
```

Or add to your `CLAUDE.md` / project instructions:

```
CLI design follows the Agent CLI Guide: https://github.com/Johnixr/agent-cli-guide/blob/main/GUIDE.md
```

## Files

- **[GUIDE.md](GUIDE.md)** — The 10 principles with rationale and examples. This is the core document.
- **[CHECKLIST.md](CHECKLIST.md)** — A concise checklist for quick reference during development.

## Sources

This guide synthesizes insights from:

- [POSIX Utility Conventions](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html) (IEEE Std 1003.1)
- [GNU Coding Standards](https://www.gnu.org/prep/standards/html_node/Command_002dLine-Interfaces.html)
- [Command Line Interface Guidelines](https://clig.dev/) (community-driven)
- [12 Factor CLI Apps](https://medium.com/@jdxcode/12-factor-cli-apps-dd3c227a0e46) (Jeff Dickey / Heroku)
- [Anthropic Tool Use Best Practices](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Lightning Labs agent-CLI design axes](https://github.com/lightninglabs/lnget/pull/14)
- [Berkeley Function Calling Leaderboard](https://gorilla.cs.berkeley.edu/leaderboard.html)
- Real-world analysis of [Feishu CLI](https://github.com/larksuite/cli) and [DingTalk CLI](https://github.com/open-dingtalk/dingtalk-workspace-cli)

## Contributing

PRs welcome. If you have real-world experience building agent-friendly CLIs, your insights are valuable — whether it's a new principle, a better example, or a correction.

## License

[CC BY 4.0](LICENSE)
