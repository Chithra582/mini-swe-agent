---
name: "stateless-bash-execution"
description: "Execute self-contained bash commands reliably via isolated subprocesses."
---

# Stateless Bash Execution Skill

Executes shell commands without maintaining persistent terminal state.

## Core Capabilities
- Runs self-contained bash commands with independent subprocess calls.
- Intercepts non-zero return codes and captures stdout/stderr streams.
- Enforces strict timeout limits to prevent hanging operations.
