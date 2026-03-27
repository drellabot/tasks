# GitHub Issue Intake for Daemon

_The daemon should pick up GitHub issues from the tasks repo and spawn new tasks from them. Implement in `drellabot/orchestrator`._

## Motivation

The orchestrator daemon can pick up spec files from `in-progress/` in the tasks repo, but the simpler intake path described in the tasks repo README — where a human just opens a GitHub issue with a prompt-style description — is not yet implemented. Issues are the lightest-weight way to kick off work.

## Proposed Solution

When `daemon.tasks_repo` is configured, the daemon should monitor that repo for open GitHub issues in addition to spec files. Each new issue should spawn a task using the issue body as the description (falling back to the title if the body is empty). Processed issues should be tracked persistently so they are not re-spawned across daemon restarts.

No new configuration is needed — this piggybacks on the existing `tasks_repo` setting.

## Acceptance Criteria

- [ ] The daemon picks up open GitHub issues from the configured tasks repo and spawns a task for each one.
- [ ] Pull requests are not treated as issues (GitHub's issues API includes PRs).
- [ ] Each issue is only picked up once, even across daemon restarts.
- [ ] An issue whose derived task is already running is skipped but not marked as processed, so it is retried next cycle.
- [ ] No new config fields are required.
- [ ] Unit tests cover the new behavior.
- [ ] Existing spec-intake tests continue to pass.

## Open Questions

None — this is intentionally minimal. Future work could add label gating, automatic issue commenting when a task starts, or closing issues when the resulting PR merges.
