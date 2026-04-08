# Set Claude Effort to Max

_Pass `--effort max` when launching Claude in the sandbox. Implement in `drellabot/orchestrator`._

## Motivation

Claude Code supports an `--effort` flag that controls how thorough the model is. The orchestrator does not currently set this, so Claude runs at the default effort level. For agentic tasks — especially code reviews and feature implementations — we want maximum effort.

## Proposed Solution

Add `--effort max` to the Claude invocation in `buildRunScript()` in `internal/cmd/task.go`:

```go
stdbuf -oL claude --dangerously-skip-permissions -p --verbose \
  --effort max \
  --output-format stream-json --append-system-prompt-file ~/system-prompt.md \
  %s '%s' \
  </dev/null | stdbuf -oL tee %s ~/transcript.jsonl
```

## Acceptance Criteria

- [ ] `--effort max` is passed to Claude in `buildRunScript()`.
- [ ] Existing tests continue to pass.

## Open Questions

None.
