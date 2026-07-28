# Stage 5 - Dynamic Confirmation

## Purpose

Stage 5 consumes the Stage 4 handoff and turns it into execution evidence.

The dynamic stage does not rediscover the driver surface. It trusts the static handoff
as the contract, normalizes it, chooses a lane, runs the case, and then grades the
result with a conservative verdict model.

The implementation is split across:

- `dynamic/src/ks_dynamic/normalize.py`
- `dynamic/src/ks_dynamic/support.py`
- `dynamic/src/ks_dynamic/runner.py`
- `dynamic/src/ks_dynamic/verdict.py`
- `dynamic/src/ks_dynamic/cli.py`

The normal entrypoint for operators is the top-level wrapper:

- `kernel-sieve run driver --stages 5`
- `kernel-sieve run driver --stages 1-5`
- `kernel-sieve run corpus --stages 5`
- `kernel-sieve run corpus --stages 1-5`

The lower-level `ks-dynamic` commands still exist inside the image, but the wrapper is
the supported production interface because it wires stage selection, queue planning,
runtime defaults, and resumable output layout together.

## Normalization

Before execution, `normalize_case()` repairs contract inconsistencies in the static manifest.

Important normalization behaviors:

- backfill `parent_candidate_id`
- infer `request.surface` from invocation data
- refine sink-specific oracles when a stronger descriptor exists
- infer missing allocation kinds for runtime objects
- normalize `pre_call_fixups`
- normalize input and output writes into `pre_call_writes`
- carry forward `normalization_warnings`

This matters because the support decision is made on the normalized case, not the raw
JSON. That is why a static manifest can be `dynamic_manifest_complete` and still end up
`guided_supported` or `observability_only` after normalization.

## Static Readiness Fields

The dynamic stage consumes several booleans emitted by Stage 4.

### `dynamic_ready`

Strongest static readiness bit. It means the static side believes the case is exact
enough for automatic confirmation:

- request synthesized
- exact dispatch/path/device story
- obligation proven
- sink-specific oracle exists
- oracle self-test exists
- no runtime device discovery required
- guards recovered cleanly

`dynamic_ready` does not mean the harness can definitely run it on every machine. It
only means the static side produced the strongest class of handoff.

### `dynamic_manifest_complete`

The manifest has the pieces needed to attempt execution:

- synthesized request
- oracle assigned
- `candidate_id`
- `execution_recipe_key`
- at least one device path candidate

This is weaker than `dynamic_ready`. Many manifest-complete cases still need guided or
observability-only handling.

### Sink-specific oracle availability

This is the difference between:

- a sink-specific oracle such as `mm_section_unmap`, `process_open`, or `file_write`
- a family-level oracle such as `process_handle_canary` or `object_registry_canary`

Sink-specific oracles can disqualify or confirm a specific modeled behavior more
aggressively. Family-level oracles are still useful, but they usually lead to
`guided_supported` rather than `automatic_supported`.

### Support status after normalization

After `normalize_case()`, `explain_case_support()` returns one of:

- `automatic_supported`
- `guided_supported`
- `observability_only`
- `unsupported`
- `harness_error` (internal runner issue, not a static case property)

These are the real execution classes the runner acts on.

## What the Support Outcomes Mean

### Exact and ready for automatic execution

This means:

- static `dynamic_ready == true`
- normalized support status is `automatic_supported`
- the selected lane is implemented

These cases can be run and scored strictly. A clean negative result on an exact case is
meaningful.

### Runnable but guided because the oracle inputs are incomplete

This means:

- the normalized support status is `guided_supported`
- the remaining gap is something the setup probe can repair automatically, such as
  unresolved service installation or an unconfirmed device path

Common reasons:

- family-level oracle only
- sink-specific allocation kinds missing
- oracle context incomplete
- exact device path not known yet
- service dependencies incomplete

The default automatic queue does not run every `guided_supported` case. It first runs
the installation probe, reapplies support on the repaired manifest, and only promotes
the case into the executable queue if the remaining support gaps are gone.

### Observability-only

This means:

- the case still has meaningful static evidence, but the current harness only has
  reachability or family-level observation for it

Typical causes:

- `instruction_trace` required
- hardware model required
- observer lane required
- kernel-caller lane required
- boot-start installation workflow

These cases stay out of the default automatic queue. They remain inventory artifacts
for manual or explicitly guided follow-up, and they are only run automatically if a
user chooses a separate observability campaign.

### Unsupported

This means the harness should not attempt execution with the current production scope.

Common causes:

- not `x86_64`
- no sink descriptor
- unsupported invocation surface
- unsupported request method
- unknown or manual-only installation mode
- incomplete driver package

## Lane Selection

The runner picks a lane from the normalized support decision:

- `kvm_fast`:
  default fast path for automatic and many guided cases
- `kvm_observer`:
  KVM plus observer-driver assistance
- `reboot_pnp`:
  reboot-capable install lane
- `tcg_instrumented`:
  instruction-trace lane for privileged register and port I/O sinks
- `tcg_test_device`:
  TCG plus synthetic hardware model, used for physical-memory mapping style cases

The installation probe is intentionally different. Driver setup probing is not sink
execution, so `probe_driver_setup()` always chooses the cheapest viable boot lane for
load-and-open validation:

- `kvm_fast` for ordinary SCM and PnP setup probes
- `reboot_pnp` only when the installation metadata itself requires a reboot lane

The probe still records the candidate's normalized sink lane in diagnostics, but the
actual probe VM lane is emitted separately as `probe_vm_lane`.

Unimplemented but planned lanes are still listed in the support layer:

- `kvm_kd`
- `kernel_caller`
- `forensic_replay`

## Execution Flow

`run-driver` in `cli.py` performs the driver-level orchestration.

For each selected case it:

1. loads `handoff.json` and `verification.json`
2. chooses the best record per `execution_recipe_key`
3. normalizes the dynamic case
4. computes `SupportDecision`
5. reuses an existing `dynamic_result.json` if resume is enabled and the case already completed
6. boots a disposable VM overlay
7. installs the driver
8. runs the manifest's `baseline -> variant -> baseline` sequence, unless the plan was normalized differently
9. collects guest results, crash artifacts, observer summaries, and optional instrumentation summaries
10. writes one `dynamic_result.json` per executed case
11. updates the per-driver `driver-result.json` incrementally

That incremental `driver-result.json` is why long corpus runs can resume cleanly after a
workstation reboot.

For corpus execution, `run-queue` consumes `queue-funnel-report` output and calls
`run-driver` only for drivers whose post-probe candidates are actually queue eligible.

## Stage 5 Outputs

### `dynamic_result.json`

Per executed case. It contains:

- `support`
- `verdict`
- execution metadata
- request summary
- evidence bundle
- crash attribution
- artifact paths
- elapsed time

This is the main per-case result.

### `driver-result.json`

Per driver. `run-driver` rewrites it after each case. It contains:

- driver identity
- toolchain/source commit pairing
- preflight receipts
- `resume_loaded`
- one result row per case

This is the dynamic-stage resumability anchor.

### `queue-run-summary.json`

Per queue execution. `run-queue` rewrites it after each driver and records:

- the source `queue-funnel-report` path
- result root
- resume mode
- one row per eligible driver
- per-driver elapsed time
- whether the driver run completed or raised an error

This is the corpus-level resumability anchor for the automatic queue.

### Installation probe diagnostics

Every `driver-installation-result.json` now separates three support snapshots:

- `support_before_probe`:
  the static manifest exactly as it came from Stage 4
- `probe_support`:
  the temporary probe-prepared manifest used only to decide how to stage and boot the
  setup probe
- `support_after_probe`:
  the repaired manifest after the probe confirmed service mode and trustworthy device
  paths

The diagnostics also carry:

- `selected_lane_before_probe`
- `selected_lane_for_probe`
- `probe_vm_lane`
- `selected_lane_after_probe`

That split matters because the candidate's sink lane is often slower than the lane
needed to answer the much narrower question "does this driver load and expose a device
we can open?"

## Verdict Model

The verdict system intentionally separates:

- reproduction status
- security claim
- evidence level
- confidence

The main verdict logic is in `evaluate()` in `verdict.py`.

### Evidence levels `E0` through `E5`

The current implementation defines six levels, but only five are emitted today.

- `E0`:
  no meaningful evidence, or the result is dominated by harness/manifest/path failure
- `E1`:
  the case ran enough to be informative, but the evidence is still weak or non-dispositive
- `E2`:
  reserved in the schema, currently not emitted by `evaluate()`
- `E3`:
  instrumented evidence reached the target sink, but not a user-visible confirmed unsafe effect
- `E4`:
  confirmed behavior or reproducible driver-attributed crash, but not a proven boundary crossing
- `E5`:
  strongest claim; controlled request plus proven vulnerable effect or boundary crossing

### How `E1` through `E5` map to real outcomes

- `E5`:
  proven vulnerability, or variant-only effect crossed the modeled security boundary
- `E4`:
  behavior confirmed or likely vulnerable crash
- `E3`:
  instrumented lane observed the sink and correlated it to the candidate
- `E1`:
  most inconclusive but still informative negatives and partial executions
- `E0`:
  harness failure, unsupported manifest, no guest result, or device path could not even be opened

### Unknowns by likelihood

There is no separate `unknown_likelihood` enum today. Unknowns are already stratified by:

- `evidence_level`
- `confidence`
- `support.status`
- `verdict.reason`

For production triage, the current code naturally separates unknowns like this:

1. highest-likelihood unknown:
   `E3` with host instrumentation proving the target sink was reached
   (`confidence` about `0.58` to `0.72`)
2. medium-likelihood unknown:
   `E1` guided cases that executed cleanly but only had family-level or incomplete oracle support
   (`confidence` about `0.42`)
3. lower-likelihood unknown:
   `E1` cases where the device opened or the IOCTL ran but no predicted effect was observed, and the request was not exact enough to disprove the model
   (`confidence` about `0.30` to `0.35`)
4. lowest-value unknown:
   `E0` path-open failures, unsupported manifests, and other harness-dominated failures
   (`confidence` `0.0` to `0.15`)

So if you want a practical ordering for unknowns, sort by:

1. `evidence_level` descending, with `E3` above `E1` above `E0`
2. `confidence` descending
3. `support.status`, preferring `guided_supported` and `observability_only` over hard failures

That gives a stable "most worth manual follow-up" ordering without inventing a second
parallel bucket system.

## Preflight and Resumability

Before real-driver execution, the harness can run the synthetic suites:

- transport
- copy
- state
- privileged
- crash

`run-driver` records those receipts in `driver-result.json`. If the same driver is run
again with resume enabled, completed `dynamic_result.json` files are reused and counted
under `resume_loaded`.

## Practical Bottleneck

The slow path is still crash handling and attribution, not ordinary `kvm_fast`
execution.

- normal KVM triage cases usually complete in seconds
- crash, dump recovery, and KD analysis cases dominate wall time

That is why the corpus runner is built around resumable per-driver state instead of one
monolithic batch job.
