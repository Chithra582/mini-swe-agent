# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **mini-swe-agent** (`mini-swe-agent`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** mini-swe-agent (`mini-swe-agent`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** Autonomous Software Engineering & Benchmark Evaluation  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

mini-swe-agent navigates codebases, formulates bug fixes, and verifies patches through a deterministic 5-stage decision pipeline.

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

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

### 2. Decision Logic & Routing Formulations

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

### 3. Thresholding & Refusal Decision Criteria

mini-swe-agent enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_COMMAND_TIMEOUT**: Shell command duration $> 120\,\text{seconds}$ halts execution with code `ERR_COMMAND_TIMEOUT`.
- **Refusal on ERR_PATCH_EMPTY**: `git diff` produces zero modifications on completion halts execution with code `ERR_PATCH_EMPTY`.
- **Refusal on ERR_SYNTAX_ERROR**: Modified Python/code files fail AST compilation checks halts execution with code `ERR_SYNTAX_ERROR`.
- **Refusal on ERR_SANDBOX_ESCAPE_ATTEMPT**: Command attempts path traversal outside `/workspace` halts execution with code `ERR_SANDBOX_ESCAPE_ATTEMPT`.
- **Refusal on ERR_STEP_LIMIT_EXCEEDED**: Total turns exceed step cap ($N_{\text{steps}} > 30$) halts execution with code `ERR_STEP_LIMIT_EXCEEDED`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **Tier 1 (Automated SelfCorrection via Traceback):** When a command outputs nonzero return codes or stack traces, feed the raw stderr directly back into the linear message trajectory for immediate selfcorrection.
- **Tier 2 (TestGuided Patch Rollback):** If an edit causes previously passing tests to fail, execute `git checkout .` to restore a clean working tree and reattempt modification.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Tier 3 (Human Review Handoff):** When the agent exhausts its 30step budget without achieving $S_{\text{patch}} \ge 0.75$, export the full linear trajectory JSON for human engineer triage.
- **Session Telemetry Auditing**: Operators inspect execution logs, routing traces, and token usage to maintain oversight.

---

## The Data It Uses

mini-swe-agent operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **Issue Prompts**: Natural language problem descriptions, GitHub issue bodies, and issue reproduction steps.
- **Repository Workspaces**: Local directory git checkouts, source trees, and test directories.
- **Execution Streams**: Real-time stdout, stderr, and exit codes from executed bash commands.

### 2. Configuration & Reference Data

- **Benchmark Specifications**: SWE-bench, ProgramBench, and DeepSWE environment setups and test runners.
- **Docker Images**: Pre-built conda/venv container environments with pre-installed language runtimes.
- **System Prompts**: Minimal, un-opinionated system prompt templates instructing the LLM on command formatting.

### 3. Base Model & Inference Lineage

- **Model Agnostic via LiteLLM**: Interfaces with Claude 3.5 Sonnet, OpenAI GPT-4o, DeepSeek-V3, Qwen 2.5, and local Ollama/vLLM endpoints.
- **Zero Weight Modification**: Operates purely as an inference harness over frozen foundation model checkpoints.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of mini-swe-agent is essential for effective deployment.

### 1. Complex GUI, web frontend, or mobile
- **Limitation**: Complex GUI, web frontend, or mobile applications cannot be evaluated via text-only bash execution.
- **Mitigation**: mini-swe-agent focuses exclusively on headless backend, systems, and CLI software engineering benchmarks.

### 2. Stateless execution requires each command to
- **Limitation**: Stateless execution requires each command to stand alone, meaning `cd dir && command` must be combined in one line.
- **Mitigation**: The system prompt explicitly instructs the language model on writing self-contained single-line bash commands.

### 3. Long-running test suites (e
- **Limitation**: Long-running test suites (e.g., full Django or SymPy test runs) can trigger command execution timeouts.
- **Mitigation**: The agent is guided to execute only targeted test files rather than the entire test repository suite.

### 4. Context window saturation can occur on
- **Limitation**: Context window saturation can occur on tasks with verbose command outputs (e.g., huge compiler logs).
- **Mitigation**: Automatic stdout truncation keeps head and tail lines while summarizing omitted intermediate content.

### 5. Foundation models can hallucinate command arguments
- **Limitation**: Foundation models can hallucinate command arguments or non-existent shell utilities.
- **Mitigation**: Non-zero exit codes from standard Linux shells immediately ground the model with exact error messages.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - Complex GUI, web frontend, or mobile | Section 1 | Verified |
| - Stateless execution requires each command to | Section 2 | Verified |
| - Long-running test suites (e | Section 3 | Verified |
| - Context window saturation can occur on | Section 4 | Verified |
| - Foundation models can hallucinate command arguments | Section 5 | Verified |
