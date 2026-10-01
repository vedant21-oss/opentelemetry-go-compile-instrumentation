# 6. Semconv Telemetry Contract Organization

Date: 2026-10-01

## Status

Proposed

## Context

[PR #696](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/pull/696)
added an OpenTelemetry Weaver registry under `schemas/otelc/`. It is the
machine-readable contract of the telemetry `otelc`'s instrumentations
emit: attributes and metrics that exist upstream are referenced with
`ref:`, and anything library-specific that upstream does not cover is
declared locally with `id:`. `make lint-schema` validates the result
with `weaver registry check --future`.

That PR established the registry but not the rules for growing it. Two
questions were left open: where contract files live as the set grows,
and how the upstream semconv version is managed across instrumentations.

Weaver constrains both answers:

- one `registry_manifest.yaml` per registry;
- one directory tree, scanned recursively, for all group files;
- no merging of independent registries;
- no validating one registry against several upstream semconv versions
  at once.

[ADR-0004](0004-instrumentation-ownership-and-compatibility.md) makes
each instrumentation responsible for its own telemetry, and
[ADR-0005](0005-import-driven-instrumentation-selection.md) treats
instrumentations as ordinary Go modules. Neither says where the
declared contract for those modules belongs.

## Decision

**One tree.** All telemetry contract files live under
`schemas/otelc/groups/`, one YAML file per instrumentation family
rather than per module. Today that is 19 files covering 32
instrumentation modules: `openai.yaml` covers `openai-go` v1, v2 and
v3; `grpc.yaml` covers client and server; `logs.yaml` covers `log`,
`log/slog`, `logrus` and `zap`. Sub-directories are permitted for
logical grouping as the set grows, since Weaver scans recursively.

**One upstream version.** The project pins a single upstream semconv
version in `.semconv-version` at the repository root, currently
v1.37.0. `registry_manifest.yaml` declares the matching dependency, and
a guard in the `Makefile` fails the build when the two disagree.
Upgrades are atomic across every instrumentation; there is no
per-instrumentation pin.

**Declaring is part of adding.** An instrumentation that emits
telemetry contributes a group file declaring it. The contract and the
code that emits it are added together, not in separate changes.

**Code generation is out of scope.** Generating Go types from the
registry is Tier 2/3 of
[#728](https://github.com/open-telemetry/opentelemetry-go-compile-instrumentation/issues/728)
and depends on this layout being settled first. It gets its own ADR.

## Consequences

The registry stays a single Weaver-validatable unit, which is what
`weaver registry check --future` requires. Adding an instrumentation
means adding one file to a known place.

Grouping by family rather than by module keeps the file count close to
the number of libraries rather than the number of Go modules, so the
three `openai-go` major versions share one declaration instead of
diverging into three.

A single pinned version means an upstream semconv bump is one change
affecting every instrumentation at once. That is the trade for Weaver's
inability to validate against several versions, and it makes the bump
reviewable in one place. It also means an instrumentation cannot adopt
a newer attribute ahead of the rest of the project. Emitting two
semconv versions at once is incompatible with a registry pinned to
exactly one, so dual-emit, if it is wanted, has to be solved in the
code layer and needs its own decision.

Declaring telemetry is a convention today, not a check. No CI job
verifies that an instrumentation which emits telemetry has a group
file, so the rule holds by review. All 32 current modules are covered,
but nothing stops the 33rd from landing without a declaration. Closing
that gap is worth a follow-up.

Colocating contract files next to the instrumentation code they
describe was considered and rejected: Weaver needs one tree, so
colocation would require an aggregation step to assemble a registry
before every validation, adding a build stage and a new way for the
contract to be wrong.
