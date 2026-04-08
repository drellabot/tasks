# Task Profiles

_Add a profile system to the orchestrator that lets each task type carry its own Claude configuration — CLAUDE.md, setup scripts, MCP servers, and settings — in a separate profiles repo. Implement in `drellabot/orchestrator`._

## Motivation

Today, every task gets the same Claude setup: a single embedded `prompt.md` written to `~/project/CLAUDE.md`, describing the `open_pr`/`update_pr` workflow. This works for implementation tasks but breaks down for other task types like code review, spec review, or security audit — each needs different instructions, different tools, and different workspace layouts.

Changing Claude's setup currently requires modifying Go source and recompiling the orchestrator. This makes iteration slow and couples task-type-specific logic to infrastructure code.

A profile system solves both problems: task-type configuration lives in its own repo (`drellabot/profiles`) as plain files that can be iterated on independently, and the orchestrator applies the right profile at task startup based on a CLI flag or issue front matter.

## Proposed Solution

### 1. Profile Directory Structure

Profiles live in a dedicated GitHub repo (`drellabot/profiles`). Each profile is a directory containing some combination of:

```
drellabot/profiles/
  code-review/
    CLAUDE.md           # Required — profile-specific instructions
    setup.sh            # Optional — workspace setup script (runs on host)
    mcp.yaml            # Optional — additional MCP servers to register
    settings.json       # Optional — Claude Code settings
  default/
    CLAUDE.md           # The current open_pr/update_pr workflow instructions
```

Only `CLAUDE.md` is required. All other files are optional and processed only if present.

### 2. CLAUDE.md Layering

The current `prompt.md` is split into two concerns:

**Base prompt** (embedded in orchestrator, always applied): Environment context only — sandbox description, credential-free note, working directory. No tool documentation (MCP tools already carry their own descriptions via the protocol) and no workflow instructions (those are profile-specific).

Written to `~/.claude/CLAUDE.md` in the sandbox, so it applies regardless of which repo or directory Claude is working in.

Example base prompt:

```markdown
You are working inside a sandboxed VM managed by the orchestrator.

## Environment
- You are in a credential-free VM — no cloud credentials or API keys are available
- Git is pre-configured with author information
- MCP tools are provided by the orchestrator — use them as described in your project context
```

**Profile CLAUDE.md** (from the profile directory): Mission-specific instructions — workflow, guidelines, rubrics, tool usage. Appended to `~/.claude/CLAUDE.md` after the base prompt, separated by a heading.

**Repo CLAUDE.md** (from cloned repos): Repo-specific conventions — coding style, test commands, architecture. These come along naturally when repos are cloned into the sandbox. Claude Code picks them up automatically when working in that directory.

The resulting hierarchy:

| Layer | Location in sandbox | Source | Purpose |
|-------|---------------------|--------|---------|
| Base | `~/.claude/CLAUDE.md` (top) | Embedded in orchestrator | Environment context |
| Profile | `~/.claude/CLAUDE.md` (appended) | `profiles/<name>/CLAUDE.md` | Mission and workflow |
| Repo | `<repo>/CLAUDE.md` | Cloned with the repo | Repo conventions |

### 3. setup.sh — Workspace Setup

When a profile includes `setup.sh`, the orchestrator runs it **on the host** after provisioning the sandbox. Running on the host is essential because setup often requires credentials (cloning private repos, fetching PR data) that the sandbox deliberately does not have.

The orchestrator provides the following environment variables to setup.sh:

| Variable | Description |
|----------|-------------|
| `SANDBOX` | Sandbox name (for gjoll operations) |
| `TASK_DIR` | Task output directory on host |
| `PROFILE_*` | Front matter key-value pairs from the issue/task (e.g., `PROFILE_REPO`, `PROFILE_PR`) |

The orchestrator also makes two helper commands available on PATH:

- **`sandbox-cp <local-path> <sandbox-path>`** — copy files to the sandbox (wraps `gjoll cp`)
- **`sandbox-ssh <command>`** — run a command inside the sandbox (wraps `gjoll ssh`)

These helpers abstract the gjoll CLI so profile authors don't need to know orchestrator internals.

Example `code-review/setup.sh`:

```bash
#!/bin/bash
set -euo pipefail

REPO=${PROFILE_REPO:?}
PR=${PROFILE_PR:?}

# Clone and check out PR on the host (has gh credentials)
TMPDIR=$(mktemp -d)
trap "rm -rf $TMPDIR" EXIT
gh repo clone "$REPO" "$TMPDIR/repo"
cd "$TMPDIR/repo"
gh pr checkout "$PR"

# Copy repo to sandbox
sandbox-cp "$TMPDIR/repo/" :~/project/
```

When no profile is specified, the orchestrator runs the current default setup: initialize `~/project` as an empty git repo with the full `prompt.md` content as `CLAUDE.md`.

### 4. mcp.yaml — Additional MCP Servers

A profile can register additional MCP servers that Claude should have access to. These are registered via `claude mcp add` inside the sandbox during setup.

```yaml
# code-review/mcp.yaml
servers:
  - name: review-tools
    transport: http
    url: http://localhost:19091/mcp
    scope: user
```

The orchestrator iterates over the `servers` list and runs `claude mcp add` for each entry. The MCP servers themselves must be running on the host (and tunneled into the sandbox via gjoll, similar to the existing orchestrator MCP server on port 19090).

The built-in orchestrator MCP server (`open_pr`, `update_pr`) is always registered regardless of profile. Profiles add servers; they do not remove or filter built-in tools.

### 5. settings.json — Claude Code Settings

If the profile includes `settings.json`, it is copied to `~/.claude/settings.json` in the sandbox. This can configure Claude Code settings such as allowed tools, custom slash commands, or hooks.

### 6. Profile Loading

When `--profile <name>` is specified on `task new`, the orchestrator loads the profile:

1. **Locate the profile source**: Use `profiles_dir` if set (local directory, for development). Otherwise, shallow-clone `profiles_repo` to a temporary directory.
2. **Find the profile**: Look for `<name>/` in the profile source. Error if not found.
3. **Validate**: Ensure `CLAUDE.md` exists in the profile directory. Warn (but don't fail) if no other files are present.
4. **Apply during sandbox setup** (see section 7).

The profiles repo is cloned fresh per task. This is simple and ensures the latest profile version is always used. For local development, `profiles_dir` points to a checkout on disk, enabling edit-and-rerun iteration without pushing.

### 7. Sandbox Setup Changes

The current `setupSandbox()` in `internal/cmd/task.go` is restructured:

**Always (both profile and no-profile paths):**
1. Configure git (`user.name`, `user.email`)
2. Register orchestrator MCP server (`claude mcp add orchestrator http://localhost:19090/mcp`)

**When a profile is specified (`--profile <name>`):**
3. Write base prompt + profile CLAUDE.md → `~/.claude/CLAUDE.md`
4. Copy `settings.json` → `~/.claude/settings.json` (if present)
5. Process `mcp.yaml` — run `claude mcp add` for each server entry (if present)
6. Write `sandbox-cp` and `sandbox-ssh` helper scripts to a temp directory on host
7. Run `setup.sh` on the host with helpers on PATH and environment variables set (if present)

**When no profile is specified (backward compatibility):**
3. `mkdir -p ~/project && cd ~/project && git init`
4. Write current full `prompt.md` → `~/project/CLAUDE.md`
5. `git add -A && git commit -m 'Initial setup'`

The no-profile path preserves today's behavior exactly. Existing tasks work without changes.

### 8. CLI Changes

Add a `--profile` flag to `task new`:

```
orchestrator task new --profile code-review review-pr-42 "Review PR #42 in org/repo"
```

The `task continue` command does NOT accept `--profile` — the sandbox is already set up from the initial run.

### 9. Config Changes

Two new fields in `orchestrator.yaml`:

```yaml
profiles_repo: "drellabot/profiles"   # GitHub repo containing profiles (cloned via gh)
profiles_dir: ""                       # Local directory override (for development iteration)
```

Both are optional. If neither is set and `--profile` is used, the orchestrator errors with a clear message.

`profiles_dir` takes precedence over `profiles_repo` when both are set.

### 10. Issue Intake Integration

When the GitHub issue intake daemon (see `github-issue-intake` spec) processes an issue, it should parse YAML front matter from the issue body:

```markdown
---
profile: code-review
repo: org/repo
pr: 42
---

Review this pull request.
```

- `profile` maps to the `--profile` flag on `task new`.
- All other front matter key-value pairs are passed to `setup.sh` as `PROFILE_*` environment variables (keys uppercased, hyphens converted to underscores).
- The body after the front matter delimiter becomes the task description.

This spec defines the front matter contract. The actual parsing is implemented as part of the `github-issue-intake` spec.

### 11. Profiles Repo Structure

The `drellabot/profiles` repo should be initialized with:

```
README.md                   # How to create and test profiles
default/
  CLAUDE.md                 # Current open_pr/update_pr workflow (extracted from prompt.md)
code-review/
  CLAUDE.md                 # Review workflow, guidelines, rubric (skeleton)
  setup.sh                  # Clone repo + checkout PR (skeleton)
```

**`default/CLAUDE.md`** contains the workflow instructions currently in `prompt.md` — the `open_pr`/`update_pr` documentation and commit-then-push workflow. This preserves the current behavior as a profile.

**`code-review/CLAUDE.md`** (skeleton) describes:
- The review mission: read the PR, understand the changes in context, post a review
- Review guidelines: what to focus on (correctness, security, maintainability), what to skip (style nitpicks, personal preference)
- Expected MCP tools: `get_pr`, `post_review` (defined in a separate spec)
- Output format: structured review with severity levels

**`code-review/setup.sh`** (skeleton) clones the target repo and checks out the PR branch using `PROFILE_REPO` and `PROFILE_PR` environment variables.

### 12. Compatibility with Multi-Agent Framework

Profiles and pipelines are orthogonal:

- **Profiles** configure the sandbox environment: what Claude sees (CLAUDE.md), what tools are available (MCP servers), what code is present (setup.sh).
- **Pipelines** (from the multi-agent-framework spec) configure agent sequencing: which roles run, in what order, with what handoff boundaries.

They compose naturally. A code-review profile could run with a single-agent pipeline (one reviewer) or a multi-agent pipeline (reviewer + adversarial reviewer). The prompt assembly becomes:

```
~/.claude/CLAUDE.md: [base prompt] + [profile CLAUDE.md]
Per-session:         [role prompt from agents/<role>.md] + [work item content]
```

No changes to the multi-agent framework spec are needed to support profiles.

### 13. Implementation Guidance

New package `internal/profile/` with:

- **`profile.go`**: `Load(source, name string) (*Profile, error)` — loads and validates a profile from a directory. `Profile` struct holds parsed content (CLAUDE.md text, setup.sh path, mcp.yaml entries, settings.json path).
- **`apply.go`**: `Apply(ctx context.Context, profile *Profile, runner GjollRunner, sandbox string, vars map[string]string) error` — applies a profile to a sandbox (writes CLAUDE.md, runs setup.sh, registers MCP servers, copies settings).
- **`frontmatter.go`**: `ParseFrontMatter(body string) (profile string, vars map[string]string, description string, error)` — extracts front matter from issue body.

Changes to existing files:

- **`internal/config/config.go`**: Add `ProfilesRepo` and `ProfilesDir` fields.
- **`internal/cmd/task.go`**: Add `--profile` flag. In `executeTask`, load and apply profile before running Claude. Refactor `setupSandbox` into profile-aware and no-profile paths.
- **`internal/cmd/prompt.md`**: Slim down to base environment context only.

## Acceptance Criteria

- [ ] A `--profile <name>` flag is available on `task new`.
- [ ] Profiles are loaded from `profiles_dir` (if set) or cloned from `profiles_repo`.
- [ ] The profile's `CLAUDE.md` is appended to the base prompt and written to `~/.claude/CLAUDE.md` in the sandbox.
- [ ] The current `prompt.md` is split: environment context becomes the base prompt; workflow instructions move to `default/CLAUDE.md` in the profiles repo.
- [ ] When no `--profile` is specified, existing behavior is preserved exactly (empty `~/project` repo with full `prompt.md` as `CLAUDE.md`).
- [ ] `setup.sh` runs on the host with `SANDBOX`, `TASK_DIR`, and `PROFILE_*` environment variables. Helper commands `sandbox-cp` and `sandbox-ssh` are available on PATH.
- [ ] `mcp.yaml` entries are registered via `claude mcp add` in the sandbox.
- [ ] `settings.json` is copied to `~/.claude/settings.json` if present.
- [ ] Front matter in issue bodies is parseable: `profile` selects the profile; other keys become `PROFILE_*` env vars for `setup.sh`.
- [ ] A `drellabot/profiles` repo is initialized with `default/` and a skeleton `code-review/` profile.
- [ ] `profiles_repo` and `profiles_dir` config fields are supported, with `profiles_dir` taking precedence.
- [ ] Unit tests cover profile loading, validation, CLAUDE.md assembly, front matter parsing, and setup.sh environment variable construction.
- [ ] Existing tests continue to pass.

## Open Questions

- **Tunnel management for profile MCP servers**: The orchestrator currently sets up gjoll tunnels for port 19090 (orchestrator MCP) and 18080 (Vertex proxy) in `sandbox.tf`. Profile MCP servers on other ports would need additional tunnels. Should the orchestrator dynamically configure tunnels based on `mcp.yaml`, or should profile authors manage tunnel configuration separately? _Recommendation: start with a fixed set of tunnel ports (e.g., 19091-19095 reserved for profile MCP servers) and revisit dynamic tunnels if the fixed range becomes limiting._
- **Profile versioning**: Should tasks pin a specific profile version (commit SHA), or always use the latest? _Recommendation: always use latest for now. Add version pinning if reproducibility becomes a requirement._
- **`default` profile vs. no-profile**: Should running without `--profile` be identical to `--profile default`, or should they remain separate code paths? _Recommendation: keep them separate initially. The no-profile path is the backward-compatible path with the embedded prompt. Once profiles are proven, the no-profile path can be deprecated in favor of `--profile default`._
