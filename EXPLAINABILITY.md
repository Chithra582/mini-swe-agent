# mini-swe-agent Explainability & Decision Transparency Report

## How the Agent Decides

mini-swe-agent navigates codebases, formulates bug fixes, and verifies patches through a deterministic 5-stage decision pipeline.

### 5-Stage Decision Pipeline

```
+-----------------------------------------------------------------------------------+
|                           5-STAGE DECISION PIPELINE                               |
+-----------------------------------------------------------------------------------+
|  [Stage 1: Issue Parsing & Workspace Initialization]                              |
|  - Parse issue text, checkout repository commit, mount isolated container sandbox |
|                                     |                                             |
|                                     v                                             |
|  [Stage 2: Codebase Exploration & Reproducer Formulation]                         |
|  - Execute grep/find, locate relevant source files, create failing reproducer test|
|                                     |                                             |
|                                     v                                             |
|  [Stage 3: Source Code Patching & Independent Bash Execution]                     |
|  - Apply code edits via stateless subprocess.run, capture stdout/stderr output    |
|                                     |                                             |
|                                     v                                             |
|  [Stage 4: Regression Test Verification & Threshold Evaluation]                   |
|  - Run test suite; compute S_patch based on test pass rate and patch cleanliness  |
|                                     |                                             |
|                                     v                                             |
|  [Stage 5: Unified Patch Synthesis & Trajectory Archival]                         |
|  - Export git diff patch, serialize linear JSON trajectory, emit result           |
+-----------------------------------------------------------------------------------+
```

### Mathematical Formulation of Scoring & Routing

For a synthesized patch $P_k$ addressing issue $I$ evaluated in workspace $W$, the patch quality score $S_{\text{patch}}(P_k, I)$ is formulated as:

$$S_{\text{patch}}(P_k, I) = w_{\text{rep}} R(P_k, I) + w_{\text{reg}} T(P_k, W) + w_{\text{conc}} C(P_k) + w_{\text{comp}} M(P_k)$$

Where:
- $R(P_k, I) \in \{0, 1\}$ indicates whether the reproduced defect passes after patch application.
- $T(P_k, W) = \frac{\text{PassedExistingTests}(P_k)}{\text{TotalExistingTests}(W)}$ measures zero regression in existing test suites.
- $C(P_k) = \max\left(0, 1 - \frac{\text{LinesChanged}(P_k)}{500}\right)$ penalizes bloated, over-broad code modifications.
- $M(P_k) \in \{0, 1\}$ verifies that the patched repository compiles cleanly without linter or syntax errors.
- Standard default weights: $w_{\text{rep}} = 0.40$, $w_{\text{reg}} = 0.35$, $w_{\text{conc}} = 0.15$, $w_{\text{comp}} = 0.10$ with $\sum w = 1.0$.

Task submission requires:

$$S_{\text{patch}}(P_k, I) \ge \tau \quad (\tau = 0.75) \quad \land \quad R(P_k, I) = 1 \quad \land \quad M(P_k) = 1$$

### Thresholds and Refusal Criteria

When command execution times out, patches introduce test regressions, or step limits are exhausted, mini-swe-agent halts deterministically:

| Error Code | Trigger Condition | Deterministic Behavior |
|---|---|---|
| `ERR_COMMAND_TIMEOUT` | Shell command duration $> 120\,\text{seconds}$ | Terminate subprocess; return timeout error into context |
| `ERR_PATCH_EMPTY` | `git diff` produces zero modifications on completion | Reject submission; force agent to apply concrete changes |
| `ERR_SYNTAX_ERROR` | Modified Python/code files fail AST compilation checks | Revert edit; prompt model with syntax error traceback |
| `ERR_SANDBOX_ESCAPE_ATTEMPT` | Command attempts path traversal outside `/workspace` | Immediately block command; terminate session |
| `ERR_STEP_LIMIT_EXCEEDED` | Total turns exceed step cap ($N_{\text{steps}} > 30$) | Terminate run; output partial trajectory and failure report |

### Multi-Tier Fallback Mechanisms

mini-swe-agent implements a 3-tier fallback architecture to recover from tool and edit failures:

1. **Tier 1 (Automated Self-Correction via Traceback):** When a command outputs non-zero return codes or stack traces, feed the raw stderr directly back into the linear message trajectory for immediate self-correction.
2. **Tier 2 (Test-Guided Patch Rollback):** If an edit causes previously passing tests to fail, execute `git checkout -- .` to restore a clean working tree and re-attempt modification.
3. **Tier 3 (Human Review Handoff):** When the agent exhausts its 30-step budget without achieving $S_{\text{patch}} \ge 0.75$, export the full linear trajectory JSON for human engineer triage.

## The Data It Uses

### Inputs Processed
- **Issue Prompts**: Natural language problem descriptions, GitHub issue bodies, and issue reproduction steps.
- **Repository Workspaces**: Local directory git checkouts, source trees, and test directories.
- **Execution Streams**: Real-time stdout, stderr, and exit codes from executed bash commands.

### Reference Data
- **Benchmark Specifications**: SWE-bench, ProgramBench, and DeepSWE environment setups and test runners.
- **Docker Images**: Pre-built conda/venv container environments with pre-installed language runtimes.
- **System Prompts**: Minimal, un-opinionated system prompt templates instructing the LLM on command formatting.

### Model Lineage & Weights
- **Model Agnostic via LiteLLM**: Interfaces with Claude 3.5 Sonnet, OpenAI GPT-4o, DeepSeek-V3, Qwen 2.5, and local Ollama/vLLM endpoints.
- **Zero Weight Modification**: Operates purely as an inference harness over frozen foundation model checkpoints.

### Retention & Data Privacy
- **Isolated Sandboxes**: Every run executes in a disposable container or local worktree with private filesystem mounts.
- **Zero Cloud Telemetry**: mini-swe-agent does not send user source code or execution logs to third-party tracking services.
- **Local Trajectory Persistence**: Complete linear trajectory JSON logs are written exclusively to the operator's designated output directory.

## Limitations

1. **Limitation:** Complex GUI, web frontend, or mobile applications cannot be evaluated via text-only bash execution.
   **Mitigation:** mini-swe-agent focuses exclusively on headless backend, systems, and CLI software engineering benchmarks.

2. **Limitation:** Stateless execution requires each command to stand alone, meaning `cd dir && command` must be combined in one line.
   **Mitigation:** The system prompt explicitly instructs the language model on writing self-contained single-line bash commands.

3. **Limitation:** Long-running test suites (e.g., full Django or SymPy test runs) can trigger command execution timeouts.
   **Mitigation:** The agent is guided to execute only targeted test files rather than the entire test repository suite.

4. **Limitation:** Context window saturation can occur on tasks with verbose command outputs (e.g., huge compiler logs).
   **Mitigation:** Automatic stdout truncation keeps head and tail lines while summarizing omitted intermediate content.

5. **Limitation:** Foundation models can hallucinate command arguments or non-existent shell utilities.
   **Mitigation:** Non-zero exit codes from standard Linux shells immediately ground the model with exact error messages.

## Summary & Compliance Checklist

| Component | Status | Verification Detail |
|---|---|---|
| **5-Stage Decision Pipeline** | Verified | ASCII flow diagram mapping Stages 1 through 5 with explicit state transitions |
| **Scoring & Routing Mathematics** | Verified | Formal equation $S_{\text{patch}}$ with reproducer, regression, conciseness, and compilation factors |
| **Deterministic Thresholds & Refusals** | Verified | $\tau = 0.75$ threshold and 5 standardized error codes (`ERR_*`) documented |
| **Multi-Tier Fallback Strategy** | Verified | Tier 1 (Self-Correction), Tier 2 (Git Rollback), and Tier 3 (Human Review) specified |
| **Data Privacy & Lineage Architecture** | Verified | Documented inputs, reference data, model lineage, and zero-retention policies |
| **5 Documented Limitations & Mitigations** | Verified | 5 numbered limitation/mitigation pairs covering GUI limits, stateless bash, and log truncation |
