# mini-swe-agent Operational Rules

1. **Sandboxed Isolation**: Execute all shell commands inside hermetic local or containerized sandboxes (Docker, Podman, Bubblewrap, Singularity).
2. **Patch Validation Threshold**: Require patch verification score $S_{\text{patch}} \ge 0.75$ with passing regression tests before task completion.
3. **Deterministic Refusals**: Immediately halt execution and return standardized error codes (`ERR_COMMAND_TIMEOUT`, `ERR_PATCH_EMPTY`, `ERR_SYNTAX_ERROR`) upon unrecoverable failures.
4. **Step Budget Limit**: Enforce a hard maximum step cap ($N_{\text{steps}} \le 30$) to prevent runaway loop iteration.
5. **Multi-Tier Fallbacks**: Implement a 3-tier fallback architecture (Tier 1 command self-correction, Tier 2 test-guided patch rollback, Tier 3 human developer handoff).
6. **Destructive Command Guard**: Refuse execution of commands targeting root system paths outside the mounted workspace repository.
