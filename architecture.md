# Architecture

## Scope

Kernel Sieve is currently scoped to one production target:

- x64 WDM drivers
- Windows Server 2022 guest images
- static first triage with manifest generation
- dynamic confirmation on QEMU/KVM, with TCG kept as a slower fallback lane

The repo is split into a static side and a dynamic side because they solve different problems:

1. `static/` recovers request surfaces, sink provenance, and dynamic handoff artifacts from a raw `.sys`.
2. `dynamic/` takes that handoff, executes the candidate in a disposable guest, and emits an evidence-backed verdict.

## End-to-End Flow

The current pipeline is:

```text
.sys
  -> Stage 1: PE intake and fingerprinting
  -> Stage 2: CFG recovery, dispatch recovery, IOCTL extraction
  -> Stage 3: provenance taint and request modeling
  -> Stage 4: local verification, proof grading, dynamic handoff packaging
  -> Stage 5: dynamic confirmation and verdict normalization
```

The important contract boundary is between Stage 4 and Stage 5. Static analysis emits a handoff tree centered on:

- `verification.json`
- `dynamic_cases/`
- `requests/`
- `models/`
- `pocs/`
- `snapshots/`
- `installation-plan.json`
- `handoff.json`

The dynamic runner never needs to rediscover the driver surface from scratch. It consumes the handoff and focuses on execution, evidence collection, crash attribution, and resumability.

## Clean Repo Layout

The cleaned tree keeps only the pieces required to build, run, and explain the system:

- `static/`
  - native analysis code
  - scanner and metric assertion tools
- `dynamic/`
  - Python host runner
  - guest agent source
  - observer driver source
  - synthetic fixtures and tests
- `vendor/`
  - `xair/` - vendored IR/lifter core
  - `xair_cfg/` - vendored CFG recovery library
  - `xair_sym/` - vendored symbolic execution library
  - `zydis/` - vendored decoder dependency
  - `z3/` - vendored solver package used by `xair_sym`
- `runtime/`
  - `vm/win2022-golden.qcow2`
  - imported guest toolchain under `toolchains/dynamic/<commit>/`
- `docs/`
  - this architecture note
  - sink catalog and execution matrix
  - per-stage implementation notes

The clean repo no longer requires the old absolute dependency roots under
`C:\Users\Jaden\Desktop\Projects\IR\...`. The current vendored snapshots were
taken from:

- `vendor/xair`: commit `f94f9cb`
- `vendor/xair_cfg`: commit `b5062f7`
- `vendor/xair_sym`: commit `9c749a4`
- `vendor/zydis`: upstream `5.0.0.0`
- `vendor/z3`: Windows package `4.13.0`

## Runtime Model

The primary operational path is now the single Docker image and the `kernel-sieve`
wrapper it exposes.

Static stages `1-4` still run as the native scanner inside the container. Dynamic
confirmation runs from the same image and uses:

- the bundled Windows Server 2022 qcow base image
- qcow overlays per case
- QEMU Guest Agent transport instead of WinRM or SSH
- a guest-side executor that performs the baseline -> variant -> baseline sequence
- optional observer-assisted crash attribution

The wrapper exposes one interface for all supported combinations:

- `kernel-sieve run driver --stages 1-4`
- `kernel-sieve run driver --stages 5`
- `kernel-sieve run driver --stages 1-5`
- `kernel-sieve run corpus --stages ...`

When stage `5` is requested for a corpus run, the wrapper automatically materializes:

- `installation-matrix.json`
- `queue-funnel-report.json`
- queue-backed per-driver dynamic results under the chosen result root

KVM is still the preferred accelerator for ordinary dynamic runs. TCG remains the
supported fallback lane for privileged register, port I/O, and synthetic hardware
contracts.

The imported guest toolchain lives under `runtime/toolchains/dynamic/<commit>/`. The
dynamic runner expects that toolchain to be at least as new as the checked-out
`dynamic/guest-agent` and `dynamic/observer-driver` sources. If those sources change,
the Windows artifacts must be rebuilt and re-imported before relying on the default
runtime-toolchain guard.

## Operational Statistics

Kernel Sieve does emit rich timing and corpus statistics, but the authoritative numbers
live in run artifacts rather than this document.

The important statistic sources are:

- static scan timing:
  `driver_timings.jsonl` under the chosen static output root
- per-driver static summaries:
  `verification.json` and `handoff.json` under `artifacts/static/<sha256>/`
- queue eligibility counts:
  `queue-funnel-report.json`
- per-driver dynamic execution summaries:
  `driver-result.json`
- corpus-level dynamic execution summaries:
  `queue-run-summary.json`

This matters operationally because the corpus, proof mix, and queue-eligible set change
whenever:

- the static recovery rules change
- sink contracts change
- installation probing changes
- the target corpus itself changes

The docs therefore describe what each artifact means, while the artifacts themselves
carry the live counts, timings, and verdict totals for a specific run.

## Design Choices

The system is intentionally deterministic first.

- Static analysis tries to recover exact device paths, exact IOCTLs, request layouts, and sink argument provenance before dynamic execution is attempted.
- Dynamic execution reuses that structure instead of rediscovering it heuristically in the guest.
- Resumability is a first-class property: static scans write per-driver artifacts; dynamic runs write incremental `driver-result.json` state and reuse completed per-case results.

## Evidence Boundary

A reproduced behavior is not automatically a security vulnerability.

The dynamic stage separates:

- request plausibility
- sink reachability
- manifest control of the sink arguments
- unsafe effect reproduction
- security boundary impact

That separation is why many cases currently land as `inconclusive` or `unknown`: the harness is conservative about claiming impact without exact sink-specific evidence.
