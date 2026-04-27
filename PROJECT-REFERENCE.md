# Eidolon Systems — Project Reference

This document holds three sections:

1. **Master Spec and Architecture** — the canonical reference for the project. Everyone reads this. Drop into the `eidolon-system` umbrella repo as `README.md` plus the spec documents under `docs/specs/`.
1. **Per-Repo Charters** — one charter per repository in the suite. Paste the relevant charter at the start of each Claude Code session for that repo.
1. **Cross-Repo Conventions** — decisions that should hold identically across all repos. Duplicate into each repo or reference the authoritative copy in `eidolon-system`.

-----

# Section 1: Master Spec and Architecture

## Overview

Eidolon Systems comprises a suite that provides command-line call interception, policy enforcement, and observability for skill-based AI integrations and other agentic tooling. The suite addresses a gap that opens as Anthropic and similar platforms move toward skills (which execute arbitrary code) over MCP (which presents typed function calls): no standardized audit surface exists for what those skills actually invoke at the syscall and binary level.

The architecture treats interfaces as the actual product. Each component remains agnostic of the others — anyone can swap in alternatives at any layer.

**Specifications (the contracts):**

- **Eidolon Event Format** — versioned spec, JSON Schema, conformance suite for events emitted by call observers
- **Eidolon Policy Format** — versioned spec, JSON Schema, canonical-form rules, signature envelope for policy artifacts

**Reference implementations:**

- **Eidoshim** — multi-call shim binary that intercepts CLI invocations, applies signed policy, and emits structured events
- **Eidocon** — policy authoring, signing, and distribution toolchain
- **Eidolens** — observability layer that consumes events and presents them to operators

All seven repositories live under the `khorum-oss` GitHub organization. JVM artifacts publish to **Reliquary** (DigitalOcean Spaces). Hosted documentation lives at `eidolon.khorum-oss.org`. Security advisories follow the existing Khorum-OSS Software Assurance scheme with the prefix `KSA-EIDO-YYYY-NNNN`.

## Architecture

### Eidoshim — interception and enforcement

A single GraalVM-compiled native binary, symlinked under many names (`curl`, `git`, `npm`, `python`, `wget`, …) à la busybox. The binary reads `argv[0]` to determine which command it impersonates, consults loaded policy, and either forwards to the real binary (resolved via a fresh PATH lookup that excludes the shim directory, proxying stdio and capturing timing/exit data) or denies with a structured error and a logged event.

Native compilation gets startup under 30ms — close enough to the real binary that latency stays imperceptible.

**Critical caveat:** Eidoshim provides observability and soft policy, not a sandbox. Anything calling binaries by absolute path, using shell builtins, or invoking syscalls directly walks past it. Production deployments should pair Eidoshim with `bubblewrap`, `landlock`, or container isolation. The Eidoshim audit trail complements those layers — well-behaved skills get policy enforcement, all skills get the audit trail.

### Eidocon — policy lifecycle

Handles author → sign → distribute → revoke. Treats HOCON and YAML as equal-weight authoring formats; both compile to a single canonical form (RFC 8785 canonical JSON) which gets signed.

Components:

- **Schema** — JSON Schema document describing all policy types (network, filesystem, exec, env). Single source of truth.
- **Authoring formats** — HOCON, YAML, and the Konstellation-generated Kotlin DSL. All three carry identical expressive power; all three round-trip through canonical form bit-for-bit.
- **Compiler** — parses any authoring format to an AST, validates against the schema, emits canonical JSON.
- **Signer** — signs the canonical JSON using a scoped signing key (PGP via 1Password, matching the existing Konstellation flow).
- **Distribution** — pushes signed bundles to DigitalOcean Spaces (Reliquary) or OCI registry; Eidoshim pulls or receives.
- **Revocation** — maintains a revocation list signed by the trust root.

### Eidolens — observability

Consumes the structured event stream Eidoshim produces (JSONL plus optional OpenTelemetry spans) and presents it. Architecture mirrors the Vaultara pattern: Spring Boot WebFlux backend, SvelteKit frontend.

Eidolens consumes from any source emitting the canonical event format — Eidoshim, application telemetry, syslog, journald, eBPF traces — making it useful even for environments that don’t run the shim layer.

## Trust Model

Three layers, kept deliberately separate:

**Trust root.** A small set of public keys baked into Eidoshim at install time. Lives in HSMs or 1Password. Changing the trust root requires reinstalling or going through a separate, more privileged key — never a runtime config change.

**Signing keys.** The keys that sign day-to-day policy. Scoped: a `network-policy-signer` whose signatures Eidoshim only honors on policies under `network/*`, a `filesystem-policy-signer` for `fs/*`, etc. SSH CA-style principal restrictions are the model. Compromise of one scoped key doesn’t authorize arbitrary syscalls.

**Revocation list.** Signed by the trust root, distributed alongside policy. Eidoshim checks every signing key against it before honoring a signature.

High-impact policy scopes can require N-of-M signatures rather than a single signer (Vaultara four-eyes pattern).

-----

## Eidolon Event Format Specification — v1.0.0

### Part 1: Scope and Versioning

#### What the spec defines

The Eidolon Event Format describes a wire format for events emitted by tools that observe command-line invocations or other agentic actions. A conforming **producer** emits events matching the format. A conforming **consumer** accepts events matching the format.

The spec covers:

- The set of event types and what each represents semantically
- The common envelope every event carries
- The fields available on each event type
- The serialization (JSON, with JSONL as the recommended transport)
- The versioning rules and compatibility guarantees
- The conformance criteria for producers and consumers

#### What the spec deliberately does not define

- **Transport.** JSONL over stdout, HTTP POST, Kafka, OTLP wrapping, syslog — the spec describes the payload, not how it gets from producer to consumer.
- **Storage and indexing.** A consumer chooses how to store events.
- **The producer’s mechanism.** Eidoshim uses a multi-call binary; an eBPF producer would observe syscalls; an in-process library would hook function calls. The format doesn’t care.
- **The decision engine.** When an event records a policy decision, the format describes what the decision was, not how the engine reached it.
- **Retention, aggregation, alerting.** Pure consumer concerns.
- **Authentication between producer and consumer.** Out of scope; transport-layer concern.

#### Conformance levels

**Producer conformance (permissive).** A conforming producer emits events whose JSON structure validates against the published JSON Schema for each event type, carries all required envelope fields, and uses field values consistent with their documented semantics. A producer is **not** required to emit every event type or fill every optional field. An eBPF producer probably can’t fill `parent_command` reliably; that’s fine, the field stays optional.

**Consumer conformance (strict).** A conforming consumer accepts any valid event, ignores unknown optional fields without error, handles older minor versions gracefully, and refuses (with a clear error) events whose major version it doesn’t support. Consumers MAY require specific optional fields for their own functionality but cannot reject events that fail to provide them — they degrade gracefully.

This asymmetry makes the format useful: producers emit what they can, consumers handle what they get.

#### Versioning

Semantic versioning applied to the format itself. Version appears in two places:

**Spec version** — the version of the specification document. Format `MAJOR.MINOR.PATCH`.

**Event schema version** — appears in every event payload as `schema_version`. Same `MAJOR.MINOR.PATCH` shape. Consumers check this at runtime.

Compatibility rules:

- **Patch bump** (1.0.0 → 1.0.1) — clarifications, typo fixes, no wire changes.
- **Minor bump** (1.0.x → 1.1.0) — additive only. New optional fields, new event types, new enum values.
- **Major bump** (1.x.x → 2.0.0) — breaking changes.

Producers SHOULD emit the highest schema version they implement. Consumers MUST accept any minor version within a major they support.

#### Discovery — producer descriptor

A consumer that wants to know what a producer emits can request a `producer_descriptor` document. Optional but useful for tools building UIs around the data.

```json
{
  "name": "eidoshim",
  "version": "0.3.1",
  "spec_version": "1.0.0",
  "event_types": ["call_attempted", "call_allowed", "call_denied", "call_completed", "call_errored", "policy_load_succeeded", "policy_load_failed", "producer_started", "producer_stopped"],
  "optional_fields_filled": ["call.parent", "call.cwd", "call.env_keys", "decision.decision_latency_ms"],
  "extensions": ["x_eidoshim_path_lookup_ms"]
}
```

#### Extensions

Fields prefixed `x_` count as extensions. Producers MAY add them; consumers MUST tolerate them; the spec MUST NOT define them. If an extension proves broadly useful, it gets promoted into the spec proper at a minor bump, dropping the `x_` prefix.

#### Casing convention

All wire-format field names use **snake_case**, matching OpenTelemetry, CloudEvents, and OCSF conventions. Implementations translate at their serialization boundary as needed (Jackson’s `PropertyNamingStrategies.SnakeCaseStrategy` for Kotlin/JVM domain models in camelCase). Application APIs that consume events MAY expose data to their own clients (frontends, SDKs) in whatever casing suits — the spec only governs the wire format.

#### Unknown event types

A consumer encountering an unknown event type MUST log it (somewhere — implementation-defined) rather than silently dropping. This prevents observability gaps when producers and consumers version-skew.

### Part 2: Event Types and Common Envelope

#### Event types

**Call-lifecycle events** describe the observation of a single command invocation:

- `call_attempted` — producer observed an invocation; pre-decision (if a decision happens at all)
- `call_allowed` — a decision engine permitted the call to proceed
- `call_denied` — a decision engine refused the call
- `call_completed` — call finished, regardless of exit status
- `call_errored` — the producer itself failed to forward or observe completion

**Operational events** describe the producer’s own state:

- `producer_started` — producer process came up
- `producer_stopped` — graceful shutdown
- `policy_load_succeeded` — relevant only to enforcing producers; new policy loaded successfully
- `policy_load_failed` — relevant only to enforcing producers; policy load failed

A conforming producer MUST emit at least one event type but isn’t required to emit any specific type. An audit-only eBPF producer might emit only `call_completed`.

#### The common envelope

Every event carries this structure at the top level:

```json
{
  "schema_version": "1.0.0",
  "event_id": "01HXYZ7K9V3QJ8M2N4P5R6S7T8",
  "event_type": "call_attempted",
  "timestamp": "2026-04-26T14:23:01.123456789Z",
  "producer": {
    "name": "eidoshim",
    "version": "0.3.1",
    "instance_id": "01HXY1234567890ABCDEFGHIJK"
  },
  "host": {
    "id": "hestia-prod-01"
  }
}
```

|Field                 |Required|Type             |Notes                                                             |
|----------------------|--------|-----------------|------------------------------------------------------------------|
|`schema_version`      |REQUIRED|string (semver)  |Spec version this event conforms to                               |
|`event_id`            |REQUIRED|string (ULID)    |Globally unique per event; ULID for time-sortable lex order       |
|`event_type`          |REQUIRED|string (enum)    |One of the documented types                                       |
|`timestamp`           |REQUIRED|string (RFC 3339)|UTC; nanosecond precision recommended                             |
|`producer.name`       |REQUIRED|string           |Short identifier (`"eidoshim"`, `"ebpf-tracer"`)                  |
|`producer.version`    |REQUIRED|string           |Producer’s own version                                            |
|`producer.instance_id`|OPTIONAL|string (ULID)    |Distinguishes multiple instances; generated at producer startup   |
|`host`                |REQUIRED|object           |Producers without meaningful host context emit `{"id": "unknown"}`|
|`host.id`             |REQUIRED|string           |Opaque host identifier                                            |

`host.id` MAY use the sentinel `"unknown"` when the producer cannot determine a meaningful value (e.g., serverless contexts).

Producers and consumers MAY add `x_*` fields under `host` for extra context (kernel version, container ID, k8s pod name).

#### What the common envelope deliberately omits

- **No `skill_id` or `session_id` at the envelope level.** Those count as call-lifecycle concepts; they live in the call envelope.
- **No `severity` or `level`.** The event type already encodes severity semantically.

#### Operational events

Operational events use the common envelope and add type-specific fields. `producer_started` example:

```json
{
  "schema_version": "1.0.0",
  "event_id": "01HXYZ...",
  "event_type": "producer_started",
  "timestamp": "2026-04-26T14:00:00.000000000Z",
  "producer": { "name": "eidoshim", "version": "0.3.1", "instance_id": "01HXY..." },
  "host": { "id": "hestia-prod-01" },
  "config_fingerprint": "sha256:a1b2c3..."
}
```

`config_fingerprint` REQUIRED on `producer_started` and `policy_load_succeeded` (events where config gets established or changed) and OPTIONAL on others.

### Part 3: Call Envelope and Identity Correlation

#### The call envelope

Every call-lifecycle event carries a `call` object:

```json
{
  "call": {
    "call_id": "01HXYZ7K9V3QJ8M2N4P5R6S7T8",
    "session_id": "01HXY1234567890ABCDEFGHIJK",
    "skill_id": "anthropic.docx@2.1.0",
    "command": "curl",
    "command_path": "/usr/bin/curl",
    "argv": ["curl", "-X", "POST", "https://api.example.com/v1/things"],
    "argv_redacted": false,
    "argv_truncated": false,
    "cwd": "/workspace/project",
    "env_keys": ["HOME", "PATH", "ANTHROPIC_API_KEY"],
    "parent": {
      "pid": 4823,
      "command": "python3"
    }
  }
}
```

|Field                |Required                  |Type            |Notes                                                                                      |
|---------------------|--------------------------|----------------|-------------------------------------------------------------------------------------------|
|`call.call_id`       |REQUIRED                  |string (ULID)   |Unique per call lifecycle; shared across attempted/allowed/denied/completed/errored        |
|`call.session_id`    |OPTIONAL                  |string (ULID)   |Groups calls in a single skill invocation                                                  |
|`call.skill_id`      |OPTIONAL                  |string          |Recommended format `<namespace>.<n>@<version>`; absence and `"unknown"` mean the same thing|
|`call.command`       |REQUIRED                  |string          |Basename of `argv[0]`                                                                      |
|`call.command_path`  |OPTIONAL                  |string          |Full path when producer can determine it                                                   |
|`call.argv`          |REQUIRED                  |array of strings|Full argv including `argv[0]`; preserves length even with redaction                        |
|`call.argv_redacted` |REQUIRED                  |boolean         |True if any element got replaced by redaction policy                                       |
|`call.argv_truncated`|OPTIONAL                  |boolean         |True if any element got truncated due to soft size cap; truncated elements end with `…`    |
|`call.cwd`           |OPTIONAL                  |string          |Absolute path; producers without meaningful cwd omit                                       |
|`call.env_keys`      |OPTIONAL                  |array of strings|Names only, never values; sorted lexically                                                 |
|`call.parent`        |OPTIONAL                  |object          |Immediate parent only                                                                      |
|`call.parent.pid`    |REQUIRED if parent present|integer         |Process ID                                                                                 |
|`call.parent.command`|OPTIONAL                  |string          |Basename when determinable                                                                 |

Deeper ancestor chains, when a producer wants to record them, go in `x_ancestors` per producer convention.

#### Skill identity

Three identification mechanisms producers MAY use:

**Environment variable.** The convention `EIDOLON_SKILL_ID` carries the skill identifier. Simplest mechanism, easiest to spoof — fine for cooperative environments.

**Process tree inspection.** The producer walks up the parent chain looking for a known marker process whose command line or environment encodes the skill identity. More robust but requires kernel/process visibility.

**Attestation.** A signed token issued by the skill runtime that the producer validates. Strongest, most complex. Out of scope for spec v1; reserved as a future extension.

A producer supporting multiple identification mechanisms SHOULD record which one it used in `x_skill_id_source` so consumers can weight the trust appropriately.

#### Identity correlation across events

- **`event_id`** — unique per event.
- **`call_id`** — unique per call lifecycle. Multiple events share it.
- **`session_id`** — unique per skill invocation. Many calls share it.

Consumers building lifecycle views query by `call_id`. Consumers building skill-behavior views query by `session_id`. Consumers doing forensic replay sort by `(timestamp, event_id)` for total ordering.

Producers MUST generate `call_id` fresh for every call, even when observing the same logical call multiple times.

#### Soft size cap

The spec recommends a soft cap of 64KB per event. Truncation rules:

- `argv` elements truncated mid-string get `…` suffix
- `call.argv_truncated` gets set true when any truncation occurs
- `decision.reason` truncated similarly with trailing `…`

Producers SHOULD respect the soft cap. Hard limits stay consumer-side; consumers MAY reject oversize events but MUST document their threshold.

### Part 4: Per-Event-Type Fields

#### `call_attempted`

Adds nothing beyond the common envelope and call envelope. The event itself signals: a call got observed at this timestamp, here’s what it was, no decision yet.

#### `call_allowed` and `call_denied`

Both add a `decision` object. Structure stays identical between the two; the `event_type` encodes the outcome.

```json
{
  "event_type": "call_allowed",
  "...": "...",
  "call": { },
  "decision": {
    "policy": {
      "policy_id": "khorum-oss/network-allowlist",
      "policy_version": "2026.04.20-1",
      "signature_fingerprint": "sha256:a1b2c3..."
    },
    "match_rule": "network/anthropic-docs",
    "reason": "Host matches allowlist entry api.anthropic.com",
    "decision_latency_ms": 0.34
  }
}
```

|Field                                  |Required|Type  |Notes                                                                              |
|---------------------------------------|--------|------|-----------------------------------------------------------------------------------|
|`decision.policy.policy_id`            |REQUIRED|string|Stable identifier across versions                                                  |
|`decision.policy.policy_version`       |REQUIRED|string|Specific version active at decision time                                           |
|`decision.policy.signature_fingerprint`|OPTIONAL|string|Hash of signature on policy bundle                                                 |
|`decision.match_rule`                  |REQUIRED|string|Specific rule within the policy                                                    |
|`decision.reason`                      |REQUIRED|string|Human-readable, display-safe (no markup, PII, secrets); SHOULD stay under 500 chars|
|`decision.decision_latency_ms`         |OPTIONAL|number|Decision timing for performance debugging                                          |

#### `call_completed`

```json
{
  "event_type": "call_completed",
  "...": "...",
  "call": { },
  "completion": {
    "exit_code": 0,
    "duration_ms": 234,
    "stdout_bytes": 1248,
    "stderr_bytes": 0,
    "signal": null
  }
}
```

|Field                    |Required|Type          |Notes                                                                          |
|-------------------------|--------|--------------|-------------------------------------------------------------------------------|
|`completion.exit_code`   |REQUIRED|integer       |Process exit code; full int32 range                                            |
|`completion.duration_ms` |REQUIRED|number        |Wall-clock; sub-millisecond precision when available                           |
|`completion.stdout_bytes`|OPTIONAL|integer       |Sizes only; content stays opt-in via separate mechanism                        |
|`completion.stderr_bytes`|OPTIONAL|integer       |Same                                                                           |
|`completion.signal`      |OPTIONAL|string or null|Signal name (`"SIGTERM"`) when terminated by signal; null/absent on normal exit|

Signal-terminated processes use `call_completed` with non-null `signal` and non-zero `exit_code` per Unix convention, not `call_errored`.

#### `call_errored`

The producer itself failed to complete observation or forwarding. Operationally distinct from a non-zero exit on `call_completed`.

```json
{
  "event_type": "call_errored",
  "...": "...",
  "call": { },
  "error": {
    "category": "exec_failed",
    "message": "Cannot find executable: curl",
    "underlying_errno": 2,
    "stage": "forwarding"
  }
}
```

|Field                   |Required|Type         |Notes                                                                            |
|------------------------|--------|-------------|---------------------------------------------------------------------------------|
|`error.category`        |REQUIRED|string (enum)|`"exec_failed"`, `"observation_lost"`, `"timeout"`, `"internal_error"`, `"other"`|
|`error.message`         |REQUIRED|string       |Human-readable, display-safe                                                     |
|`error.underlying_errno`|OPTIONAL|integer      |System errno when meaningful                                                     |
|`error.stage`           |OPTIONAL|string       |Producer-defined pipeline stage                                                  |

A `call_errored` event MUST NOT precede `call_completed` for the same `call_id`.

#### `policy_load_succeeded`

```json
{
  "event_type": "policy_load_succeeded",
  "...": "...",
  "policy": {
    "policy_id": "khorum-oss/network-allowlist",
    "policy_version": "2026.04.20-1",
    "signature_fingerprint": "sha256:a1b2c3...",
    "load_source": "https://reliquary.example.com/policies/network-allowlist-2026.04.20-1.bundle"
  },
  "config_fingerprint": "sha256:..."
}
```

`policy.load_source` records where the producer fetched the policy from — file path, URL, OCI reference. `config_fingerprint` REQUIRED.

#### `policy_load_failed`

```json
{
  "event_type": "policy_load_failed",
  "...": "...",
  "policy": {
    "policy_id": "khorum-oss/network-allowlist",
    "policy_version": "2026.04.20-1",
    "load_source": "https://reliquary.example.com/policies/network-allowlist-2026.04.20-1.bundle"
  },
  "error": {
    "category": "signature_invalid",
    "message": "Signature verification failed: untrusted signing key 0xABCD..."
  },
  "active_policy_status": "previous_retained",
  "config_fingerprint": "sha256:..."
}
```

|Field                 |Required|Type         |Notes                                                                                                          |
|----------------------|--------|-------------|---------------------------------------------------------------------------------------------------------------|
|`error.category`      |REQUIRED|string (enum)|`"signature_invalid"`, `"signature_revoked"`, `"schema_invalid"`, `"parse_failed"`, `"fetch_failed"`, `"other"`|
|`active_policy_status`|REQUIRED|string (enum)|`"none"`, `"previous_retained"`, `"fail_open"`, `"fail_closed"`                                                |

`signature_fingerprint` gets omitted on `policy_load_failed` because the signature didn’t verify.

#### `producer_started` and `producer_stopped`

`producer_started` already shown in Part 2. `config_fingerprint` REQUIRED.

```json
{
  "event_type": "producer_stopped",
  "...": "...",
  "stop_reason": "shutdown_requested",
  "graceful": true
}
```

|Field        |Required|Type   |Notes                                                            |
|-------------|--------|-------|-----------------------------------------------------------------|
|`stop_reason`|REQUIRED|string |Free-form short identifier; producers MAY define their own values|
|`graceful`   |REQUIRED|boolean|Whether producer flushed state before exit                       |

A crashed producer can’t emit `producer_stopped`; the missing event itself signals the crash.

### Part 5: Event Format Conformance

The conformance suite lives in the `eidolon-system` repository under `conformance/events/`. Versions alongside the spec.

#### Producer conformance suite (permissive)

The producer suite operates by inspection of emitted events. Three layers:

**Layer 1: Schema validation.** Every event the producer emits validates against the published JSON Schema for its declared `event_type`.

**Layer 2: Cross-event consistency.**

- Every `event_id` stays unique across the stream
- Events sharing a `call_id` carry the same `host.id`, `producer.instance_id`, and consistent `call.skill_id`
- Lifecycle ordering: `call_attempted` precedes `call_allowed`/`call_denied`, which precede `call_completed`/`call_errored`
- A `call_id` never appears with both `call_completed` and `call_errored`
- A `call_id` never appears with both `call_allowed` and `call_denied`
- `producer_started` precedes any other events from the same `producer.instance_id`
- Timestamps within a single `call_id` stay non-decreasing

**Layer 3: Producer descriptor accuracy.** If the producer publishes a descriptor, every event type and required field it actually emits must appear in the descriptor (the descriptor functions as an upper bound).

#### Consumer conformance suite (strict)

The consumer suite drives the consumer with fixtures and verifies outputs. Three categories:

**Category A: Standard events.** Valid events covering every event type and field combination. Consumer MUST accept all.

**Category B: Forward-compatibility events.** Valid envelope and event_type with unknown optional fields, unknown `x_*` extensions, and unknown enum values where extensible. Consumer MUST accept without error.

**Category C: Adversarial events.** Spec-violating events. Consumer behavior gets partly prescribed:

- Unknown event types: consumer MUST log them
- Schema-invalid events: consumer behavior stays consumer-defined; documented in the consumer’s conformance report
- Oversized events: consumer behavior stays consumer-defined; documented in conformance report

#### Round-trip conformance (optional, with strict mode)

Implementations doing both producer and consumer roles MAY claim round-trip conformance. Strict round-trip mode requires producer and consumer conformance plus end-to-end fidelity for shared fields. Useful for tightly-coupled stacks like Eidoshim + Eidolens.

#### Suite distribution

Canonical form: a repo of language-agnostic fixtures (JSON files plus expected-output declarations). The JVM harness publishes to Reliquary as a convenience runner; other languages can replicate against the same fixtures.

#### Reporting

```json
{
  "spec_version": "1.0.0",
  "implementation_name": "eidoshim",
  "implementation_version": "0.3.1",
  "implementation_role": "producer",
  "run_timestamp": "2026-04-26T14:23:01Z",
  "result": "passed",
  "layers": {
    "schema_validation": { "result": "passed", "events_checked": 1248 },
    "cross_event_consistency": { "result": "passed", "checks_performed": 9 },
    "descriptor_accuracy": { "result": "passed" }
  },
  "consumer_handling": null,
  "notes": []
}
```

No central registry in v1. Implementations publish reports alongside their releases; future infrastructure (registry repo, dockerized validator) stays open.

#### What the suite deliberately doesn’t test

- **Performance** — no normative requirements
- **Transport correctness** — payload-level only
- **Producer determinism** — different runs of the same scenario will differ in IDs and timestamps; that’s fine
- **Operational quality** — reliability, responsiveness, crash behavior count as implementation concerns

-----

## Eidolon Policy Format Specification — v1.0.0

### Part 1: Scope and Structure

#### What the spec defines

The Eidolon Policy Format describes a wire format for policy artifacts that govern call-observation tools. A conforming **author** produces policies matching the format. A conforming **enforcer** consumes policies and applies them to observed calls.

The spec covers:

- The set of policy scopes (network, filesystem, exec, env) and what each governs
- The structure of rules within each scope, including matchers and decisions
- Conflict resolution and evaluation order
- Bundle structure: how rules group, how versions identify, how metadata attaches
- Canonical form rules for byte-stable serialization
- Signature envelope structure
- Versioning rules and compatibility guarantees
- Conformance criteria for authors and enforcers

#### What the spec deliberately does not define

- **The authoring format.** HOCON, YAML, Kotlin DSL, raw JSON, hand-written canonical JSON — all valid as long as they produce conforming canonical JSON.
- **The signing implementation.** PGP, cosign, future schemes — the spec defines the envelope structure, not which scheme to use.
- **The distribution mechanism.** Reliquary, OCI registry, file copy, HTTP fetch — out of scope.
- **The trust root mechanism.** How an enforcer establishes trust in signing keys stays implementation-specific.
- **The decision engine implementation.** The spec defines what decisions a rule expresses; an enforcer implements evaluation however it likes.
- **Producer-side authorization.** Whether a particular human or CI job may sign a particular policy counts as governance, not format.

#### Conformance levels

Both author and enforcer conformance run **strict**. The major difference from the Event Format spec, where producer conformance ran permissive.

Strict on both sides because:

- Authors MUST produce policies that all enforcers interpret identically. A policy meaning “deny” on enforcer A and “allow” on enforcer B counts as a security incident.
- Enforcers MUST implement every documented matcher. An enforcer silently ignoring a matcher it doesn’t understand could permit calls a policy intended to deny. Failing closed (refusing to load policies with unknown matchers) stays the only safe default.

Determinism counts as the core property: same policy, same call, same decision, regardless of which enforcer evaluates it.

#### Versioning

Same model as Event Format. The `required: false` flag on new matchers handles the strict-on-both-sides versioning problem: matchers added in minor bumps carry the flag, allowing older enforcers to skip rules using them rather than fail closed. Authors who care about wide compatibility avoid flagged matchers; authors who need them accept the implicit “needs enforcer >= X” requirement.

- **Patch bump** — clarifications, no wire changes.
- **Minor bump** — new matchers, new scopes (with `required: false` flag for new matchers/scopes).
- **Major bump** — anything changing how an existing policy gets interpreted.

#### The three levels of structure

A policy artifact has three nested levels:

**Bundle** — the signed unit. Contains one or more policies, metadata, and signature(s). The bundle gets distributed and what an enforcer loads.

**Policy** — a logical grouping of rules under a single identifier. A bundle MAY contain multiple policies; the common case stays one. Each policy carries its own `policy_id`, version, and scope-grouped rules. Multi-policy support remains supported in the format but deferred from v1 conformance testing.

**Rule** — a single matcher-plus-decision unit. The smallest evaluable thing.

#### Policy identity and naming

Every policy carries a `policy_id` of the form `<namespace>/<name>`. Recommended convention follows existing khorum-oss patterns: `khorum-oss/network-allowlist`, `internal/team-deploy-policy`. The `<namespace>/<name>` form counts as convention, not enforced syntax — enforcers MUST treat `policy_id` as opaque except where the spec says otherwise.

Each policy also carries a `policy_version`. The spec doesn’t mandate a specific versioning scheme — date-based (`2026.04.20-1`), semver (`1.2.0`), content-hash, build-number — all valid. Enforcers MUST treat `policy_version` as opaque.

The pair `(policy_id, policy_version)` MUST uniquely identify a specific signed artifact.

#### Bundle metadata

```json
{
  "schema_version": "1.0.0",
  "bundle_id": "01HXY...",
  "bundle_version": "2026.04.26-1",
  "created_at": "2026-04-26T14:00:00Z",
  "created_by": "ci-job-12345",
  "policies": [
    { "policy_id": "khorum-oss/network-allowlist", "policy_version": "2026.04.20-1", "...": "..." }
  ]
}
```

`created_by` REQUIRED, free-form string (human name, CI job ID, `"automated"` — author’s choice).

#### Extensions

Authors MAY use custom matchers via the `x_*` namespace. The bundle’s `extensions` array declares each extension with a `required` flag:

```json
{
  "extensions": [
    {
      "extension_id": "x_violabs_geofence",
      "vendor": "violabs",
      "required": true,
      "spec_url": "https://violabs.io/eidolon-extensions/geofence-v1"
    }
  ]
}
```

If `required: true`, an enforcer not implementing the matcher MUST refuse to load the bundle. If `required: false`, an enforcer MAY skip rules using the matcher but MUST log that it skipped them.

Naming convention: `x_<vendor>_<matcher>`.

### Part 2: Policy Schema

#### Policy structure

```json
{
  "policy_id": "khorum-oss/network-allowlist",
  "policy_version": "2026.04.20-1",
  "policy_description": "Default network allowlist for Anthropic skill execution",
  "default_decision": "deny",
  "scopes": {
    "network": [],
    "filesystem": [],
    "exec": [],
    "env": []
  },
  "redactions": []
}
```

|Field               |Required|Type         |Notes                                                              |
|--------------------|--------|-------------|-------------------------------------------------------------------|
|`policy_id`         |REQUIRED|string       |Per Part 1                                                         |
|`policy_version`    |REQUIRED|string       |Per Part 1                                                         |
|`policy_description`|REQUIRED|string       |Human-readable; display-safe                                       |
|`default_decision`  |REQUIRED|string (enum)|`"allow"`, `"deny"`, `"log_only"`                                  |
|`scopes`            |REQUIRED|object       |Maps scope name to rule arrays                                     |
|`redactions`        |OPTIONAL|array        |Top-level (not under scopes); applied to argv before event emission|

#### The four scopes

- **`network`** — outbound network calls. Matches on URL components, hostnames, ports, methods, headers.
- **`filesystem`** — filesystem access. Matches on paths and access modes.
- **`exec`** — subprocess invocation. Matches on command name, argv patterns, parent process. Meta-scope: controls which commands may run at all.
- **`env`** — environment variable access. Matches on variable names; values stay invisible to policy.

The four scopes overlap deliberately. A `curl https://evil.com` invocation hits both `exec` and `network`.

#### Rule structure

```json
{
  "rule_id": "anthropic-docs",
  "rule_description": "Allow API calls to Anthropic documentation",
  "match": {},
  "decision": "allow",
  "decision_metadata": {
    "reason_template": "Host matches allowlist entry {{ matched_host }}"
  }
}
```

|Field                              |Required|Type         |Notes                                  |
|-----------------------------------|--------|-------------|---------------------------------------|
|`rule_id`                          |REQUIRED|string       |Unique within policy; appears in events|
|`rule_description`                 |REQUIRED|string       |Human-readable, display-safe           |
|`match`                            |REQUIRED|object       |Scope-specific matchers; all AND’d     |
|`decision`                         |REQUIRED|string (enum)|`"allow"`, `"deny"`, `"log_only"`      |
|`decision_metadata.reason_template`|OPTIONAL|string       |Template for events `decision.reason`  |

All matchers within a single rule AND together. For OR-logic, write multiple rules. For complex compound conditions, the spec adds `any_of`, `all_of`, `not` operators at a future minor bump if real authoring patterns demonstrate the need; v1 sticks with implicit AND.

#### Network scope matchers

|Matcher         |Type   |Operators                          |
|----------------|-------|-----------------------------------|
|`host`          |string |`equals`, `glob`, `suffix`, `in`   |
|`port`          |integer|`equals`, `in`, `range`            |
|`method`        |string |`equals`, `in`                     |
|`scheme`        |string |`equals`, `in`                     |
|`path`          |string |`equals`, `glob`, `prefix`, `regex`|
|`header_present`|string |`equals`, `in`                     |

Operator semantics:

- `equals`: exact match
- `in`: value appears in the provided array
- `glob`: shell-style glob (`*` matches any sequence except `/`, `**` matches across `/`, `?` matches single char)
- `suffix`: string ends with value
- `prefix`: string starts with value
- `range`: integer falls within `[min, max]` inclusive
- `regex`: ECMA-262 regex; required for enforcer support; enforcers MUST timeout regex evaluation (10ms hard cap) and treat timeout as non-match

Authors MUST NOT mix multiple operators on a single matcher in a single rule.

#### Filesystem scope matchers

|Matcher         |Type   |Operators                                                                |
|----------------|-------|-------------------------------------------------------------------------|
|`path`          |string |`equals`, `glob`, `prefix`, `regex`                                      |
|`access`        |string |`equals`, `in` (values: `read`, `write`, `execute`, `delete`, `metadata`)|
|`path_under_cwd`|boolean|`equals` (true means path lies within or equals the call’s `cwd`)        |

#### Exec scope matchers

|Matcher         |Type   |Operators                                                      |
|----------------|-------|---------------------------------------------------------------|
|`command`       |string |`equals`, `in`, `glob`                                         |
|`argv_contains` |string |`equals`, `glob`, `regex` (matches if any argv element matches)|
|`argv_at`       |object |Positional matcher; see below                                  |
|`argv_count`    |integer|`equals`, `range`                                              |
|`parent_command`|string |`equals`, `in`, `glob`                                         |

`argv_at` for position-specific matching:

```json
"argv_at": {
  "position": 1,
  "operator": "equals",
  "value": "--unsafe-mode"
}
```

#### Env scope matchers

|Matcher          |Type   |Operators                                                      |
|-----------------|-------|---------------------------------------------------------------|
|`env_key_present`|string |`equals`, `in`, `glob` (matches if any present env key matches)|
|`env_key_count`  |integer|`equals`, `range`                                              |

Env scope matchers operate on key names only, never values.

#### Redactions

Redactions live at the top level of the policy, parallel to `scopes`:

```json
{
  "redactions": [
    {
      "redaction_id": "curl-bearer-tokens",
      "command": { "equals": "curl" },
      "argv_pattern": { "regex": "Bearer [A-Za-z0-9_-]+" },
      "replacement": "Bearer <redacted>"
    }
  ]
}
```

A redaction matches command and an argv pattern; matched substrings get replaced before the event gets emitted. Multiple redactions can apply to the same call; they apply in declaration order.

### Part 3: Conflict Resolution and Evaluation Order

#### Layer 1: Scope evaluation order

Scopes evaluate in a fixed order, defined by the spec:

1. **`exec`** — first
1. **`env`** — second
1. **`filesystem`** — third
1. **`network`** — fourth

The first scope producing a deny decision short-circuits subsequent scopes. Allow decisions don’t short-circuit — every scope evaluates fully unless something denies.

Order stays fixed for determinism. Authors cannot reorder it.

**Debug evaluation mode.** Enforcers MUST support a debug mode evaluating all scopes regardless of short-circuit, recording secondary matches as `x_secondary_matches` on emitted events. Mode toggles via implementation choice (config flag, env var, command-line option) — spec doesn’t dictate how.

#### Layer 2: Within-scope rule evaluation

Within a single scope, rules evaluate in **document order** — the order they appear in canonical JSON. Document order stays stable because canonical JSON stays byte-stable.

Within a scope, the evaluation rule: **first match wins**. The first rule whose `match` block evaluates true contributes its decision; subsequent rules don’t evaluate.

Authoring tools (Eidocon, the Kotlin DSL) SHOULD render rules in their declared order without reordering, and SHOULD warn when later rules become unreachable due to earlier broader matches.

#### Layer 3: Cross-scope decision combination

After each scope evaluates, the enforcer has up to four scope-level decisions plus the policy’s `default_decision`. Precedence (highest to lowest):

```
deny > allow > log_only > default_decision
```

Network’s deny doesn’t outrank exec’s deny; either is just “deny.” No scope hierarchy beyond the evaluation order in Layer 1.

#### Determinism guarantees

Three guaranteed properties for any conforming enforcer:

1. **Determinism property.** Given the same canonical policy bytes and the same input call, every conforming enforcer produces the same decision and the same `match_rule` value.
1. **Stability property.** A canonical policy byte-identical to a previously-evaluated one produces identical decisions for identical calls across enforcer restarts and instances.
1. **Locality property.** A rule’s evaluation depends only on its own `match` block, the call being evaluated, and its position in document order.

#### What conflict resolution deliberately omits

- **No priority field on rules.** Document order serves as priority.
- **No “stricter wins” semantic across scopes.** Decisions don’t reference each other.
- **No rule chaining or referencing.**
- **No conditional evaluation based on prior decisions.**

### Part 4: Canonical Form

#### Why canonical form matters

The signature on a policy bundle hashes bytes. Without canonical form, two enforcers might parse the same logical policy and produce slightly different serializations whose signatures don’t verify. Canonical form defines exactly one byte sequence per logical policy.

#### RFC 8785 JSON Canonicalization Scheme

Eidolon adopts RFC 8785 (JCS) as the canonical form for signed bundles. RFC 8785 specifies:

- UTF-8 encoding without BOM
- Object keys sorted lexically by Unicode code point
- No insignificant whitespace
- Numbers serialized per IEEE 754 double-precision rules with specific formatting
- Strings escaped with the minimal JSON escape set
- No comments

#### Eidolon-specific canonicalization rules

**Rule 1: Number type pinning.** RFC 8785 doesn’t distinguish “the number 1” from “the number 1.0” — both serialize to `1`. The Eidolon schema declares the type of every numeric field. Authors writing `1.0` for an integer field, or `1` for a float field, get a schema validation failure at compile time.

When an integer-valued float and an integer field both serialize to `1` per RFC 8785, the canonical bytes match — which counts as correct, because schema validation already disambiguated the types upstream. Worth being explicit: identical canonical bytes for `1.0` (float field) and `1` (integer field) reflect that schema typing happens before canonicalization, not a canonicalization bug.

**Rule 2: Boolean and null normalization.** RFC 8785 already handles this. Authors writing `True` or `TRUE` in HOCON or YAML get normalized to `true` at the AST level before canonicalization. Same for `null`.

**Rule 3: Array order is meaningful and preserved.** Per Part 3, document order in arrays carries semantic significance (rule evaluation order). Reordering an array of rules changes the canonical bytes, which changes the signature.

#### Author-format-specific normalization

**HOCON:**

- Substitutions (`${var}`) MUST resolve before canonicalization.
- Includes (`include "other.conf"`) MUST resolve at compile time.
- Duplicate keys at the same level count as an authoring error. The compiler MUST reject input with duplicate keys. (This diverges from standard HOCON behavior, which treats later keys as overrides — Eidolon overrides this for safety.)
- Number type stays ambiguous in HOCON; the schema’s declared type wins per Rule 1.
- Comments get stripped during AST construction.

**YAML:**

- Anchors and aliases MUST expand before canonicalization.
- Merge keys (`<<:`) MUST resolve.
- Implicit type coercion (the “Norway problem”, `1.0` vs `1`) doesn’t apply at the policy level. Schema-declared type counts as the source of truth.
- Comments get stripped.
- Multi-document YAML files (`---` separators) don’t count as valid policy input. One file = one bundle.

**Kotlin DSL:**

- Produces canonical JSON directly via Konstellation-generated builders.
- Default values declared in the DSL MUST apply before serialization.
- `null` values for OPTIONAL fields MUST get omitted from canonical form.

**Raw JSON:**

- Skips the AST step but still goes through schema validation and RFC 8785 canonicalization.

#### Round-trip equivalence guarantee

For any logical policy expressible in multiple authoring formats, all valid encodings produce **byte-identical canonical JSON**.

#### Bundle structure for signing

The signed unit counts as the bundle, not individual policies. A bundle’s canonical form holds bundle-level metadata, the `extensions` array, and the `policies` array — all canonicalized as one JSON document, signed as one unit.

#### What sits outside the canonical form

- **Human-readable source.** Signed bundles ship alongside the original HOCON/YAML/Kotlin source for reviewer benefit. Source counts as informational only — MUST NOT influence enforcer behavior.
- **Comments and rationale.** Stripped during canonicalization. Authors record rationale via `policy_description` and `rule_description`.
- **Build provenance.** Information about how the bundle got built (CI job, commit hash, builder identity) lives in the distribution envelope outside the signature, because it changes per build.
- **Signature(s).** The signature wraps the canonical bytes; it can’t sit inside them.

#### Distribution packaging

v1 recommends OPTIONAL tarball/archive packaging (canonical JSON + signature envelope + optional source files). The normative artifact stays the canonical JSON bytes.

Future minor bumps may add OCI artifact distribution as a parallel packaging shape. The artifact-vs-packaging separation in this spec keeps that path open.

### Part 5: Signature Envelope

#### What the signature envelope is

The canonical JSON bundle from Part 4 counts as the signed unit. The signature envelope wraps that canonical bundle with the signature(s), the scope(s) being claimed, and metadata about who signed and when.

The envelope itself counts as JSON but deliberately doesn’t canonicalize the same way the bundle does — the envelope can change without the bundle changing (countersignatures, revocations recorded), and forcing canonical form on the envelope would break that.

```json
{
  "envelope_version": "1.0.0",
  "bundle": {
    "canonical_bytes_b64": "eyJzY2hlbWFfdmVyc2lvbiI6IjEuMC4wIiwi...",
    "canonical_hash": "sha256:a1b2c3..."
  },
  "signatures": [
    {
      "signature_id": "01HXY...",
      "key_fingerprint": "4AEE18F83AFDEB23",
      "key_scope": ["network/*", "filesystem/read/*"],
      "algorithm": "pgp-ed25519",
      "signed_at": "2026-04-26T14:00:00Z",
      "signer_identity": "ci-job-12345",
      "signature_b64": "iQEzBAABCgAdFiEE..."
    }
  ]
}
```

The envelope itself stays unsigned. An attacker could add fake signature objects, but verification rejects them because their signature math fails against the bundle. Worth being explicit so implementers don’t try to “fix” what isn’t broken.

#### Envelope fields

|Field                       |Required|Type           |Notes                                  |
|----------------------------|--------|---------------|---------------------------------------|
|`envelope_version`          |REQUIRED|string (semver)|Distinct from bundle’s `schema_version`|
|`bundle.canonical_bytes_b64`|REQUIRED|string         |Base64-encoded canonical JSON          |
|`bundle.canonical_hash`     |REQUIRED|string         |`<algorithm>:<hex>` format             |
|`signatures`                |REQUIRED|array          |One or more signature objects          |

#### Signature object

|Field            |Required|Type             |Notes                                      |
|-----------------|--------|-----------------|-------------------------------------------|
|`signature_id`   |REQUIRED|string (ULID)    |Unique per signature; useful for revocation|
|`key_fingerprint`|REQUIRED|string           |PGP fingerprint or equivalent              |
|`key_scope`      |REQUIRED|array            |Scopes this signing key authorizes for     |
|`algorithm`      |REQUIRED|string           |`pgp-ed25519`, `pgp-rsa` in v1             |
|`signed_at`      |REQUIRED|string (RFC 3339)|When the signature got produced            |
|`signer_identity`|REQUIRED|string           |Free-form identifier                       |
|`signature_b64`  |REQUIRED|string           |Algorithm-dependent format, base64-encoded |

#### Scoped keys

A signing key carries scope constraints limiting what it authorizes. Verification checks two things:

1. The claimed scopes match what the bundle contains. Every rule must lie under at least one signature whose `key_scope` covers it.
1. The trust root permits the key to claim those scopes.

Scope paths follow the hierarchy:

- `network/*` — all network rules
- `filesystem/*` — all filesystem rules
- `filesystem/read/*` — only filesystem rules with `access` matchers limited to read modes
- `exec/*` — all exec rules
- `env/*` — all env rules and redactions
- `*` — all scopes (the universal key, typically the trust root itself)

The hierarchy stays intentionally shallow in v1.

#### N-of-M signing

The bundle declares its signing requirements in a `signing_policy` field at the bundle level (inside the canonical bundle, not the envelope, so the requirements themselves stay signed):

```json
{
  "schema_version": "1.0.0",
  "signing_policy": {
    "scope_requirements": [
      { "scope_pattern": "exec/*", "min_signatures": 2 },
      { "scope_pattern": "network/deny/*", "min_signatures": 2 },
      { "scope_pattern": "*", "min_signatures": 1 }
    ]
  }
}
```

Two signatures from the same key fingerprint count as one — prevents a single signer from satisfying N-of-M by signing twice.

`signing_policy` is REQUIRED on every bundle. Bundles without an explicit declaration fail to load. This forces authors to acknowledge their signing requirements rather than rely on a permissive default.

#### Revocation list

Revocation runs as a separately-signed companion artifact, distributed alongside policy bundles:

```json
{
  "envelope_version": "1.0.0",
  "revocation_list": {
    "list_id": "khorum-oss/primary-revocation-list",
    "list_version": "2026.04.26-1",
    "issued_at": "2026-04-26T14:00:00Z",
    "revoked_keys": [
      {
        "key_fingerprint": "4AEE18F83AFDEB23",
        "revoked_at": "2026-04-25T10:00:00Z",
        "reason": "Key compromise reported",
        "applies_to_signatures_after": "2026-04-25T08:00:00Z"
      }
    ],
    "revoked_signatures": [
      {
        "signature_id": "01HXY...",
        "revoked_at": "2026-04-26T11:00:00Z",
        "reason": "Authoring error in revoked bundle"
      }
    ]
  },
  "list_signature": {
    "key_fingerprint": "TRUST_ROOT_KEY_FP",
    "algorithm": "pgp-ed25519",
    "signed_at": "2026-04-26T14:00:00Z",
    "signature_b64": "..."
  }
}
```

Two revocation modes:

- **Key revocation.** A signing key gets distrusted from a given timestamp forward. Signatures the key produced before `applies_to_signatures_after` remain valid; signatures after don’t. Omitting `applies_to_signatures_after` means revoke all signatures from this key (the simple-aggressive default).
- **Signature revocation.** A specific signature gets revoked, regardless of timing.

The revocation list itself MUST be signed by the trust root. Enforcers MUST verify the trust-root signature on the list before honoring its contents.

#### Verification order

Enforcer verification pipeline runs in fixed order. Failure at any step causes rejection:

1. Parse the envelope. Reject on syntax errors.
1. Decode `bundle.canonical_bytes_b64`.
1. Compute hash of decoded canonical bytes; verify it matches `bundle.canonical_hash`.
1. Parse the canonical bundle. Reject on syntax errors.
1. Validate the bundle against the schema. Reject on schema-invalid bundles.
1. For each signature in the envelope:
- Verify the cryptographic signature against the canonical bytes
- Verify the key fingerprint resolves to a known key in the trust root
- Verify the key isn’t in the revocation list (with timing rules per above)
1. Verify the bundle’s `signing_policy` gets satisfied by the valid signatures.
1. Verify every rule’s scope falls under at least one valid signature’s `key_scope`.

#### Revocation list caching

The spec recommends a maximum cache age of 1 hour for revocation lists but doesn’t mandate. Enforcers document their caching behavior in conformance reports.

#### Bootstrap and trust-on-first-use

The trust root must land on the enforcer somehow. The spec doesn’t define how but does require:

- Enforcers MUST document their trust root bootstrap mechanism.
- Enforcers MUST treat trust root rotation as a deliberate, audited operation. A runtime config change MUST NOT modify the trust root.
- Enforcers SHOULD support pinning the trust root by checksum at install time.

Concretely for Eidoshim: the trust root’s public keys get baked into the binary at build time, with a fingerprint pinned in the install script. Rotation requires reinstalling.

### Part 6: Policy Format Conformance

#### Author conformance (strict)

Three layers:

**Layer 1: Round-trip equivalence.** Given a fixture expressing the same logical policy in HOCON, YAML, Kotlin DSL, and raw JSON, the author MUST produce byte-identical canonical JSON for all four. Any byte-level divergence fails conformance.

**Layer 2: Schema validation pre-canonicalization.** The author MUST reject schema-invalid input before emitting canonical form.

**Layer 3: Format-specific normalization.** Each authoring format has rules from Part 4 (HOCON substitutions resolved, YAML anchors expanded, duplicate-key rejection, comment stripping).

#### Enforcer conformance (strict)

Five layers:

**Layer 1: Envelope and signature verification.** Given fixture envelopes (some valid, some with various signature failures), the enforcer MUST follow the eight-step verification pipeline from Part 5 in order.

**Layer 2: Decision determinism.** Given a verified bundle and a set of fixture calls, the enforcer MUST produce decisions matching the suite’s expected outputs:

- ~50 base fixtures covering each scope with simple rules
- ~30 cross-scope fixtures exercising conflict resolution
- ~20 edge-case fixtures (empty scopes, catch-all rules, default_decision fallback, document-order tie-breaking)
- ~10 multi-policy fixtures (deferred from v1 conformance; suite includes them as forward-compatibility checks)

**Layer 3: Debug evaluation mode.** Enforcers MUST support a debug mode that evaluates all scopes regardless of short-circuit. The suite verifies by running the same fixtures in both modes.

**Layer 4: Schema rejection.** Enforcers MUST reject bundles whose canonical JSON validates as JSON but fails the policy schema.

**Layer 5: Extension handling.** Required extensions MUST cause unknown-extension enforcers to reject. Optional extensions MUST cause skip-with-log behavior.

#### Cross-implementation conformance

Beyond per-implementation conformance, the suite includes a cross-implementation harness verifying that two enforcers produce identical decisions. Run nightly in CI for the reference implementations.

#### Suite distribution

Same model as the Event Format conformance suite. Canonical form: a repo of language-agnostic fixtures. JVM reference runner publishes to Reliquary.

#### What the suite deliberately omits

- **Performance** — no normative timing requirements.
- **Operational quality** — reliability under load, crash recovery, distribution mechanisms.
- **Trust root bootstrap** — out of scope per Part 1.
- **Signing operations** — the suite verifies enforcers correctly verify signatures, not that authors correctly produce them.
- **Key management** — out of scope.

-----

## Decisions Made

The following decisions stay locked across the project:

**Format-level:**

- HOCON and YAML count as first-class authoring formats. Neither converts through the other.
- Canonical signed form: RFC 8785 canonical JSON.
- Wire format casing: snake_case (matches OTel/CloudEvents/OCSF). Application APIs translate to camelCase at their boundaries.
- Event identifiers use ULID (sortable lex order by time).
- Timestamps use RFC 3339 with nanosecond precision recommended.
- Soft 64KB event size cap with truncation conventions.
- `x_*` extension namespace for both formats.

**Architectural:**

- Two-level bundle structure (bundle → policies → rules) supported in v1; conformance suite tests single-policy only.
- Matchers extensible via declared `extensions` block with per-extension `required` flag.
- Scope evaluation order fixed by spec: exec → env → filesystem → network.
- Short-circuit on first deny by default. Enforcers MUST support debug-evaluation mode.
- `prompt` decision deferred from v1 (will arrive at minor bump with a designed prompt-handling protocol).
- Decision precedence: deny > allow > log_only > default_decision.
- N-of-M signing required to be explicit (`signing_policy` mandatory in every bundle).
- Distribution: tarball recommended OPTIONAL packaging in v1; OCI as documented future packaging path.

**Implementation-level:**

- Signing scheme: PGP via 1Password at launch. Cosign as alternative comes later if needed.
- Distribution backend: DigitalOcean Spaces (Reliquary) and OCI registry from day one.
- Hot-reload of policy at runtime supported (Spektr pattern proven).
- Native enforcer (Eidoshim) and JVM evaluator (in eidocon-dsl test harness) count as separate implementations validated by a shared conformance suite.
- Producer conformance: permissive. Consumer conformance: strict. Author conformance: strict. Enforcer conformance: strict.
- No central conformance registry in v1. Future options (registry repo, dockerized validator) stay open.
- Skill identity in events asserted, not verified by the format itself; absence and `"unknown"` mean the same thing.

## Open Questions

Genuinely unresolved, needs investigation during implementation:

- Naming for the per-skill policy bundle (lean “manifest”)
- Webhook vs pull-only model for policy distribution from Eidocon to Eidoshim. Likely needs both.
- Bootstrap of self-hosted Eidolens endpoints — needs a small signed config alongside the trust root.

-----

# Section 2: Per-Repo Charters

## Repository inventory

Seven repositories under the `khorum-oss` GitHub organization:

|Repository               |Purpose                                                |Language                    |Distribution                            |
|-------------------------|-------------------------------------------------------|----------------------------|----------------------------------------|
|`eidolon-system`         |Specs, conformance fixtures, master architecture docs  |Markdown + JSON             |Git tags                                |
|`eidolon-conformance-jvm`|JVM reference runner for both conformance suites       |Kotlin                      |Maven via Reliquary                     |
|`eidoshim`               |Reference enforcer; multi-call shim binary             |Kotlin (GraalVM native)     |Native binaries via Reliquary           |
|`eidocon`                |Policy CLI: compiler, signer, distributor              |Kotlin (GraalVM native)     |Native binaries via Reliquary           |
|`eidocon-dsl`            |Konstellation-generated Kotlin DSL for policy authoring|Kotlin                      |Maven via Reliquary                     |
|`eidolens`               |Reference event consumer; observability backend        |Kotlin (Spring Boot WebFlux)|Docker image / DigitalOcean App Platform|
|`eidolens-svelte`        |Frontend for Eidolens                                  |TypeScript (SvelteKit)      |Bundled with Eidolens or static deploy  |

## Charter: `eidolon-system` (Umbrella Spec Repository)

### Purpose and scope

Holds the canonical Eidolon Event Format and Eidolon Policy Format specifications, the JSON Schema files for both, and the conformance suite fixtures. This repository serves as the source of truth for what the format means; every implementation references it.

The repository contains no executable code. Markdown, JSON Schema, and JSON fixtures only.

**Deliberately out of scope:** No language-specific tooling. No reference implementations of producers, consumers, authors, or enforcers (those have their own repos). No documentation of how to deploy any implementation.

### Position in architecture

Depends on no other Eidolon repositories. Every other repository depends on this one — directly for specs and fixtures, indirectly because their behavior must conform to its definitions. Changes here ripple outward, so versioning and release discipline matter.

### Public surface

Three deliverables:

- **Spec documents** under `docs/specs/` — `event-format-1.0.0.md`, `policy-format-1.0.0.md`. Versioned filenames; old versions stay accessible.
- **JSON Schema files** under `schemas/` — `event-1.0.0.schema.json`, `policy-1.0.0.schema.json`, `envelope-1.0.0.schema.json`, `revocation-list-1.0.0.schema.json`.
- **Conformance fixtures** under `conformance/events/` and `conformance/policy/` — language-agnostic, organized per the structures defined in spec Part 5/6.

### Dependencies

None. JSON and Markdown only.

### Quality bar

- All Markdown lints clean (markdownlint defaults).
- All JSON Schema files validate as valid JSON Schema 2020-12.
- All fixture JSON files parse as valid JSON.
- Every fixture covers a documented spec requirement; coverage report available on demand.
- Spec changes go through PR review with at least one approver who didn’t author the change.

### Release and distribution

Versioned via git tags matching spec versions: `events-v1.0.0`, `policy-v1.0.0`. The repo can carry both specs at different versions independently; tag prefixes distinguish them.

Specs render to HTML via a static-site generator (likely just `mkdocs`) and publish to `eidolon.khorum-oss.org`. CI handles the publishing on tag push.

### Conformance obligations

This repository is the source of conformance, not a target. It carries no conformance obligations of its own.

### Notes for Claude Code session

When working in this repo, sessions write spec text, JSON Schema, and JSON fixtures only. Reject any suggestion to add code, build tooling beyond the docs site generator, or implementation logic. The cleanliness of this repo’s “no code” rule keeps the spec genuinely independent of any single language ecosystem.

-----

## Charter: `eidolon-conformance-jvm`

### Purpose and scope

JVM reference runner for both the Event Format and Policy Format conformance suites. Reads fixtures from the `eidolon-system` repository, executes them against a target implementation (producer/consumer/author/enforcer), and produces conformance reports.

Serves as a convenience for JVM-based implementations. Other languages write their own runners against the same fixtures.

**Deliberately out of scope:** No fixtures of its own (those live in `eidolon-system`). No assumptions about which target implementation it tests (target gets specified at runtime via SPI or CLI flag).

### Position in architecture

Depends on `eidolon-system` for fixtures and JSON Schema files. Used by `eidoshim`, `eidocon`, `eidolens`, and `eidocon-dsl` for their own conformance testing.

### Public surface

- `EventConformanceRunner` class: ingests an event stream from a target producer or sends fixtures to a target consumer; produces a `ConformanceReport`.
- `PolicyConformanceRunner` class: drives a target author or enforcer through fixtures; produces a `ConformanceReport`.
- `ConformanceReport` data class with the report shape defined in spec Part 5 (events) and Part 6 (policy).
- CLI wrapper (`eidolon-conformance` command) for ad-hoc runs outside Gradle/Maven test integration.

The Maven coordinate publishes to Reliquary as `io.violabs.eidolon:conformance-jvm`.

### Dependencies

- Kotlin 2.x
- Jackson for JSON parsing
- A JSON Schema validator (lean networknt/json-schema-validator)
- Clikt for the CLI wrapper
- JUnit 5 / Spock for testing the runner itself

### Quality bar

- 100% pass rate against reference fixtures from `eidolon-system`.
- The runner itself has unit and integration tests; a fixture’s expected output and the runner’s check logic stay separated.
- Published artifacts include source jars and dependency-license reports.
- Coverage > 85% via Kover.

### Release and distribution

Maven artifact published to Reliquary. Version aligned with spec versions: `1.0.0` of the runner runs against `1.0.0` of the spec. CI publishes on git tag push.

### Conformance obligations

This repo runs conformance; it carries no conformance obligations of its own. Its tests verify it correctly reports passes and failures against fixture variants.

### Notes for Claude Code session

The runner stays language-and-target-agnostic. It accepts a target via SPI or programmatic configuration; never hardcodes Eidoshim, Eidolens, or any specific implementation. When in doubt, ask: “would this work for a hypothetical third-party implementation?” If no, refactor.

-----

## Charter: `eidoshim`

### Purpose and scope

Reference enforcer for the Eidolon Policy Format. Multi-call shim binary that intercepts CLI invocations via PATH-ordered symlinks, applies signed policy from a configured source, and emits structured events to a configured sink.

Compiles to a GraalVM native-image for sub-30ms cold start. Runs as `argv[0]`-aware busybox-style: a single binary symlinked under many command names.

**Deliberately out of scope:** Sandbox-grade isolation (use bubblewrap/landlock/containers for that). Real-time OS-level interception (eBPF, syscall hooks). Policy authoring. Event storage.

### Position in architecture

Depends on `eidolon-system` for spec, JSON Schema, and the policy/events formats. Depends on `eidolon-conformance-jvm` as a dev dependency for testing. Receives signed policy bundles produced by `eidocon` (or any conforming author). Emits events consumable by `eidolens` (or any conforming consumer).

### Public surface

A single native binary. Behavior:

- Reads `argv[0]`’s basename to determine which command it impersonates.
- Looks up policy from a configured source (file, HTTP URL, OCI ref).
- Verifies signature, validates schema, checks revocation, evaluates rules.
- On allow: forwards to the real binary via fresh PATH lookup, proxies stdio, captures completion data.
- On deny: returns a non-zero exit and emits `call_denied`.
- Emits events to a configured sink (stdout JSONL, file, HTTP webhook, OTLP).

CLI mode (when invoked as `eidoshim` directly rather than as a symlink) provides:

- `eidoshim install` — installs symlinks for configured commands
- `eidoshim status` — shows current policy, trust root, recent events
- `eidoshim verify <bundle>` — dry-run verification of a policy bundle
- `eidoshim test <command...>` — evaluates a hypothetical call without forwarding

### Dependencies

- Kotlin 2.x
- GraalVM 24+ for native-image
- Bouncy Castle for PGP verification (with explicit reflection config)
- Clikt for CLI parsing
- A pure-Kotlin JSON parser (kotlinx.serialization) — not Jackson, to keep native-image binary small
- A pure-Kotlin canonical JSON parser (or write a minimal one — RFC 8785 is small)
- networknt JSON Schema validator (with native-image config)

Notably absent: Spring Boot, HOCON parser, YAML parser. Eidoshim only reads canonical JSON; authoring formats stay in eidocon’s domain.

### Quality bar

- Cold start under 30ms on M-series Mac and Linux x86_64.
- Policy decision latency under 1ms after policy load.
- Signature verification under 100ms.
- Event emission overhead under 0.5ms.
- Passes all five layers of enforcer conformance.
- Cross-implementation conformance with the JVM evaluator from `eidocon-dsl` test harness.
- Coverage > 85% via Kover (excluding native-image-only code paths).

### Release and distribution

Native binaries built per platform: `darwin-arm64`, `darwin-x86_64`, `linux-x86_64`, `linux-arm64`. Published to Reliquary. Future Homebrew formula and Linux package distributions.

Built via GitHub Actions on Hestia self-hosted runners.

### Conformance obligations

Strict enforcer. Publishes a conformance report with each release. Targets cross-implementation conformance against the JVM evaluator.

### Notes for Claude Code session

The native-image constraint shapes every choice. Reflection requires explicit config; classes hidden behind dynamic loading break at runtime; library size matters because every megabyte hits cold-start latency. When suggesting libraries, default to small/zero-dependency Kotlin libraries over JVM frameworks. Spring Boot, Jackson, and broad reflection-heavy libraries don’t belong here.

The PATH-shim mechanism is subtle: the binary must compute a forwarded-process PATH that excludes its own directory, otherwise child processes loop into the shim. The Spektr file-watcher pattern is adaptable for hot-reload of policy.

-----

## Charter: `eidocon`

### Purpose and scope

Reference policy authoring/signing/distribution toolchain. Reads policies in HOCON, YAML, Kotlin DSL, or raw JSON; validates against the schema; emits canonical JSON; signs with a scoped key; pushes to a distribution backend; manages revocation.

Compiles to a GraalVM native-image so it runs on every CI agent and developer machine without a JVM. Also publishes a JVM JAR for environments that already have one.

**Deliberately out of scope:** Policy enforcement (Eidoshim’s domain). Event observation. Trust root rotation tooling (deliberate operation, not automated CLI).

### Position in architecture

Depends on `eidolon-system` for spec, JSON Schema. Optionally depends on `eidocon-dsl` (when authoring in Kotlin DSL). Uses `eidolon-conformance-jvm` as a dev dependency for testing.

Produces signed bundles consumed by Eidoshim or any conforming enforcer.

### Public surface

CLI subcommands:

- `eidocon compile <input>` — parse any authoring format, emit canonical JSON; `--canonical` flag for byte-stable output.
- `eidocon sign <canonical-json>` — sign with a scoped key (PGP via 1Password integration).
- `eidocon push <signed-bundle>` — distribute to configured backend (Reliquary or OCI registry).
- `eidocon revoke <key|signature>` — append to revocation list, re-sign with trust root key.
- `eidocon convert <input> --to <format>` — round-trip between HOCON, YAML, raw JSON.
- `eidocon validate <input>` — schema validation only, no signing.

The CLI takes input via stdin or file paths; outputs to stdout or file paths. Designed for shell pipelines.

### Dependencies

- Kotlin 2.x
- GraalVM 24+ for native-image
- Clikt for CLI parsing
- Typesafe Config for HOCON parsing
- snakeyaml-engine for YAML parsing
- networknt JSON Schema validator
- Bouncy Castle for PGP signing
- AWS SDK v2 for S3-compatible Spaces (Reliquary)
- An OCI client (lean oras-java) for OCI distribution
- 1Password CLI shell-out for key retrieval (avoids embedding secrets)

### Quality bar

- Passes all three layers of author conformance.
- Round-trip equivalence verified across HOCON, YAML, Kotlin DSL, raw JSON via golden-file CI test.
- Cold start under 50ms (slightly more lenient than Eidoshim because eidocon runs less frequently).
- Coverage > 85% via Kover.

### Release and distribution

Both native binaries (per-platform via Reliquary) and a JVM JAR (Maven via Reliquary) published. CI on Hestia.

### Conformance obligations

Strict author. Publishes a conformance report with each release.

### Notes for Claude Code session

The HOCON/YAML/Kotlin/JSON-to-canonical-JSON pipeline is the single most important code path. Bugs here break signatures silently. The golden-file test should run on every PR and block merge on any byte-level divergence.

The 1Password integration has historical baggage; reuse the patterns from Konstellation rather than designing fresh.

-----

## Charter: `eidocon-dsl`

### Purpose and scope

Konstellation-generated Kotlin DSL for authoring Eidolon policies. Provides type-safe builders for every scope and matcher, enforced at compile time via sealed hierarchies.

Also provides a pure-Kotlin policy evaluator (the JVM evaluator) for testing policies before signing. The evaluator mirrors Eidoshim’s logic; the cross-implementation conformance test verifies they produce identical decisions.

**Deliberately out of scope:** Direct enforcement (Eidoshim’s domain). Distribution (Eidocon’s domain). Schema definition (lives in `eidolon-system`).

### Position in architecture

Depends on `eidolon-system` for the policy schema. Generated by Konstellation from that schema. Used by `eidocon` (when authoring in Kotlin DSL) and by any user wanting type-safe policy authoring or in-test policy evaluation.

### Public surface

- Kotlin DSL builders: `policy { scopes { network { rule(...) { match { host { suffix(".anthropic.com") } } } } } }` style.
- A `PolicyEvaluator` class: takes a parsed policy and a call, produces a decision. Mirrors Eidoshim’s evaluation logic.
- A `PolicyTestHarness` class for Spock/JUnit: assert that a given DSL policy produces an expected decision for a given call.
- Maven coordinate `io.violabs.eidolon:eidocon-dsl` published to Reliquary.

### Dependencies

- Kotlin 2.x
- Konstellation runtime (for DSL generation infrastructure)
- A pure-Kotlin JSON serializer (kotlinx.serialization) for canonical JSON output

No HOCON, YAML, or signing dependencies — those belong to `eidocon`.

### Quality bar

- DSL produces canonical JSON byte-identical to the same logical policy expressed in HOCON, YAML, raw JSON.
- The JVM evaluator passes the same enforcer conformance fixtures Eidoshim does.
- Cross-implementation conformance against Eidoshim runs nightly.
- Coverage > 90% (smaller surface, higher bar).

### Release and distribution

Maven artifact via Reliquary. Versioned aligned with the policy spec version.

### Conformance obligations

Tested as part of `eidocon`’s author conformance for the DSL → canonical JSON path. Tested as part of cross-implementation enforcer conformance for the evaluator.

### Notes for Claude Code session

Konstellation does the heavy lifting on DSL generation. The annotations on the schema declare what gets generated; manual builder code should be minimal. When the DSL needs a feature Konstellation doesn’t support, propose a Konstellation enhancement rather than hand-rolling.

The evaluator and Eidoshim share semantics, not code. Cross-implementation conformance enforces this. Don’t try to share a code module — that defeats the dual-implementation safety property.

-----

## Charter: `eidolens`

### Purpose and scope

Reference event consumer. Spring Boot WebFlux backend ingesting events via HTTP POST (JSONL) and OTLP, persisting to PostgreSQL via R2DBC, exposing a query API for the SvelteKit frontend.

**Deliberately out of scope:** Event production. Policy enforcement. Frontend rendering (separate repo).

### Position in architecture

Depends on `eidolon-system` for spec and JSON Schema. Uses `eidolon-conformance-jvm` as a dev dependency for testing. Consumed by `eidolens-svelte` over HTTP.

### Public surface

HTTP REST API (camelCase, since this is the application boundary, not the spec wire boundary):

- `POST /api/v1/events` — bulk ingest (JSONL or JSON array)
- `POST /api/v1/events/otlp` — OTLP ingestion
- `GET /api/v1/events?...` — query with filters (skill, command, decision, time range)
- `GET /api/v1/skills/{skillId}/events` — per-skill view
- `GET /api/v1/sessions/{sessionId}/events` — per-session view
- `GET /api/v1/calls/{callId}/lifecycle` — assemble a single call’s full lifecycle

The internal Kotlin uses camelCase domain models with Jackson `SnakeCaseStrategy` translating at the spec wire boundary.

### Dependencies

- Kotlin 2.x
- Spring Boot WebFlux 3.x
- R2DBC PostgreSQL
- Liquibase for migrations
- Jackson for JSON
- Reactor for reactive stream handling
- An OTLP receiver library (lean opentelemetry-java)

### Quality bar

- Strict consumer conformance: passes all three categories (A, B, C).
- Ingestion rate target: 10K events/sec on a single instance.
- Query p99 latency under 500ms for typical queries.
- Coverage > 80%.

### Release and distribution

Docker image built via Spring Boot’s buildpacks support. Deployable to DigitalOcean App Platform or self-hosted Hestia. Helm chart deferred to Phase 7.

### Conformance obligations

Strict consumer. Publishes a conformance report with each release documenting Category C handling (which spec-violating events get rejected vs quarantined vs logged).

### Notes for Claude Code session

The Vaultara pattern is the template for everything: directory layout, R2DBC patterns, Liquibase migration shape, four-eyes-where-applicable, BCrypt for any auth that needs it. Don’t redesign — adapt.

The HTTP API uses camelCase; the wire format coming from producers uses snake_case; Jackson handles translation. Domain models stay camelCase throughout the application.

OTLP support is a “second ingestion shape” not a “core feature.” If OTLP integration requires significant rework, defer it to Phase 7.

-----

## Charter: `eidolens-svelte`

### Purpose and scope

Frontend for Eidolens. SvelteKit app rendering the event stream, per-skill views, query interface, and dashboards.

**Deliberately out of scope:** Direct event ingestion. Policy authoring or enforcement. Authentication (handled at the deployment layer or by Eidolens, not the frontend).

### Position in architecture

Consumes Eidolens HTTP API. Doesn’t directly consume any spec — it sees only what Eidolens exposes (camelCase, application-shaped).

### Public surface

Web app served at `/` with routes:

- `/` — dashboard with live event stream and recent activity
- `/events` — searchable event log
- `/skills` — per-skill view
- `/skills/{id}` — drill-down into a specific skill’s behavior
- `/calls/{id}` — single call lifecycle view
- `/sessions/{id}` — session timeline

### Dependencies

- SvelteKit 2.x
- TailwindCSS for styling
- A charting library (lean Chart.js or Recharts equivalent for Svelte)
- A virtualization library for the event stream view (for high event counts)

### Quality bar

- Fully responsive (mobile, tablet, desktop).
- Accessible (WCAG 2.1 AA target).
- Initial page load under 1s on broadband.
- Live event stream renders without lag at 100 events/sec sustained.

### Release and distribution

Bundled with Eidolens Docker image (served as static assets by the Spring Boot backend), or deployed independently to DigitalOcean App Platform with reverse-proxy routing.

### Conformance obligations

None — frontend, doesn’t consume specs directly.

### Notes for Claude Code session

The Vaultara SvelteKit app is the design template for layout, navigation, and component structure. Match its aesthetic.

camelCase throughout — the spec’s snake_case never reaches this codebase. Eidolens does the translation.

State management lean: Svelte stores for global state, page-level state otherwise. Avoid heavyweight state management libraries.

-----

# Section 3: Cross-Repo Conventions

These decisions hold identically across all repos. Drift here costs hours of refactoring later.

## Directory layout (Kotlin/Gradle repos)

```
.
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── release.yml
│   │   └── conformance.yml (where applicable)
│   └── dependabot.yml
├── docs/
│   └── (repo-specific docs; charter copy lives here)
├── gradle/
│   └── wrapper/
├── src/
│   ├── main/
│   │   ├── kotlin/
│   │   └── resources/
│   ├── test/
│   │   ├── kotlin/
│   │   └── resources/
│   └── conformance/   (where applicable; separate source set)
├── build.gradle.kts
├── gradle.properties
├── settings.gradle.kts
├── README.md
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
├── SECURITY.md         (KSA-EIDO scheme reference)
└── .gitignore
```

Native-image projects (eidoshim, eidocon) add:

```
├── src/main/resources/META-INF/native-image/
│   ├── reflect-config.json
│   ├── resource-config.json
│   ├── proxy-config.json
│   └── jni-config.json
```

## Gradle and build conventions

**Kotlin version:** 2.1.x or latest stable. Pinned in `gradle.properties` as `kotlin.version`.

**Gradle version:** Latest stable (use the wrapper). Updates via Dependabot.

**JVM target:** 17 LTS for non-native-image builds (matches the `khorum-oss/kotlin-project-template` toolchain pin); native-image builds use the GraalVM JDK matched to the GraalVM version in CI.

**Plugin pattern:** Use the plugin DSL (`plugins { ... }` block) exclusively. No legacy `apply plugin:` calls.

**Convention plugins:** A `build-conventions/` composite build defines shared configuration. Each repo includes it. The conventions plugin establishes:

- Kotlin compiler options (`-Xjsr305=strict`, `-Xexplicit-api=strict`, etc.)
- Common test dependencies
- Jacoco/Kover configuration
- Dependency-license reporting (CycloneDX or similar)
- SonarCloud configuration

**Dependency management:** A version catalog (`gradle/libs.versions.toml`) per repo. The conventions plugin can reference shared versions.

**Native-image specifics (eidoshim, eidocon):** GraalVM plugin (`org.graalvm.buildtools.native`). Reachability metadata via the `metadata-repository` setting. Build-time PGO when targeting performance-critical paths.

## Kotlin style

- **Explicit API mode** (`-Xexplicit-api=strict`) on all library modules. Forces explicit visibility on public API.
- **Indentation:** 4 spaces, no tabs.
- **Line length:** 120 characters.
- **Imports:** No wildcard imports; star imports forbidden.
- **Naming:**
  - Classes/objects: `PascalCase`
  - Functions/properties: `camelCase`
  - Constants: `UPPER_SNAKE_CASE`
  - Top-level constants in companions: same
- **Null handling:** Prefer non-nullable types; use `?` only where null carries semantic meaning; avoid `!!` except in test code with documented justification.
- **Coroutines:** Structured concurrency throughout. No `GlobalScope`. Use `CoroutineScope` injection where lifecycle matters.
- **Data classes:** For value-shaped types only. For domain entities with identity, plain classes with explicit `equals`/`hashCode`.
- **Sealed hierarchies:** Use them for closed sets (decision types, scope types, error categories). Beats enums when state varies per case.

Detekt + ktlint configuration shared via the convention plugin. Pre-commit hooks run both.

## Error handling

- **Sealed result types** for operations with multiple expected failure modes:

```kotlin
sealed interface PolicyLoadResult {
    data class Success(val policy: Policy) : PolicyLoadResult
    data class SignatureInvalid(val reason: String) : PolicyLoadResult
    data class SchemaInvalid(val errors: List<ValidationError>) : PolicyLoadResult
    data class FetchFailed(val cause: Throwable) : PolicyLoadResult
}
```

- **Exceptions** for unexpected/programmer errors (NPE, illegal state). Typed exceptions for cross-module boundaries:

```kotlin
sealed class EidolonException(message: String, cause: Throwable? = null) : Exception(message, cause)
class PolicyVerificationException(message: String, cause: Throwable? = null) : EidolonException(message, cause)
class CanonicalFormException(message: String, cause: Throwable? = null) : EidolonException(message, cause)
```

- **No exceptions across the FFI boundary.** Native code returns sealed result types; the JVM side translates if it wants exceptions.

## Logging

- **Structured logging only.** Plain string logs forbidden in production code paths.
- **Logger choice:** `kotlin-logging` wrapping SLF4J for JVM code; a custom thin abstraction for native-image code (no SLF4J — too much reflection).
- **Log levels:**
  - `ERROR`: an operation failed; user/operator action needed
  - `WARN`: degraded operation; might recover
  - `INFO`: significant lifecycle events
  - `DEBUG`: diagnostic detail
  - `TRACE`: developer-targeted, off in production
- **MDC keys** (consistent across repos): `event_id`, `call_id`, `session_id`, `skill_id`, `policy_id`, `producer_instance_id`. Same names as the wire format for grep-ability.
- **Format:** JSON in production. Plain in development.

## Testing

- **Frameworks:** JUnit 5 + Spock 2.x (Spock for behavioral, JUnit for plain). Spock pairs well with the existing khorum-oss patterns.
- **Naming:** test classes match production classes plus `Test` or `Spec` suffix. Test methods use full sentences with backticks: ``compiles HOCON to canonical JSON byte-identically to YAML`()`.
- **Coverage:** 85% minimum per repo (90% for eidocon-dsl). Kover enforces in CI.
- **Test categories:**
  - **Unit:** pure logic, no I/O, fast. The bulk of tests.
  - **Integration:** with real dependencies (PostgreSQL via Testcontainers, Spaces via mock S3, etc.).
  - **Conformance:** runs the conformance suite against the implementation. Separate Gradle source set, separate CI job.
- **Fixture organization:** `src/test/resources/fixtures/<category>/<name>/`. Each fixture self-contained.
- **Property-based testing:** Use Kotest’s PBT for canonical-form round-trip tests. Generate arbitrary policies, round-trip through HOCON/YAML/Kotlin/JSON, verify byte-identical output.

## CI/CD

GitHub Actions on Hestia self-hosted runners (with cloud-hosted GitHub-provided runners as fallback for OS coverage native-image builds need).

**Standard workflows per repo:**

- `ci.yml` — runs on every PR. Steps: checkout, Gradle build, unit tests, integration tests, coverage report, SonarCloud, CodeQL, dependency-license check.
- `release.yml` — runs on tag push. Steps: build native binaries (where applicable), build JAR, sign artifacts (PGP via 1Password), publish to Reliquary, create GitHub release.
- `conformance.yml` — runs nightly. Runs the full conformance suite against the latest main. Posts results back to the repo as a check.

**Quality gates that block merge:**

- All tests pass
- Coverage at threshold
- SonarCloud quality gate passes
- CodeQL clean (no high/critical findings)
- License check clean (no incompatible licenses introduced)
- Dependabot security alerts clean

**Branch strategy:** Trunk-based. Feature branches short-lived (under a week). PRs squash-merge. Tags only on main.

## KSA-EIDO security advisory scheme

Every repo includes `SECURITY.md` referencing the scheme:

```
# Security

This repository participates in the Khorum-OSS Software Assurance scheme.
Vulnerabilities receive identifiers of the form KSA-EIDO-YYYY-NNNN.

Report vulnerabilities to: khorum.oss@gmail.com (PGP key: <fingerprint>)
Disclosure policy: 90-day standard window with extension for complex cases.
Advisory archive: https://eidolon.khorum-oss.org/advisories/
```

When a vulnerability is fixed, the advisory publishes to `eidolon.khorum-oss.org/advisories/KSA-EIDO-YYYY-NNNN/` with:

- Affected versions
- Fixed versions
- CVSS score
- Detailed description (after disclosure window)
- Acknowledgments

The advisory site is generated from a separate repo (deferred — for now, advisories live in the affected repo’s `docs/advisories/`).

## Versioning

**SemVer everywhere**, with pre-1.0 caveat: every repo starts at `0.x.y` until the public API stabilizes. `0.x` increments are minor (additive); `0.x` to `0.(x+1)` may include breaking changes during pre-1.0.

**Spec versioning** (eidolon-system repo) versions independently from implementation versioning. An implementation can target spec `1.0.0` while internally being `0.5.2`.

**Maven artifact versioning** matches the source repo’s tag. Reliquary stores all historical versions; latest tag determines what’s “current.”

**Native binary versioning** matches the source repo’s tag. Released as `<repo>-<version>-<platform>.tar.gz` (or `.zip` for Windows, deferred).

## Documentation

**Per-repo README** structure:

1. One-paragraph description (lift from charter)
1. Quick start (install + minimal example)
1. Documentation links (to eidolon.khorum-oss.org/<repo>/)
1. Contributing pointer (to CONTRIBUTING.md)
1. License

**Spec docs** live in `eidolon-system` only. Other repos link to them; they don’t duplicate.

**API docs** generated from KDoc via Dokka (`./gradlew dokkaHtml`). Published to `eidolon.khorum-oss.org/<repo>/api/`.

**Changelog** in `CHANGELOG.md`, Keep-a-Changelog format. Updated as part of every PR.

**Architecture decision records** (ADRs) under `docs/adr/`. Numbered, append-only, used for choices that future contributors might question.

## Naming conventions

**Repo names:** lowercase, hyphenated, scoped to the suite (`eidolon-system`, `eidoshim`, `eidocon`, `eidolens`).

**Maven coordinates:** `io.violabs.eidolon:<artifact>` for everything in the suite.

**Package names:** `io.violabs.eidolon.<repo-purpose>`. E.g., `io.violabs.eidolon.shim` for eidoshim, `io.violabs.eidolon.policy` for eidocon-dsl, `io.violabs.eidolon.lens` for eidolens.

**Container images:** `khorum-oss/<repo-name>:<version>` — pushed to GHCR.

**S3 paths in Reliquary:** `eidolon/<repo>/<version>/<artifact-filename>`.

**Shared concept names** (use these consistently):

- “bundle” — signed unit
- “policy” — logical policy within a bundle
- “rule” — single matcher-plus-decision unit
- “scope” — network/filesystem/exec/env
- “matcher” — a single matching condition within a rule
- “envelope” — the signature wrapper around canonical bytes
- “manifest” — reserved for future per-skill policy bundles (per Open Questions)

Avoid these synonyms:

- “config” for bundle/policy (reserved for application configuration)
- “rules” for policy (a policy contains rules; they aren’t synonyms)
- “permission” for matcher or rule (different conceptual model)

-----

# Appendix: How to Use This Document with Claude Code

The intended workflow:

**Step 1: Set up the umbrella repo first.**

Create the `eidolon-system` GitHub repo. Drop Section 1 of this document into it as `README.md` plus `docs/specs/event-format-1.0.0.md` and `docs/specs/policy-format-1.0.0.md`. Write the JSON Schema files for events and policies (one Claude Code session per schema is reasonable).

**Step 2: For each implementation repo, run a Claude Code session.**

For each of `eidolon-conformance-jvm`, `eidoshim`, `eidocon`, `eidocon-dsl`, `eidolens`, `eidolens-svelte`:

1. Create the empty GitHub repo.
1. Drop a `CONTEXT.md` file at the root containing:
- The repo’s charter (Section 2 of this document)
- The cross-repo conventions (Section 3)
- A link to the master spec (Section 1, hosted at eidolon.khorum-oss.org)
1. Open Claude Code in the repo, run `/plan` with: “Implement this repository per CONTEXT.md, satisfying its conformance obligations.”
1. Claude Code reads CONTEXT.md and produces a grounded implementation plan against the actual filesystem.
1. Iterate the plan until satisfied, then execute.

**Step 3: For sequencing.**

The right order to bring up repos:

1. `eidolon-system` — specs and fixtures must exist before anything else can target them.
1. `eidolon-conformance-jvm` — gives every other repo a way to verify itself.
1. `eidocon-dsl` — depends only on the schema.
1. `eidoshim` and `eidocon` in parallel — both target the policy spec; both depend on conformance-jvm.
1. `eidolens` — depends on the events spec and conformance-jvm.
1. `eidolens-svelte` — last; depends on Eidolens HTTP API.

This order keeps dependencies forward-pointing. No repo waits on a repo that hasn’t started.

**Step 4: When in doubt about a cross-repo decision, update this document first.**

If you’re in a agent session such as claude code for `eidoshim` and discover that the canonical-form rules need clarification, don’t just patch eidoshim. Update the spec in `eidolon-system` first, propagate the change to other repos that referenced the affected text, then implement in eidoshim. The spec stays the source of truth.

The hardest discipline is resisting the urge to make repo-local choices that diverge from the suite. Every divergence costs you later. When in doubt, ask: “if a third party were implementing this against the spec, would they make the same choice?”