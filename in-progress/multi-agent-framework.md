# Multi-Agent Framework

_Replace the single-agent execution model with an extensible multi-agent framework where specialized agents collaborate on each work item, starting with a producer/validator pairing. Implement in `drellabot/orchestrator`._

## Motivation

Today, a single Claude session handles each implementation request end-to-end: reading the spec, writing code, running tests, and opening the PR. This works for proving out the factory infrastructure, but a single agent has no internal check on quality. There is no second set of eyes to catch bugs, security issues, missing tests, or spec misinterpretation before a human reviewer sees the PR.

A multi-agent approach lets us decompose work into focused responsibilities. A producer agent writes the implementation, then a validator agent reviews it with fresh context and a critical eye. Over time we can introduce more specialized roles (adversarial QE, product security, performance) without rearchitecting the core execution model.

## Proposed Solution

### 1. Agent Role Definitions in Git

Define agent roles as markdown files in a new `agents/` directory at the root of the orchestrator repo. Each file describes one role and contains the system prompt content for that role. The filename (minus extension) is the role identifier used elsewhere in config and code.

```
agents/
  producer.md        # Implements the work item
  validator.md       # Reviews the producer's output
```

The orchestrator loads these at task startup. Adding a new agent role is as simple as adding a new markdown file and wiring it into a pipeline definition (see below).

Agent prompt files should contain role-specific instructions only. The orchestrator will continue to prepend common context (sandbox environment description, available tools, workflow rules) from a shared base prompt, then append the work-item-specific content (spec body or issue text). The resulting prompt structure for each agent session is:

```
[base prompt] + [role prompt from agents/<role>.md] + [work item content]
```

### 2. Pipeline Definitions

Introduce a pipeline concept that defines the ordered sequence of agent roles that execute for a work item. Pipelines are defined in the orchestrator config (`orchestrator.yaml`):

```yaml
pipelines:
  default:
    - role: producer
    - role: validator
```

The `default` pipeline is used unless overridden. This keeps the initial implementation simple while allowing future pipelines for different work types (e.g., a `security-critical` pipeline that adds a `product-security` agent).

Each step in the pipeline runs sequentially. A step receives the output and artifacts from prior steps.

### 3. Agent Execution Model

Each agent role in the pipeline runs as its own Claude session inside the same gjoll VM sandbox. The key changes to task execution in `internal/cmd/task.go`:

**Sequential execution within one VM**: Rather than running a single Claude session per task, `runTask()` iterates through the pipeline steps. Each step:
1. Selects the role prompt from `agents/<role>.md`.
2. Builds the system prompt: base prompt + role prompt + work item content.
3. For the first agent (producer), the work item description is the user prompt. For subsequent agents (validator), the prompt includes a summary of what the prior agent did (the PR diff, or a structured handoff artifact).
4. Runs `claude` with the assembled prompt, capturing the transcript to a role-specific file (e.g., `transcript-producer.jsonl`, `transcript-validator.jsonl`).

**Shared filesystem**: All agents operate on the same VM filesystem. The producer's committed code is available to the validator without extra tooling. The validator can run tests, read code, and inspect the git log to understand what the producer did.

**Independent Claude sessions**: Each agent gets a fresh Claude session (no `--continue`). This is intentional - the validator should form its own assessment rather than inheriting the producer's reasoning context.

### 4. Inter-Agent Handoff and Information Boundaries

When one agent completes and the next begins, the orchestrator constructs a handoff prompt for the incoming agent. The handoff is assembled by the orchestrator, not by the agents themselves. Agents do not need awareness of the multi-agent framework; they simply receive a prompt and do their work.

What each agent sees is controlled per-role in the pipeline definition via a `handoff` field:

```yaml
pipelines:
  default:
    - role: producer
    - role: validator
      handoff:
        include_diff: true
        include_prior_transcript: false
        include_prior_summary: false
```

Available handoff fields:

| Field | Default | Description |
|-------|---------|-------------|
| `include_diff` | `true` | Git diff of changes from the prior agent |
| `include_prior_transcript` | `false` | The full final assistant message from the prior agent's transcript |
| `include_prior_summary` | `false` | A structured summary extracted from the prior agent's transcript |

For the initial producer/validator pairing, the validator receives the git diff and the original work item, but **not** the producer's reasoning or transcript. This is intentional — the validator should evaluate the code against the spec independently, without anchoring on the producer's assumptions. The validator can inspect the git log and code on the shared filesystem, but the producer's chain-of-thought stays private.

This information boundary is a design principle, not just a default. Future roles may have different isolation needs. For example, an adversarial QE agent might receive only the spec and the test suite (no implementation diff), forcing it to test against the specification rather than the code it saw. The per-step `handoff` configuration makes these boundaries explicit and configurable without code changes.

### 5. Validator Agent Behavior and Iteration Loop

The validator is the first non-producer role. Its prompt should instruct it to:

1. Review all changes made by the producer against the original spec/issue.
2. Run the full test suite and linters.
3. Check for common issues: missing test coverage, spec requirements not addressed, obvious bugs, poor error handling, incomplete documentation updates.
4. Produce a structured verdict at the end of its response using a simple format the orchestrator can parse (e.g., a `VERDICT: pass` or `VERDICT: fail` line, with findings listed above it).

The validator should not open new PRs. It operates on the PR branch created by the producer.

**Iteration loop**: The producer and validator are not a single linear pass. They iterate:

1. The **producer** implements the work item and opens a PR.
2. The **validator** reviews and produces a verdict.
3. If the verdict is **pass**, the pipeline advances (to the next step, or completes).
4. If the verdict is **fail**, the orchestrator feeds the validator's findings back to a **fresh producer session**. The new producer session receives the original work item, the current state of the code (via the shared filesystem), and the validator's feedback as its prompt. It does not receive a `--continue` of the prior producer session — fresh context avoids compounding flawed reasoning.
5. The loop repeats until the validator passes or the **max iteration cap** is hit.

The max iteration cap is configurable per pipeline step (default: 3 round-trips). If the cap is reached without a passing verdict, the orchestrator:
- Adds a `needs-human-input` label to the PR.
- Posts a PR comment summarizing the iteration history: what the validator flagged each round and what the producer attempted.
- Stops the pipeline and leaves the PR in draft state for human triage.

```yaml
pipelines:
  default:
    - role: producer
    - role: validator
      max_iterations: 3    # producer-validator round-trips before escalation
```

Per-iteration transcripts are captured with iteration indices (e.g., `transcript-producer-1.jsonl`, `transcript-validator-1.jsonl`, `transcript-producer-2.jsonl`, etc.) to preserve the full history for debugging.

### 6. PR Lifecycle and Readiness Signaling

Today, a PR opened by drellabot signals "ready for human review." With the multi-agent pipeline, the PR exists before the factory is done with it. We need to differentiate "factory still working" from "ready for human review."

**Draft PR approach**: The producer opens the PR as a **draft**. The orchestrator converts the PR from draft to ready-for-review only after the full pipeline completes successfully. This provides clear signaling:

- **Draft PR** = factory is still working (producer-validator loop in progress).
- **Ready PR** = pipeline complete, awaiting human review.

If the pipeline exhausts its iteration cap without a passing verdict, the PR remains in draft and gets a `needs-human-input` label so humans know to intervene.

This requires a small addition to the MCP `open_pr` tool: support for a `draft` parameter (defaulting to `true` when running in a pipeline with more than one step). A new MCP tool or orchestrator action (`mark_pr_ready`) converts the draft to ready when the pipeline completes. Since this is an orchestrator-level action (not agent-initiated), it can be done directly via the GitHub API from the orchestrator after the final pipeline step.

**Pipeline state tracking**: The orchestrator tracks pipeline progress in `state.json` so the daemon knows not to pick up the task for comment-driven continuation until the full pipeline completes:

```json
{
  "pipeline": "default",
  "current_step": 1,
  "iteration": 2,
  "steps": [
    {"role": "producer", "status": "completed", "iterations": 2},
    {"role": "validator", "status": "running", "iteration": 2, "verdict": "pending"}
  ]
}
```

### 7. Changes to the Base Prompt

Factor the current `on_init.md` into two parts:

- **`base.md`** (new): The environment description and common workflow rules (sandbox details, commit guidelines, tool usage). Prepended to every agent session.
- **`agents/producer.md`**: The implementation-focused instructions (write code, run tests, open PR). Replaces the bulk of today's `on_init.md`.
- **`agents/validator.md`**: The review-focused instructions described in section 5.

The existing `on_pr_comment.md` remains unchanged - it is used by the daemon's continuation flow, which is orthogonal to the pipeline.

### 8. Config Changes

New fields in `orchestrator.yaml`:

```yaml
# Directory containing agent role prompt files (default: "agents")
agents_dir: "agents"

# Pipeline definitions
pipelines:
  default:
    - role: producer
    - role: validator
      max_iterations: 3
      handoff:
        include_diff: true
        include_prior_transcript: false
        include_prior_summary: false
```

### 9. Extensibility Path

This design supports future agent roles without structural changes:

- **Adding a role**: Create `agents/<role>.md`, add the role to a pipeline definition.
- **Adding a pipeline**: Define a new named pipeline in config. Future work could select pipelines based on labels, repo, or spec metadata.
- **Planned future roles** (not in scope for this spec):
  - `adversarial-qe`: Actively tries to break the implementation with edge cases and fault injection.
  - `product-security`: Reviews for OWASP top 10, dependency vulnerabilities, secrets exposure.
  - `performance`: Runs benchmarks, checks for regressions, validates resource usage.

## Acceptance Criteria

- [ ] Agent role prompt files are loaded from a configurable directory in the orchestrator repo (`agents/` by default).
- [ ] A `producer.md` and `validator.md` agent role are defined with appropriate prompts.
- [ ] The current `on_init.md` is refactored into a shared base prompt and the producer role prompt without changing the effective prompt content for the producer.
- [ ] Pipeline definitions in `orchestrator.yaml` specify the ordered sequence of agent roles, including `handoff` and `max_iterations` configuration.
- [ ] `runTask()` executes each pipeline step sequentially as an independent Claude session within the same VM.
- [ ] Each agent session receives: base prompt + role prompt + work item content (+ handoff context for non-first agents).
- [ ] Handoff content is controlled per-step via the `handoff` configuration. The validator receives the git diff but not the producer's transcript or reasoning by default.
- [ ] The validator produces a structured verdict (`pass`/`fail`) that the orchestrator can parse.
- [ ] On a `fail` verdict, the orchestrator re-runs a fresh producer session with the validator's feedback, then re-runs the validator — looping until pass or max iterations.
- [ ] Max iterations is configurable per pipeline step (default: 3). When exhausted, the PR gets a `needs-human-input` label and a summary comment, and the pipeline stops.
- [ ] The producer opens PRs as drafts. The orchestrator converts the PR from draft to ready-for-review only after the full pipeline completes successfully.
- [ ] Per-agent transcripts are captured to iteration-indexed files (e.g., `transcript-producer-1.jsonl`, `transcript-validator-1.jsonl`).
- [ ] `state.json` tracks pipeline progress (current step, iteration count, verdict) so the daemon does not trigger continuation until the pipeline completes.
- [ ] Adding a new agent role requires only a new markdown file and a config change (no code changes).
- [ ] Existing single-agent behavior is preserved when the pipeline has only one step (PR opened as ready, no draft/undraft cycle).
- [ ] Unit tests cover pipeline execution, agent prompt assembly, handoff construction, iteration loop, and max-iteration escalation.
- [ ] Existing tests continue to pass.

## Open Questions

- **Pipeline selection**: Should pipelines be selectable per-work-item (e.g., via issue labels or spec frontmatter), or is a single default pipeline sufficient for now? _Recommendation: start with a single default pipeline and add selection logic when the first alternative pipeline is needed._
- **Agent-to-agent context size**: The handoff includes the full git diff. For very large changes, this could exceed context limits. Should we summarize large diffs, or is the full diff preferred? _Recommendation: pass full diff for now; add summarization if context limits become a practical problem._
- **Validator fixing vs. reporting only**: Should the validator make direct fixes (push commits), or only report issues for the producer to address? _Recommendation: start with report-only (the producer iterates on fixes). This keeps the validator's role purely evaluative and avoids muddying the iteration loop with two agents committing to the same branch simultaneously. Direct-fix behavior can be added as a future validator mode if needed._
