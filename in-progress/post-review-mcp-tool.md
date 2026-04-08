# post_review MCP Tool

_Add a `post_review` tool to the orchestrator's MCP server that submits GitHub pull request reviews from the sandbox. Implement in `drellabot/orchestrator`._

## Motivation

The orchestrator can open PRs (`open_pr`), push updates (`update_pr`), and comment on its own PRs (`comment_on_pr`). But there is no way for a sandboxed Claude session to submit a formal GitHub review on a PR it didn't open — the kind with APPROVE, REQUEST_CHANGES, or COMMENT verdicts and inline file comments.

This is needed for code review profiles. A review profile clones a repo, checks out a PR, and reviews it. To post the review back to GitHub, it needs a tool that wraps `gh pr review` through the host (which has GitHub credentials).

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

Add `PostReview` to the `PROpener` interface in `internal/mcp/server.go`:

```go
type PROpener interface {
    AuthenticatedUser(ctx context.Context) (string, error)
    EnsureFork(ctx context.Context, upstream string) (string, error)
    PushBranch(ctx context.Context, repoDir, forkFullName, branch, sourceRef string) error
    CreatePR(ctx context.Context, upstream, forkOwner, branch, base, title, body string) (string, error)
    AddCoAuthorTrailers(ctx context.Context, repoDir, upstream, base, sourceRef, trailer string) error
    CommentOnPR(ctx context.Context, prURL, body string) error
    UpdatePRTitle(ctx context.Context, prURL, title string) error
    PostReview(ctx context.Context, repo string, pr int, event, body string) error
}
```

### 3. MCP Tool Registration

Register `post_review` in `server.go` alongside the existing tools, under the same conditions (non-nil `prOpener` and non-empty `allowedRepos`). The tool uses the same `isRepoAllowed` check.

**Input schema:**

```go
type PostReviewInput struct {
    Repo  string `json:"repo" jsonschema_description:"Target repository as owner/repo (e.g. osbuild/osbuild)"`
    PR    int    `json:"pr" jsonschema_description:"Pull request number"`
    Event string `json:"event" jsonschema_description:"Review action: APPROVE, REQUEST_CHANGES, or COMMENT"`
    Body  string `json:"body" jsonschema_description:"Review body text (markdown supported)"`
}
```

**Tool description:** `"Submit a review on a GitHub pull request"`

**Behavior:**
1. Check repo is in the allowed repos list
2. Call `prOpener.PostReview` with the input fields
3. Return success message: `"Review posted on {repo}#{pr} ({event})"`
4. Return error on failure (invalid event, gh failure, repo not allowed)

Unlike `comment_on_pr`, this tool does **not** require the PR to have been opened by the current task — its purpose is to review PRs opened by others.

### 4. Prompt Update

Add `post_review` to the tool documentation in `internal/prompts/on_init.md` if appropriate, or rely on the MCP tool's self-description. Since review profiles provide their own CLAUDE.md with workflow instructions, the tool's `jsonschema_description` fields are likely sufficient.

## Acceptance Criteria

- [ ] `PostReview` method added to `internal/github/github.go` wrapping `gh pr review`.
- [ ] `PostReview` added to the `PROpener` interface in `internal/mcp/server.go`.
- [ ] `post_review` MCP tool registered alongside the existing tools under the same conditions.
- [ ] The tool validates the repo against `allowedRepos` before posting.
- [ ] Invalid `event` values return a clear error.
- [ ] The `stubPROpener` test helper is updated with `PostReview`.
- [ ] `TestServerListTools` updated to expect `post_review`.
- [ ] Unit tests cover: successful review, repo not allowed, invalid event, `gh` command failure.
- [ ] Existing tests continue to pass.

## Open Questions

None.
