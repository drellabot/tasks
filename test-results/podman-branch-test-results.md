## Orchestrator `podman` Branch Test Results

**Branch:** `podman` on `beav/orchestrator`
**Tested at commit:** `4b9d673` (fix: address code review -- gjoll adapter, Cp paths, shell quoting, file ownership)
**Commits in branch:** 4 commits ahead of `main`

---

### 1. Build & Compilation

| Check | Result |
|-------|--------|
| `go build ./...` | :white_check_mark: Pass -- binary compiles cleanly |
| `go vet ./...` | :white_check_mark: Pass -- no issues |
| Binary output | :white_check_mark: Produces working `orchestrator` binary |

### 2. Unit Test Results

**All 164 tests pass** (`go test -race -cover ./...`)

| Package | Tests | Coverage | Status |
|---------|-------|----------|--------|
| `internal/cmd` | 21 | 24.2% | :white_check_mark: PASS |
| `internal/config` | 13 | 100.0% | :white_check_mark: PASS |
| `internal/daemon` | 28 | 70.4% | :white_check_mark: PASS |
| `internal/github` | 30 | 76.3% | :white_check_mark: PASS |
| `internal/gjoll` | 12 | 70.8% | :white_check_mark: PASS |
| `internal/logging` | 5 | 53.7% | :white_check_mark: PASS |
| `internal/mcp` | 22 | 89.6% | :white_check_mark: PASS |
| `internal/profile` | 25 | 70.2% | :white_check_mark: PASS |
| `internal/shellutil` | 11 | 100.0% | :white_check_mark: PASS |
| `internal/task` | 21 | 88.9% | :white_check_mark: PASS |
| `internal/tasksource` | 12 | 100.0% | :white_check_mark: PASS |
| `internal/sandbox` | 0 | 0.0% | :warning: No test files |
| `internal/prompts` | 0 | N/A | No test files |

No race conditions detected (ran with `-race` flag).

### 3. Gjoll Support Assessment

**Code review: :white_check_mark: Looks correct**

- `GjollAdapter` (`internal/sandbox/gjoll_adapter.go`) properly wraps the existing `gjoll.Runner` to implement the new `sandbox.Runner` interface
- Path translation in `Cp()` correctly converts podman-style `name:/path` to gjoll-style `:/path` via `toGjollPath()`
- `SSHOpts` conversion from `sandbox.SSHOpts` to `gjoll.SSHOpts` handles nil safely via `toGjollOpts()`
- `HelperScripts()` properly shell-quotes the sandbox name using `shellutil.Quote()` to prevent injection
- All existing gjoll unit tests (`internal/gjoll`) pass -- command construction for `Up`, `Start`, `SSH`, `Cp`, `Stop`, `SSHProxy`, `SSHProxyOutput`, and `Down` verified
- Integration tests (`integration_test.go`) still reference gjoll directly for VM-based E2E testing -- these require libvirt/KVM which is not available in this test environment
- **Note:** gjoll binary is not installed on this test VM, so integration tests could not be run. The gjoll adapter is a thin wrapper, and its correctness is validated through the profile shell-injection tests that exercise `NewGjollAdapter()` with a mock gjoll script

### 4. Podman Support Assessment

**Code review: :white_check_mark: Looks correct**

- `PodmanRunner` (`internal/sandbox/podman.go`) implements all `sandbox.Runner` interface methods
- Container provisioning in `Up()`:
  - Creates container with `--network host` and `--security-opt label=disable`
  - Creates non-root `claude` user with proper home directory ownership
  - Installs Claude Code via `su - claude -c 'curl -fsSL https://claude.ai/install.sh | bash'`
  - API key handling: resolves `~/` paths, creates `.anthropic` directory, copies key with `chmod 600`
  - Proper cleanup on failure -- calls `Down()` on every error path
- `SSH()` maps to `podman exec`, `SSHProxy()` maps to `podman exec -it`
- `Pull()` implements git code extraction using `podman cp` + temp directory
- `HelperScripts()` generates correct scripts for `sandbox-cp` and `sandbox-ssh`
- `Down()` uses `podman rm -f` for forceful removal
- Podman v5.6.2 is available on this test VM; binary was verified present

### 5. Sandbox Abstraction Layer

**Code review: :white_check_mark: Well designed**

- `sandbox.Runner` interface (`internal/sandbox/sandbox.go`) is clean with 9 methods covering the full lifecycle
- `NewFromConfig()` factory function correctly routes to either backend based on `sandbox_backend` config
- Defaults: empty/missing backend defaults to `gjoll` for backward compatibility
- Unknown backends produce a clear error message

### 6. Configuration Changes

**:white_check_mark: Properly integrated**

- New config fields: `sandbox_backend`, `podman_image`, `anthropic_key_file`
- Defaults: `sandbox_backend: "gjoll"`, `podman_image: "fedora:43"`, `anthropic_key_file: "~/.anthropic/api_key"`
- Config tests updated to verify new defaults are applied (100% coverage on `internal/config`)
- `orchestrator.yaml.example` updated with new fields
- `README.md` has comprehensive documentation including a comparison table

### 7. Documentation

**:white_check_mark: Thorough**

- README updated with podman setup instructions and prerequisites
- Feature comparison table (startup time, resource usage, isolation, etc.)
- Configuration table includes all new fields
- New `configs/sandbox-anthropic-api.tf.example` for direct Anthropic API gjoll setup

### 8. Security

**:white_check_mark: Shell injection protection verified**

- Profile `apply_test.go` has 10 shell injection tests that exercise `sandbox.NewGjollAdapter()` -- all pass
- `shellutil.Quote()` used consistently in `HelperScripts()` for both gjoll and podman backends
- API key permissions set to `600` in podman containers
- Podman containers use non-root `claude` user

### 9. Issues / Observations

1. **No unit tests for `internal/sandbox`** -- The sandbox package has 0% coverage. While the code is exercised indirectly through profile tests (which use `NewGjollAdapter` with mock scripts), there are no dedicated tests for:
   - `NewFromConfig()` factory function
   - `PodmanRunner` methods
   - `toGjollPath()` path translation
   - `NewPodman()` default image behavior
   
   Adding tests for `NewFromConfig()` (backend routing, unknown backend error) and `toGjollPath()` (path translation logic) would be straightforward and valuable.

2. **Integration tests are gjoll-only** -- The `integration_test.go` file only tests with `gjoll.New("")` directly. A parallel podman integration test would be valuable for CI environments without libvirt.

3. **`SSHProxy` proxy/tunnel opts are ignored in podman** -- The `SSHProxy()` and `SSHProxyOutput()` methods accept `*SSHOpts` but don't use the `Proxy` or `ReverseTunnels` fields. For podman containers on `--network host`, reverse tunnels aren't needed (host ports are already accessible), and there's no credential proxy concept. This is correct behavior but worth documenting in the code or interface.

4. **`executeTask` still calls `filepath.Abs(cfg.GjollEnv)` for all backends** -- In `task.go:185-188`, the gjoll `.tf` path is resolved even when using the podman backend. The resolved path is passed to `runner.Up()` where the podman backend ignores it (it uses its own image name). This is harmless but could be cleaned up.
