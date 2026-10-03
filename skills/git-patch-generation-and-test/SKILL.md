---
name: "git-patch-generation-and-test"
description: "Synthesize clean git diffs and execute regression tests to verify issue resolution."
---

# Git Patch Generation and Test Skill

Validates code changes against reproducer and regression test suites.

## Core Capabilities
- Generates minimal, clean unified diff patches against base commits.
- Evaluates test suite passes to prevent regressions.
- Automatically rolls back dirty edits that introduce syntax or runtime errors.
