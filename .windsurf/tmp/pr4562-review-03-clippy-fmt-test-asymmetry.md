# PR #4562 Review — Comment #3: Clippy/Fmt vs Test Asymmetry

**Reviewer:** MikeFalcon77
**File:** `Makefile` (clippy and fmt targets)
**AI TODO:** "Investigate if it's valid concern carefully. I'm not sure here
because clippy and fmt validates code that is not supposed to be in scope of
current change. If so, a developer/CI will need to run it's own test GEAR=xxx
checks"
**Severity:** 🟡 **Low** — Partially valid but edge-case only. `fmt` gains nothing from reverse deps. `clippy` on reverse deps catches rare propagated-lint scenarios. Full CI still runs and covers both. Only becomes relevant if gear-scoped CI replaces full CI.
**Verdict:** ✅ **Partially resolved** — `fmt` and `clippy` now use `GEAR_PKGS` (resolved via
`cargo gears ls packages --filter GEAR_NAME_REGEXP`), which includes all gear packages
(main + SDK) but intentionally NOT reverse deps. `test` uses the same command with
`--include-rdeps` for full reverse-dep coverage. The asymmetry is now **intentional and
documented**: fmt/clippy check the gear's own crates; test checks consumers too.
Full CI still runs for complete coverage.

---

## MikeFalcon77's Concern

`test GEAR=` runs the gear's crates **plus their transitive reverse deps** via
`resolve_gear_deps.py`, but `clippy` (and `fmt`) run on `-p $(GEAR_PKG)` only.

This asymmetry matters for exactly the scopes the new workflow targets: a
change in `libs/toolkit-db` that breaks a lint in a downstream gear passes
the scoped clippy lane and only surfaces in full CI. Since the goal is
eventually to replace full CI for these PRs, clippy and fmt should use the
same resolved package set as test.

## Investigation

### Current behavior

```makefile
# fmt: single package only
fmt:
    $(if $(GEAR),cargo fmt -p $(GEAR_PKG) --check,cargo fmt --all --check)

# clippy: single package only
clippy:
    # ...
    cargo clippy -p $(GEAR_PKG) $(GEAR_CLIPPY_ARGS)

# test: transitive reverse deps via resolve_gear_deps.py
test:
    GEAR_SCOPE=$$($(PYTHON) tools/scripts/resolve_gear_deps.py $(GEAR))
    cargo nextest run $$GEAR_SCOPE ...
```

### Is this a valid concern?

**Partially valid, with nuance.**

MikeFalcon77's argument holds in a narrow scenario: if you change a type
signature in `libs/toolkit-db` and a downstream gear uses the old type in a way
that triggers a Clippy lint, scoped clippy on `toolkit-db` alone won't catch it.

**However, your intuition is also correct:**

- `cargo fmt` is syntactic and per-file — formatting of downstream crates is
  NOT affected by upstream changes. Running `fmt` on reverse deps adds cost
  with zero value.
- `cargo clippy` on reverse deps catches "lint breakage propagated downstream,"
  but this scenario is quite rare. Most clippy lints are local to the crate
  being edited. The typical case where this matters is a type/API change that
  causes unused-import or dead-code warnings in consumers.
- For the **current scope** of this workflow (experimental, additive, full CI
  still runs), the asymmetry is acceptable. It only becomes a real issue if/when
  gear-scoped CI replaces full CI.

### Key distinction

- **fmt**: No value in running on reverse deps. Format is per-file.
- **clippy**: Minor value in running on reverse deps. Most lint issues are local.
  The full CI catches anything missed.
- **test**: Correct to run on reverse deps — a breaking change in a lib can
  cause test failures in consumers.

---

## Action Options

### Option A: Keep current behavior (recommended) ✅ SELECTED

`fmt` and `clippy` use `GEAR_PKGS` (gear's own packages: main + SDK).
`test` uses `cargo gears ls packages --include-rdeps` (gear + reverse deps).
The asymmetry is intentional:
- **fmt**: Per-file, no value from reverse deps.
- **clippy**: Most lints are local. Edge-case propagated-lint scenarios are
  caught by full CI.
- **test**: Reverse deps are critical — a breaking change causes test failures
  in consumers.

### Option B: Extend clippy to include reverse deps

Route clippy through the same `--include-rdeps` resolver as test:

```makefile
clippy:
    GEAR_SCOPE=$$(cargo gears ls packages --scope-dirs gears,libs \
      --filter '$(GEAR_NAME_REGEXP)' --include-rdeps -f cargo-flags)
    cargo clippy $$GEAR_SCOPE $(GEAR_CLIPPY_ARGS)
```

Only worth doing if gear-scoped CI is positioned as a full CI replacement.
