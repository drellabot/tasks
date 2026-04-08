# post_review MCP Tool

_Add a `post_review` tool to the orchestrator's MCP server that submits GitHub pull request reviews from the sandbox. Implement in `drellabot/orchestrator`._

## Motivation

The orchestrator's MCP server exposes `open_pr` and `update_pr` tools so sandboxed Claude sessions can push code and create pull requests through the host (which has GitHub credentials). There is no equivalent for submitting PR reviews. A code review profile needs to post its review back to GitHub, but the sandbox is credential-free and cannot call `gh pr review` directly.

Adding `post_review` to the host MCP server follows the same trust boundary pattern as the existing PR tools: the sandbox requests the action, the host validates and executes it.

## Proposed Solution

### 1. GitHub Runner Method

Add a `PostReview` method to `internal/github/github.go`:

```go
func (r *Runner) PostReview(ctx context.Context, repo string, pr int, event, body string) error
```

- `event` must be one of `APPROVE`, `REQUEST_CHANGES`, or `COMMENT` (case-insensitive)
- Maps to `gh pr review <pr> --repo <repo> --approve|--request-changes|--comment --body <body>`
- Returns an error if the event is invalid or the `gh` command fails

### 2. PROpener Interface

Add `PostReview` to the existing `PROpener` interface in `internal/mcp/server.go`:

```go
type PROpener interface {
    AuthenticatedUser(ctx context.Context) (string, error)
    EnsureFork(ctx context.Context, upstream string) (string, error)
    PushBranch(ctx context.Context, repoDir, forkFullName, branch, sourceRef string) error
    CreatePR(ctx context.Context, upstream, forkOwner, branch, base, title, body string) (string, error)
    PostReview(ctx context.Context, repo string, pr int, event, body string) error
}
```

### 3. MCP Tool Registration

Register `post_review` in `server.go` alongside `open_pr` and `update_pr`, under the same conditions (non-nil `prOpener` and non-empty `allowedRepos`). The tool uses the same `isRepoAllowed` check.

**Input schema:**

```go
type PostReviewInput struct {
    Repo  string `json:"repo"`   // owner/repo
    PR    int    `json:"pr"`     // PR number
    Event string `json:"event"`  // APPROVE, REQUEST_CHANGES, or COMMENT
    Body  string `json:"body"`   // Review body text
}
```

**Tool description:** `"Submit a review on a GitHub pull request"`

**Behavior:**
1. Check repo is in the allowed repos list (same as `open_pr`)
2. Call `prOpener.PostReview` with the input fields
3. Return success message: `"Review posted on {repo}#{pr} ({event})"`
4. Return error message on failure

**Note:** This tool does not require `CodePuller` — no code needs to be pulled from the sandbox to post a review.

### 4. Prompt Update

Add `post_review` documentation to `internal/cmd/prompt.md` following the existing format for `open_pr` and `update_pr`:

```markdown
### post_review
Submit a review on a GitHub pull request.

**Input:**
\`\`\`json
{
  "repo": "owner/repo",
  "pr": 42,
  "event": "COMMENT",
  "body": "Review body text"
}
\`\`\`

- `repo`: Target repository as `owner/repo` (must be in the allowed repos list)
- `pr`: Pull request number
- `event`: One of `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`
- `body`: The review body text
```

## Acceptance Criteria

- [ ] `PostReview` method added to `internal/github/github.go` wrapping `gh pr review`.
- [ ] `PostReview` added to the `PROpener` interface in `internal/mcp/server.go`.
- [ ] `post_review` MCP tool registered alongside `open_pr` and `update_pr` under the same conditions.
- [ ] The tool validates the repo against `allowedRepos` before posting.
- [ ] Invalid `event` values return a clear error.
- [ ] `post_review` is documented in `prompt.md`.
- [ ] Unit tests cover: successful review, repo not allowed, invalid event, `gh` command failure.
- [ ] The `stubPROpener` test helper is updated with `PostReview`.
- [ ] `TestServerListTools` updated to expect `post_review` in the tool list.
- [ ] Existing tests continue to pass.

## Open Questions

None — this is a straightforward addition following established patterns.
