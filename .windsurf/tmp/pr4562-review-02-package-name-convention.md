# PR #4562 Review — Comment #2: Package Name Convention (`cf-gears-$(GEAR)`)

**Reviewer:** MikeFalcon77
**File:** `Makefile` (GEAR_PKG, GEAR_SDK_FLAG, GEAR_PKGS)
**AI TODO:** "to confirm it's a valid concern. If yes - propose a robust
solution to fix the root cause. It must be fixed in a way that gears do not
differ in their naming convention in this project. You will need to investigate
what is the better option of doing that - either make a local per-project CI
or maybe suggest a new rule in cargo-gears. But I'm not sure cargo-gears
because other gears are not necessary coming from cf vendor and not necessary
to be published. So maybe local make check and CI is OK"
**Severity:** 🔴 **High** — `GEAR_PKG` and `GEAR_SDK_FLAG` silently resolve to wrong package names for BSS crates, examples, grouped gears, and all libs. `build` and `clippy` either fail or silently omit crates, giving false confidence.
**Verdict:** ✅ **Resolved** — `GEAR_PKG`, `GEAR_SDK_PKG`, and `GEAR_SDK_FLAG` have been removed.
`GEAR_PKGS` is now resolved via `cargo gears ls packages --scope-dirs gears,libs --filter '$(GEAR_NAME_REGEXP)' -f cargo-flags`,
which discovers packages by Cargo package name regex against `cargo metadata` — no convention assumptions.
Additionally, `cargo gears ls features` replaces the awk-based feature extraction, and
`cargo gears tools check-version` / `--check-version` replaces shell+Python version checks.
Issue [#4591](https://github.com/constructorfabric/gears-rust/issues/4591) tracks the separate rename of non-compliant crates to `cf-gears-` prefix.
cargo-gears enhancements: [cargo-gears#107](https://github.com/constructorfabric/cargo-gears/pull/107).

---

## MikeFalcon77's Concern

Two problems with deriving the package set by convention:

1. `cf-gears-$(GEAR)` is wrong for several crates:
   - `gears/bss/ledger/ledger` → package is `bss-ledger`, not `cf-gears-ledger`
   - `users-info` / `calculator` don't carry the `cf-gears-` prefix
2. The wildcard `$(wildcard gears/$(GEAR)/$(GEAR)-sdk)` only looks under
   `gears/<gear>/`, so for grouped gears (`bss/ledger`, `system/cluster`) and
   all of `libs/*`, the SDK crate **silently drops out** of build and clippy.

Suggestion: route `GEAR_PKGS` through `resolve_gear_deps.py` instead of
maintaining a second, weaker resolver.

## Investigation

### Confirmed: Valid concern

The Makefile derives packages as:

```makefile
GEAR_PKG ?= cf-gears-$(GEAR)
GEAR_SDK_PKG ?= $(GEAR_PKG)-sdk
GEAR_SDK_FLAG := $(if $(wildcard gears/$(GEAR)/$(GEAR)-sdk),-p $(GEAR_SDK_PKG))
GEAR_PKGS := -p $(GEAR_PKG) $(GEAR_SDK_FLAG)
```

Actual mismatches found in the workspace:

| GEAR= value | Derived GEAR_PKG | Actual package name |
|---|---|---|
| `ledger` | `cf-gears-ledger` | `bss-ledger` |
| `rate-provider` | `cf-gears-rate-provider` | `bss-rate-provider` |
| `users-info` | `cf-gears-users-info` | `users-info` (no prefix) |
| `calculator` | `cf-gears-calculator` | `calculator` (no prefix) |

The SDK wildcard `$(wildcard gears/$(GEAR)/$(GEAR)-sdk)` fails for:
- Grouped gears: `gears/bss/ledger/ledger-sdk` → path is not `gears/ledger/ledger-sdk`
- Libs: `libs/toolkit-db` → path is not `gears/toolkit-db/toolkit-db-sdk`

`resolve_gear_deps.py` already handles both correctly via `_find_gear_dir()`
(multi-root lookup) and `cargo metadata` (real package names). It is already
used for the `test GEAR=` target.

### Root cause analysis

The naming convention `cf-gears-<name>` is NOT universal in this project:
- BSS crates use `bss-<name>` prefix
- Example crates (`users-info`, `calculator`) use no prefix
- Some system crates use `cf-gears-<name>`

This is a **legitimate design choice** — not all gears are from the `cf`
vendor, and non-vendor gears should not be forced into the `cf-gears-` prefix.
A cargo-gears enforcement rule would be too restrictive.

---

## Action Options

### Option A: Route GEAR_PKGS through `resolve_gear_deps.py` (recommended)

Call `resolve_gear_deps.py` at Makefile evaluation time to get the correct
package names for `GEAR_PKG` and `GEAR_SDK_PKG`. This eliminates the
convention-based derivation. The script already does the right thing.

```makefile
# When GEAR is set, resolve the package list from cargo metadata
ifdef GEAR
  GEAR_PKGS := $(shell $(PYTHON) tools/scripts/resolve_gear_deps.py $(GEAR))
endif
```

This reuses the existing, tested resolver for all GEAR-scoped targets (not
just `test`). The `GEAR_PKG` / `GEAR_SDK_PKG` overrides from the CI workflow
(`classify_scope()`) would still work as explicit overrides when passed on the
command line.

### Option B: Use `resolve_gear_deps.py --names` to populate GEAR_PKG

Add a `--primary` flag to `resolve_gear_deps.py` that returns just the main
package name (not the full set). Use it to set `GEAR_PKG` correctly.

```makefile
GEAR_PKG := $(shell $(PYTHON) tools/scripts/resolve_gear_deps.py $(GEAR) --primary)
```

Then keep the existing `GEAR_SDK_PKG` and `GEAR_PKGS` derivation but from
the correct base.

### Option C: Add a gear-manifest field for package names

Add a `package_name:` field to `e2e.yaml` or create a `gear.toml` manifest
per gear that declares the crate names. The Makefile reads this instead of
using conventions. Adds a maintenance burden for each new gear.

### Option D: Enforce `cf-gears-` naming for all in-repo gears

Rename `bss-ledger` → `cf-gears-bss-ledger`, `users-info` → `cf-gears-users-info`,
etc. This is the most invasive option and may not be desirable since BSS crates
and example crates are intentionally named differently. This is NOT recommended
per the user's note that gears are not necessarily from cf vendor.
