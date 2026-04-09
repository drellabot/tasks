# Use Absolute Path for Claude CLI in Sandbox

_Use the absolute path to the `claude` binary when running commands via `gjoll ssh` in the sandbox. Implement in `drellabot/orchestrator`._

## Motivation

The orchestrator runs `claude mcp add` inside the sandbox via `gjoll ssh` to register MCP servers. This uses a non-interactive shell that does not source `~/.bashrc`, so `claude` is not on PATH and the command fails:

```
Error: setting up sandbox: applying profile: registering MCP server "patternfly-docs": gjoll ssh: exit status 1
```

This affects both the orchestrator's own MCP registration in `internal/cmd/task.go` and profile MCP registration in `internal/profile/apply.go`.

## Proposed Solution

Replace bare `claude` with its absolute path `~/.claude/bin/claude` in all `gjoll ssh` invocations that call the CLI. This is cleaner than sourcing `~/.bashrc` because it avoids pulling in the full shell environment and reduces surface area.

### Files to change

**`internal/cmd/task.go`** — line 299:
```go
// Before:
"claude mcp add --transport http orchestrator %s --scope user"
// After:
"~/.claude/bin/claude mcp add --transport http orchestrator %s --scope user"
```

**`internal/profile/apply.go`** — `registerMCPServer()` function, lines 83-95:
```go
// Before:
args = []string{"claude", "mcp", "add", ...}
// After:
args = []string{"~/.claude/bin/claude", "mcp", "add", ...}
```

## Acceptance Criteria

- [ ] All `gjoll ssh` invocations that call `claude` use `~/.claude/bin/claude`.
- [ ] Profile MCP server registration succeeds in the sandbox.
- [ ] Orchestrator MCP server registration succeeds in the sandbox.
- [ ] Existing tests continue to pass.

## Open Questions

None.
