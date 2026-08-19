# PR #4562 Review — Comment #4: CI Job Missing Tools

**Reviewer:** MikeFalcon77
**File:** `.github/workflows/gear-scoped-ci.yml` (scoped-ci job)
**AI TODO:** "investigate if it's a valid concern carefully, suggest a fix"
**Severity:** 🔴 **High** — The scoped-ci job will fail on the very first invocation. 5 of 8 `check` sub-targets require tools not installed in the workflow. This is a latent blocker (never triggered yet because the gate hasn't fired).
**Verdict:** ✅ **Resolved** — Added `gear-ci` Makefile target (fmt + clippy + test only, no extra
tools). Updated `gear-scoped-ci.yml` to call `make gear-ci GEAR=<gear>` instead of
`make all GEAR=<gear>`. Simplified `classify_scope()` — removed `GEAR_PKG`/`GEAR_SDK_PKG`
passthrough. Implemented Option A. See commit `a4f8c6e7`.

---

## MikeFalcon77's Concern

The scoped-ci job can't run `make all` with the tools it installs.

`make all GEAR=… → check → fmt cfs-validate clippy lychee security dylint
gts-docs test`. The job installs only **protoc, Go, cargo-hack, nextest, and
cargo-llvm-cov**.

`check_tool` will fail on:
- **`cfs`** — installed via `pipx install` from the Studio repo
- **`lychee`** — link checker
- **`cargo-gears`** — needed by `dylint`
- **`gts-validator`** — needed by `gts-docs`
- **`cargo-deny`** — needed by `security` (which runs before `test`)

`touch .setup-stamp` only short-circuits the `setup` target; nothing here
actually invokes `setup`, so it doesn't help. The job has never executed
because no PR has triggered it yet.

## Investigation

### Confirmed: Valid concern

The `check` target chain is:

```makefile
check: fmt cfs-validate clippy lychee security dylint gts-docs test
```

And `all` expands to:

```makefile
all: check test-sqlite e2e-local openapi
```

The scoped-ci job installs (line 237-239):
```yaml
tool: cargo-hack, nextest, cargo-llvm-cov
```

Missing tools that would cause `check_tool` failures:

| Tool | Required by | Install method in full CI |
|---|---|---|
| `cfs` | `cfs-validate` | `pipx install git+...studio.git` |
| `lychee` | `lychee` | `make install-tools` / system package |
| `cargo-gears` | `dylint` | `cargo install cargo-gears` |
| `gts-validator` | `gts-docs` | `cargo install gts-validator` |
| `cargo-deny` | `security` (→ `deny`, `fips-policy`) | `make install-tools` |

The `touch .setup-stamp` trick (line 249) is ineffective because `check` does
NOT depend on `setup` — each sub-target calls `check_tool` independently.

The job would fail on the first `check_tool` call that finds a missing binary.
Since the workflow has never fired (hit rate ~2.7%, and this PR isn't scoped),
the bug is latent.

---

## Action Options

### Option A: Replace `make all` with a narrower target (recommended)

Create a new Makefile target `gear-ci` that runs only the tool-light subset:

```makefile
gear-ci: fmt clippy test
	$(call print_target_banner)
```

Update `gear-scoped-ci.yml` to call `make gear-ci GEAR=<gear>` instead of
`make all GEAR=<gear>`. This skips `cfs-validate`, `lychee`, `security`,
`dylint`, `gts-docs`, `test-sqlite`, `e2e-local`, and `openapi` — none of
which are useful for a fast-path scoped check. The `GEAR_HAS_SERVER_FEATURE`
guard (now implemented) already handles openapi/e2e-local gracefully.

### Option B: Run `make install-tools` as a CI step

Add `make install-tools` before the scoped command. `install-tools` now
auto-installs `cargo-gears`, `cargo-nextest`, and `cargo-deny` with version
checks via `cargo gears --check-version` / `tools check-version`. This covers
3 of the 5 missing tools. Remaining gaps: `cfs`, `lychee`, `gts-validator`.

### Option C: Install the full tool set in gear-scoped-ci.yml

Add steps to install all missing tools. Makes the scoped CI a true mirror of
full CI but loses the speed advantage.
