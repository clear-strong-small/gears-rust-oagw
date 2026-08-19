# PR #4562 Review — Comment #1: Scope → Server-Feature Mapping

**Reviewer:** MikeFalcon77
**File:** `Makefile` (GEAR_SERVER_OPTIONAL_FEATURES)
**AI TODO:** "to confirm it's a valid concern. If yes - suggest a fix for it"
**Severity:** 🔴 **High** — `make all GEAR=<scope>` fails outright for ~41/53 scopes. Any gear without a matching example-server feature causes a hard Cargo build error. Blocks the stated purpose of gear-scoped local dev and CI.
**Verdict:** ✅ **Resolved** — Server-dependent targets (`run`, `openapi`, `.example-server-build`)
now check `GEAR_HAS_SERVER_FEATURE` (validated against `EXAMPLE_SERVER_ALL_FEATURES` resolved
via `cargo gears ls features`). When `GEAR` has no matching server feature (e.g. `toolkit-db`,
`ledger`), these targets print `SKIP: GEAR=... has no example-server feature` and exit cleanly.
Library-safe targets (`fmt`, `clippy`, `test`, `build`) continue to work for all gears.
Implemented Option A from the action options below.

---

## MikeFalcon77's Concern

The gear-scoped CI passes the scope directory name straight through as a Cargo
feature of `cf-gears-example-server`, but for most scopes that feature **does
not exist**.

Specifically, the example-server's feature list is missing:
`toolkit-db`, `cluster`, `event-broker`, `quota-enforcement`,
`license-resolver`, `approval-service`, `llm-gateway`, `model-registry`,
`serverless-runtime`, `infrastructure-resource-manager`,
`simple-resource-registry`.

Additionally, `gears/bss/ledger` resolves to `ledger`, but the actual feature
is `bss-ledger`.

So `.example-server-build` and `openapi` fail with "package does not have
feature X", making `make all GEAR=<scope>` fail outright. Of 53 scope roots,
roughly 12 work. Every `libs/*` scope fails.

## Investigation

### Confirmed: Valid concern

The Makefile derives the server feature from `GEAR` directly:

```makefile
GEAR_SERVER_OPTIONAL_FEATURES := $(filter-out $(GEAR_SERVER_ALWAYS_LINKED),$(GEAR))
GEAR_SERVER_FEATURES ?= $(GEAR_SERVER_OPTIONAL_FEATURES),$(GEAR_SERVER_BASE_FEATURES)
```

The example-server (`apps/cf-gears-example-server/Cargo.toml`) has only these
optional gear features:

```
file-parser, simple-user-settings, nodes-registry, resource-group, grpc-hub,
oagw, credstore, fips, users-info-example, oop-example, single-tenant,
static-tenants, static-authn, static-authz, tr-authz, tenant-resolver-rg,
static-credstore, mini-chat, chat-engine, k8s, otel, account-management,
static-idp, file-storage, bss-rate-provider, bss-ledger, usage-collector,
timescaledb-usage-collector
```

Any `GEAR=` value that doesn't match one of these features will fail at Cargo
build time. This affects:

- **All `libs/*` scopes** — no corresponding features exist (e.g. `toolkit-db`,
  `toolkit-http`, `toolkit-canonical-errors`, etc.)
- **System gears without features** — `cluster`, `event-broker`,
  `quota-enforcement`, etc.
- **Name mismatches** — `GEAR=ledger` → feature would be `ledger`, but the
  actual feature is `bss-ledger`

Repro: `make all GEAR=toolkit-db` → Cargo error "package does not have feature
toolkit-db".

---

## Action Options

### Option A: Validate feature at Makefile time, skip server targets if invalid (recommended)

Use `cargo gears ls features --manifest $(EXAMPLE_SERVER_MANIFEST)` (already
available) to check whether `$(GEAR)` is a valid server feature. If not,
set `GEAR_SERVER_FEATURE_ARGS` to empty and skip server-dependent targets
(run, openapi, e2e-local) automatically.

```makefile
# Resolve whether GEAR is a valid example-server feature.
ifdef GEAR
  EXAMPLE_SERVER_HAS_FEATURE := $(filter $(GEAR),$(EXAMPLE_SERVER_ALL_FEATURES))
  GEAR_SERVER_OPTIONAL_FEATURES := $(if $(EXAMPLE_SERVER_HAS_FEATURE),$(GEAR),)
endif
```

Server-dependent targets (`.example-server-build`, `openapi`, `run`) would
guard on `EXAMPLE_SERVER_HAS_FEATURE` being non-empty. Library-safe targets
(fmt, clippy, test, build) continue to work for all gears.

This is the minimal fix — no new fields, no new tools, uses data already
available via `EXAMPLE_SERVER_ALL_FEATURES`.

### Option B: Separate `make all` into library-safe and server-dependent targets

Split the `check` / `all` target so that when `GEAR=` is set for a lib or
non-server gear, only library-safe targets (fmt, clippy, test) run, and
server-dependent targets (example-server build, openapi, e2e-local) are
skipped automatically.

```makefile
# Library-safe targets — always safe with GEAR=
gear-check: fmt clippy test

# Full targets — only when GEAR has a server feature
all: gear-check $(if $(EXAMPLE_SERVER_HAS_FEATURE),test-sqlite e2e-local openapi,)
```

This is more explicit but changes the `make all` contract.

### Option C: Add a `server_feature` field to e2e.yaml manifests

Each gear with an e2e.yaml manifest could declare its own `server_feature:`
mapping. The Makefile / CI would use this instead of `$(GEAR)`. Gears
without manifests or without a `server_feature` field skip server-related
targets automatically. Adds per-gear maintenance burden.
