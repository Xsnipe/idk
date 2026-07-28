# Stage 4 - Local Verification and Dynamic Handoff

## Purpose

Stage 4 is where Stage 3 candidates become either:

- a locally justified request model,
- a weaker but still useful guided handoff,
- or a static-only triage record.

The main implementation is `static/src/ks_verify.c`. Packaging into the portable handoff
tree is finished in `static/src/ks_cfg.c`.

The driver-level loop is `verify_flow()`.

## What Stage 4 Actually Verifies

For each `ks_flow_candidate`, Stage 4 computes:

- whether the request model is satisfiable at all
- which required sink arguments are rooted in request state
- which required sink arguments are concretely proven to satisfy the sink obligation
- whether fallback symbolic execution can recover a missing handler-to-sink proof
- whether a runtime oracle exists for this sink
- whether the candidate is precise enough to become a dynamic-ready case

The important record fields live in `ks_verification_record`:

- `obligation_feasible`
- `obligation_proven`
- `sink_arg_locally_rooted`
- `sink_arg_locally_proven`
- `fallback_reached_sink`
- `dynamic_oracle_available`
- `oracle_sink_specific`
- `oracle_self_test_supported`
- `dynamic_manifest_complete`
- `dynamic_ready`
- `proof_kind`
- `proof_strength`

## Local Symbolic Verification

The first proof pass is `prove_flow_with_symbolic_engine()`.

This is real symbolic execution, but it is not whole-driver exploration. It builds a
small XAIR verifier module with:

- one `dwIoControlCode` parameter
- four symbolic sink-argument parameters
- a symbolic request environment created by `request_symbolic_env_create()`

The local verifier then:

1. infers effective input and output sizes, including recovered guards;
2. describes the sink obligation for the target sink;
3. creates symbolic request-backed expressions for request fields and buffer loads;
4. resolves request bindings for sink arguments;
5. adds equality constraints when a buffer root must match a chosen request value;
6. assumes per-sink obligation expressions;
7. asks `xair_sym_check()` whether the local structured constraints are satisfiable.

If the query is satisfiable, Stage 4 records:

- `rooted_arg_mask`:
  required sink args that were connected to request roots
- `proven_arg_mask`:
  required sink args whose concrete witness values satisfy the sink obligation
- `constraint_count`
- `proof_kind`

This is why `local_exact` and `local_approximate` are stronger than plain binding
labels: they come from a local SAT-backed proof, not just from surface-level taint.

## How Symbolic Fallback Is Used

If the local proof is incomplete, Stage 4 may run `run_handler_fallback()`.

Fallback is triggered when:

- some required sink argument is still unproven, or
- the candidate has rank `>= 70` and no locally proven sink arguments

The fallback path is not whole-image symbolic execution either. It is a bounded
handler-to-sink slice:

- `build_handler_sink_slice()` slices the handler CFG down to the sink
- if the original CFG is insufficient, `build_focused_fallback_cfg()` rebuilds a
  focused CFG rooted at the handler, case entry, helper, and sink
- `initialize_binary_segments()` maps the driver image into the symbolic state
- `initialize_handler_entry_state()` creates a synthetic WDM environment:
  `DEVICE_OBJECT`, `DEVICE_EXTENSION`, `IRP`, `IO_STACK_LOCATION`, `SystemBuffer`,
  `Type3InputBuffer`, `UserBuffer`, and `MDL` as needed
- request bytes are made symbolic and tainted as `request_input`

The explore configuration is bounded:

- coverage search
- hybrid concretization mode
- `max_states = 4096`
- `max_block_steps = 65536`
- `max_visits_per_block = 128`

On the first sink hit, the callback captures:

- `reached_sink`
- `completed_states`
- `slice_block_count`
- an optional `xair_sym_snapshot`

That result feeds two different proof buckets:

- `fallback_symbolic_exact` or `fallback_symbolic_approximate`:
  the bounded symbolic slice actually reached the sink, and Stage 4 lifted missing
  sink operands back into request provenance
- `metadata_lift_exact` or `metadata_lift_approximate`:
  the full symbolic slice did not reach the sink, but metadata/provenance lifting still
  recovered missing request-backed operands

If neither succeeds, the candidate can still degrade to `request_binding_*` or
`candidate_only`.

## How Grading Works

Stage 4 grading is built from several orthogonal booleans.

### Request and path exactness

- `request_layout_exact`:
  the required sink operands can be laid out in the synthesized request without losing
  precision
- `request_exact`:
  currently tracks the same exactness threshold as `request_layout_exact`
- `device_path_exact`:
  there is one unique exact device path candidate
- `ioctl_case_exact`:
  the IOCTL-to-case mapping was exact
- `path_reachable_exact`:
  the sink was reached from an exact case path
- `sink_exclusive_to_case`:
  the sink is unique to that case rather than shared with siblings
- `path_shared_tail`:
  the path reaches the sink through a shared tail, so attribution is weaker
- `path_attribution_exact`:
  shorthand for exact path reachability plus sink exclusivity

### Obligation proof

- `obligation_feasible`:
  the required sink operands are at least rooted in request state
- `obligation_proven`:
  the required sink operands are concretely proven to satisfy the sink obligation

### Runtime readiness

- `dynamic_oracle_available`:
  Stage 4 found a runtime oracle rule for the sink
- `oracle_sink_specific`:
  the oracle is sink-specific rather than only family-level
- `oracle_self_test_supported`:
  the dynamic harness has a self-test for that oracle
- `dynamic_manifest_complete`:
  the request was synthesized, an oracle exists, `candidate_id` exists,
  `execution_recipe_key` exists, and at least one device path candidate exists
- `dynamic_ready`:
  `dynamic_manifest_complete` is true and `dynamic_class` is
  `automatic_confirmation`

### Dynamic class

`classify_dynamic_candidate()` maps the record into one of three classes:

- `automatic_confirmation`:
  user-mode reachable, obligation proven, request layout exact, device path exact,
  IOCTL exact, path exact, sink-exclusive, sink-specific oracle present, oracle
  self-test present, no runtime discovery required, and guard recovery status `ok`
- `guided_exploration`:
  enough structure exists to run the case, but exact automatic confirmation is not yet
  justified
- `static_triage`:
  useful as static evidence only; do not treat it as a runnable confirmation case

## Output Artifacts

Stage 4 writes the portable handoff tree under
`artifacts/static/<driver-sha256>/`.

### `verification.json`

The per-driver index for every verification record. It contains:

- corpus-level counts such as `dynamic_ready_count`
- one record per candidate flow
- relative paths to emitted artifacts
- all proof booleans and the final `proof_strength`

This is the file the dynamic stage uses to build a per-driver plan.

### `request_model.json`

The Stage 3 request-model summary copied into the handoff root. It is driver-wide, not
candidate-specific. It explains the recovered IOCTL surface and source vocabulary that
Stage 4 used to synthesize per-candidate requests.

### `requests/`

One `request_XXXX.json` per candidate. This is the static request synthesis contract.
Each file records:

- IOCTL bytes and transfer method
- input and output size contracts
- recovered guards
- sink obligation metadata
- sink argument witness details
- candidate device paths
- synthesized pre-call writes

If you want to understand why the static side believes a request should work, this is
the most detailed artifact.

### `models/`

One `model_XXXX.kstc` per candidate. This is the serialized symbolic testcase emitted by
the local verifier. It contains the IOCTL code and any concrete sink witness values that
were recovered during the local proof.

It is a compact machine-oriented artifact, not a human-facing report.

### `pocs/`

One `poc_XXXX.c` per candidate. These are generated user-mode C harnesses that:

- allocate the modeled buffers
- write synthesized little-endian witness values into them
- require an explicit device path and `--send`
- call `CreateFileA` and `DeviceIoControl`

They exist for auditability and manual reproduction, not for the automated dynamic
stage.

### `dynamic_cases/`

One `dynamic_case_XXXX.json` per candidate. This is the actual static-to-dynamic
contract. Each file contains:

- driver identity and install metadata
- entry-point and dispatch provenance
- device path candidates
- full request manifest
- target sink metadata
- proof booleans
- a `dynamic_plan` with oracle, backend, run sequence, timeout, and execution
  requirements

The dynamic stage normalizes and executes these files.

### `snapshots/`

Fallback artifacts live here:

- `fallback_XXXX.json`:
  summary of the fallback attempt, including trigger, strategy, block counts, reason,
  and lifted argument masks
- `fallback_XXXX.kssnap`:
  optional serialized XAIR symbolic state captured on the first sink hit

These are the main artifacts for debugging Stage 4 proof failures.

### `handoff.json`

The top-level static/dynamic contract. It tells the dynamic side:

- which driver binary belongs to the handoff
- which schemas to expect
- where `verification.json`, `dynamic_cases/`, `requests/`, `models/`, `pocs/`, and
  `snapshots/` live
- how many candidates, manifest-complete records, dynamic-ready records, and unique
  dynamic tests were emitted

### `installation-plan.json`

This is the static installation summary carried into Stage 5. It records:

- `declared_mode`
- `resolution_status`
- `probe_strategies`
- `package_available`
- `service_name`
- optional package metadata such as `inf_path` or dependency lists when the static side
  already knows them

For many real drivers the static side still writes a conservative unresolved plan. That
is intentional. Stage 5 either:

- repairs the plan through `probe_driver_setup()`, or
- preserves the unresolved status and keeps the case out of the automatic queue

This file is therefore an execution-planning input, not a claim that installation has
already been proven.

### `stage_reports/`

This directory is not part of the dynamic runner's main execution contract, but it is
important operationally. It contains the rewritten Stage 2-4 status files with
handoff-relative paths:

- `cfg_status.json`
- `wdm_status.json`
- `taint_status.json`
- `verification_status.json`

## What the Readiness Outcomes Mean

There are two layers of readiness: static readiness and dynamic harness support.

### Exact and ready for automatic execution

This means:

- `dynamic_ready == true`
- the case survives normalization cleanly
- `explain_case_support()` returns `automatic_supported`

These are the strongest candidates. A negative result can actually disqualify the
specific modeled request.

### Runnable but guided because oracle inputs are incomplete

This usually means:

- `dynamic_manifest_complete == true`
- the case is executable
- `dynamic_ready == false`, or the normalized support status is `guided_supported`

Common reasons:

- family-level oracle only
- missing sink-specific allocation kinds
- missing oracle context fields
- runtime device discovery required
- service dependencies incomplete

### Observability-only

The case is still executable, but only via an instrumented lane. Typical causes:

- instruction tracing required
- hardware model required
- observer lane required
- kernel-caller lane required for `IRP_MJ_INTERNAL_DEVICE_CONTROL`
- boot-start install workflow

These normalize to `observability_only`.

### Unsupported

The dynamic harness should not run the case automatically. Common reasons:

- architecture is not `x86_64`
- no sink descriptor matches
- request method unsupported
- install mode unknown or manual-only
- driver package incomplete
- invocation surface unsupported

These normalize to `unsupported`.

## Resumability

The handoff tree is hash-rooted and artifact-relative. That gives two production
properties:

- static corpus scans can stop and resume per driver
- dynamic execution can consume the same tree after a reboot or crash without having to
  rediscover the driver surface
