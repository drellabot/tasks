# Daemon Front Matter Parsing

_Wire the existing `ParseFrontMatter` function into the daemon's issue intake so that YAML front matter in issue bodies is parsed and passed to `task new` as `--profile` and `--var` flags. Also add a `--var` flag to `task new` for CLI usage. Implement in `drellabot/orchestrator`._

## Motivation

The profile system supports YAML front matter in issue bodies to specify a profile and pass variables to `setup.sh`. The front matter parser (`internal/profile/frontmatter.go`) and the documentation (`README.md`) both exist, but the daemon does not actually call the parser. When an issue contains front matter like:

```markdown
---
profile: code-review
repo: org/repo
pr: 42
---

Review this pull request.
```

The daemon passes the entire body — including the `---` delimiters — as the task description. Cobra interprets `---` as a malformed flag and the task fails.

## Proposed Solution

### 1. Wire front matter parsing into the daemon

In `defaultNewTaskFunc` in `internal/daemon/daemon.go`, parse the description with `profile.ParseFrontMatter` before building the `task new` command. If a profile is found, add `--profile <name>` to the args. If vars are found, add `--var KEY=VALUE` for each. Use the stripped description (without front matter) as the task description.

```go
func (d *Daemon) defaultNewTaskFunc(ctx context.Context, taskName, description string) error {
    exe, err := os.Executable()
    if err != nil {
        return fmt.Errorf("getting executable path: %w", err)
    }

    args := []string{"task", "new", "--config", d.configPath}

    profileName, vars, strippedDesc, err := profile.ParseFrontMatter(description)
    if err != nil {
        slog.Warn("Failed to parse front matter, using raw description", "task", taskName, "error", err)
        strippedDesc = description
    }
    if profileName != "" {
        args = append(args, "--profile", profileName)
    }
    for k, v := range vars {
        args = append(args, "--var", k+"="+v)
    }
    args = append(args, taskName, strippedDesc)

    // ... rest unchanged
}
```

### 2. Add `--var` flag to `task new`

Add a `--var` string slice flag to `taskNewCmd` that accepts `KEY=VALUE` pairs. Parse them into a `map[string]string` and pass to `profile.Apply` instead of the current `nil`.

```go
var profileVars []string

func init() {
    taskNewCmd.Flags().StringSliceVar(&profileVars, "var", nil, "profile variables as KEY=VALUE (e.g. --var PROFILE_PR=42)")
    // ...
}
```

In `setupSandboxWithProfile`, parse the `--var` values into a map and pass to `profile.Apply`:

```go
vars := parseVarFlags(profileVars)
if err := profile.Apply(ctx, p, runner, taskName, taskDir.Path(), prompts.Base, vars); err != nil {
```

### 3. Update `NewTaskFunc` signature (if needed)

The current `NewTaskFunc` signature is `func(ctx context.Context, taskName, description string) error`. This is sufficient because the front matter is embedded in the description and parsed within `defaultNewTaskFunc`. No signature change is needed.

## Acceptance Criteria

- [ ] The daemon parses YAML front matter from issue bodies before passing to `task new`.
- [ ] `--profile` is passed to `task new` when the front matter contains a `profile` key.
- [ ] `--var KEY=VALUE` is passed for each non-profile front matter key.
- [ ] The `---` delimiters are stripped from the task description.
- [ ] Issues without front matter continue to work unchanged.
- [ ] A `--var` flag is added to `task new` for CLI usage.
- [ ] Profile vars from `--var` flags are passed to `profile.Apply`.
- [ ] Unit tests cover: issue with front matter, issue without front matter, malformed front matter fallback.
- [ ] Existing tests continue to pass.

## Open Questions

None.
