# PR #4562 Review — Comment #6: Two-Dot Diff vs Three-Dot Diff

**Reviewer:** MikeFalcon77
**File:** `.github/workflows/gear-scoped-ci.yml` (detect job, diff command)
**AI TODO:** "investigate if it's a valid concern carefully, suggest a fix"
**Severity:** 🟠 **Medium** — Two-dot diff inflates the changed-file set with unrelated merged commits, making the already-low 2.7% hit rate even lower for any branch that isn't freshly rebased. Not a correctness bug (it errs on the safe side by falling back to full CI) but significantly reduces the workflow's practical value.
**Verdict:** ✅ **Resolved** — Replaced two-dot diff with three-dot via `git merge-base`.
Only the PR's own changes are now considered, preventing spurious non-scoped verdicts
on long-lived branches. Implemented Option A. See commit `7b6b04b3`.

---

## MikeFalcon77's Concern

The detect job uses a **two-dot diff** against `base.sha`:

```python
diff = subprocess.run(
    ["git", "diff", "--name-only", os.environ["BASE_SHA"], os.environ["HEAD_SHA"]],
    ...
)
```

Two-dot diff (`BASE..HEAD`) compares the two tips, so anything merged into
`main` after the PR branched shows up as changed. Given the gate requires
every changed path to sit in one scope, that turns into **spurious
non-scoped verdicts** that grow with branch age — on an already very low
hit rate.

Three-dot (`$BASE_SHA...$HEAD_SHA`) or an explicit `git merge-base` gives the
actual PR contents. `ci.yml` sidesteps this by using `dorny/paths-filter`,
which handles the base resolution internally.

## Investigation

### Confirmed: Valid concern

The `base.sha` in `github.event.pull_request.base.sha` is the tip of the base
branch at the time the PR was opened/updated. On GitHub Actions:

- **Two-dot** `git diff BASE HEAD` shows all commits reachable from HEAD but
  not from BASE — this includes everything merged into main since the PR
  branched.
- **Three-dot** `git diff BASE...HEAD` shows only the commits introduced by
  the PR (equivalent to `git diff $(git merge-base BASE HEAD) HEAD`).

For a PR that was opened when `main` was at commit A, if 10 other PRs are
merged into `main` since then:
- Two-dot: includes all files changed by those 10 PRs + the PR's own changes
- Three-dot: includes only the PR's own changes

Since the gate requires ALL changed files to be under exactly one scope,
including unrelated merged changes almost guarantees `scoped=false`. This
makes an already low hit rate (2.7%) even lower for long-lived branches.

**Note:** `github.event.pull_request.base.sha` is specifically the commit SHA
that the base branch pointed to when the webhook fired (the PR open/sync
event), not the merge base. So the two-dot diff is technically correct in
showing "what would change if we merged" but is wrong for the purpose of
"what did this PR author change."

---

## Action Options

### Option A: Use `git merge-base` for three-dot semantics (recommended)

Replace the diff command:

```python
merge_base = subprocess.run(
    ["git", "merge-base", os.environ["BASE_SHA"], os.environ["HEAD_SHA"]],
    check=True, capture_output=True, text=True,
).stdout.strip()

diff = subprocess.run(
    ["git", "diff", "--name-only", merge_base, os.environ["HEAD_SHA"]],
    check=True, capture_output=True, text=True,
)
```

This gives the true PR diff regardless of how stale the branch is.

### Option B: Use three-dot syntax directly

```python
diff = subprocess.run(
    ["git", "diff", "--name-only",
     f"{os.environ['BASE_SHA']}...{os.environ['HEAD_SHA']}"],
    ...
)
```

Equivalent to Option A but more concise. Requires `fetch-depth: 0` (already
set).

### Option C: Switch to `dorny/paths-filter`

Replace the custom Python detector with `dorny/paths-filter`, which handles
base resolution, merge-base computation, and path matching automatically.
This is what `ci.yml` uses. Downside: the custom gear-scope discovery logic
would need to be reimplemented in paths-filter's config syntax, which may
not express the dynamic scope-root discovery.

### Option D: Use GitHub API to get PR files

Replace the git diff with a GitHub API call:
```
gh api repos/{owner}/{repo}/pulls/{pr}/files
```

The API always returns only the PR's own changed files, regardless of base
branch state. Downside: requires `GITHUB_TOKEN` permissions and has
pagination limits (max 3000 files per PR, but that's not a practical issue
here).
