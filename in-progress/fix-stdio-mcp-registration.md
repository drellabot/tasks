# Fix stdio MCP Server Registration

_Add `--` separator when registering stdio MCP servers via `claude mcp add` so that command arguments are not parsed as flags. Implement in `drellabot/orchestrator`._

## Motivation

When registering a stdio MCP server from a profile's `mcp.yaml`, the orchestrator builds a command like:

```
claude mcp add --transport stdio patternfly-docs npx -y @patternfly/patternfly-mcp@latest
```

The `-y` argument (intended for `npx`) is parsed as a flag to `claude mcp add`, causing the command to fail. The correct syntax uses `--` to separate `claude mcp add` flags from the command and its arguments:

```
claude mcp add --transport stdio patternfly-docs -- npx -y @patternfly/patternfly-mcp@latest
```

## Proposed Solution

In `registerMCPServer()` in `internal/profile/apply.go`, insert `--` between the server name and the command for stdio transport:

```go
case "stdio":
    args = []string{"claude", "mcp", "add", "--transport", "stdio"}
    if server.Scope != "" {
        args = append(args, "--scope", server.Scope)
    }
    args = append(args, server.Name, "--", server.Command)
    args = append(args, server.Args...)
```

## Acceptance Criteria

- [ ] `registerMCPServer` inserts `--` before the command for stdio transport.
- [ ] A profile MCP server with args containing dashes (e.g. `npx -y ...`) registers successfully.
- [ ] Existing tests continue to pass.

## Open Questions

None.
