# PR #4562 Review — Comment #5: No sccache, Per-Scope Cache Buckets

**Reviewer:** MikeFalcon77
**File:** `.github/workflows/gear-scoped-ci.yml` (scoped-ci job, cache step)
**AI TODO:** "investigate if it's a valid concern carefully, suggest a fix"
**Severity:** 🟠 **Medium** — The scoped CI "fast path" will likely be slower than full CI without sccache. Per-scope cache keys waste GHA cache quota by duplicating the shared dependency graph ~50×. Not a correctness issue but defeats the performance purpose of the workflow.
**Verdict:** ✅ **Resolved** — Added sccache (`mozilla-actions/sccache-action` + `RUSTC_WRAPPER`
probe) and changed cache key from per-scope (`pr-gear-scoped-<scope_id>-<toolchain>`) to shared
(`pr-gear-scoped-<toolchain>`). Implemented Option A. See commit `7b6b04b3`.

---

## MikeFalcon77's Concern

1. **No sccache** — every Rust job in `ci.yml` sets up
   `mozilla-actions/sccache-action` and probes it into `RUSTC_WRAPPER`. Without
   it, the "fast path" is a cold build of the whole dependency graph and
   plausibly ends up **slower** than the lanes it's meant to short-circuit.

2. **Per-scope cache buckets** — `shared-key` includes `scope_id`, so each gear
   gets its own cache bucket and none of them share the common dependency build:
   ```yaml
   shared-key: pr-gear-scoped-${{ needs.detect.outputs.scope_id }}-${{ hashFiles('rust-toolchain.toml') }}
   ```

## Investigation

### Confirmed: Valid concern (both points)

**Point 1 — sccache missing:**

Full CI (`ci.yml`) has this pattern in every Rust job:
```yaml
- name: Install sccache
  uses: mozilla-actions/sccache-action@...

- name: Probe sccache & set RUSTC_WRAPPER
  run: |
    if sccache --start-server >/dev/null 2>&1; then
      echo "RUSTC_WRAPPER=sccache" >> "$GITHUB_ENV"
      echo "SCCACHE_GHA_ENABLED=true" >> "$GITHUB_ENV"
    fi
```

`gear-scoped-ci.yml` has none of this. Without sccache, every invocation
compiles all transitive dependencies from scratch. For a gear like
`file-parser` which depends on `toolkit`, `toolkit-db`, `sea-orm`, etc., this
is a full workspace dependency build (~15-25 min on ubuntu-latest from cold)
vs. ~3-5 min with sccache warm.

**Point 2 — per-scope cache keys:**

The `shared-key` is `pr-gear-scoped-<scope_id>-<toolchain_hash>`. Since
`scope_id` is unique per gear (e.g. `gears-file-parser`, `libs-toolkit-db`),
each gear's cache is completely isolated. The common dependency graph
(`tokio`, `sea-orm`, `toolkit`, etc.) is rebuilt and cached separately for
every gear. On a project with ~50 scopes, this means ~50 independent cache
buckets with massive duplication of the same compiled dependencies.

In contrast, `ci.yml` uses a small number of shared keys
(`pr-clippy-<toolchain>`, `pr-<os>-<toolchain>`) so all jobs share the common
dependency compilation.

---

## Action Options

### Option A: Add sccache + use a shared cache key (recommended)

Add sccache setup (copy from ci.yml) and change the cache key to a shared
one that all gear-scoped runs share:

```yaml
- name: Install sccache
  uses: mozilla-actions/sccache-action@fc920bf0ec8de6ee65d409111f7ec508035751ba

- name: Probe sccache & set RUSTC_WRAPPER
  run: |
    if sccache --start-server >/dev/null 2>&1; then
      echo "RUSTC_WRAPPER=sccache" >> "$GITHUB_ENV"
      echo "SCCACHE_GHA_ENABLED=true" >> "$GITHUB_ENV"
    fi
```

And change the cache key:
```yaml
shared-key: pr-gear-scoped-${{ hashFiles('rust-toolchain.toml') }}
```

This way all gear-scoped runs share one cache bucket for the common
dependency graph, and sccache handles incremental object reuse.

### Option B: Add sccache only, keep per-scope cache keys

Add sccache but keep the per-scope `shared-key`. sccache's own GHA backend
cache is content-addressed (hash of compilation inputs), so it naturally
deduplicates across gear scopes even if the Cargo target cache is per-scope.
This is simpler but the target/ cache remains bloated.

### Option C: Drop rust-cache entirely, rely on sccache alone

Remove the `Swatinem/rust-cache` step and use only sccache. sccache stores
compiled objects in a content-addressed store and doesn't need the target/
directory to persist between runs. This is the simplest approach and avoids
the cache-key design issue entirely. Some CI setups use this pattern
successfully.

### Option D: Enable cache-targets (with sccache)

Set `cache-targets: true` alongside sccache with a shared key. This caches
the target/ directory so even without sccache, subsequent runs for ANY gear
benefit from the pre-compiled dependency tree. Combined with sccache this
gives the fastest possible cold-start for any gear scope.
