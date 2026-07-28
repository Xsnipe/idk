# Stage 3 - Provenance Taint and Request Modeling

## Purpose

Stage 3 answers two static questions:

1. Which request-visible values can reach the modeled sink arguments?
2. What concrete request shape should later stages use to exercise that path?

The main implementations are:

- `static/src/ks_request.c`
- `static/src/ks_taint.c`

`ks_request.c` builds the request-side vocabulary for each recovered IOCTL. `ks_taint.c`
turns modeled sink callsites into flow candidates with provenance, attribution, and
request-binding metadata.

## Request Model Construction

`ks_request_build_models()` writes `request_model.json` once per driver. Each model is
keyed by the recovered dispatch/IOCTL/case tuple and records the input surface the
static pipeline believes exists.

For every recovered IOCTL, the request model records:

- `dispatch_candidate_id`
- `ioctl_code`
- `handler_va`
- `case_entry_va`
- `helper_va`
- transfer method and access bits
- minimum input and output sizes
- request sources that the driver can read

The sources are method-aware:

- all methods get `IoControlCode`, `InputBufferLength`, and `OutputBufferLength`
- `METHOD_BUFFERED` adds `SystemBuffer`
- `METHOD_IN_DIRECT` and `METHOD_OUT_DIRECT` add `SystemBuffer`, `MdlAddress`, and `MdlSystemAddress`
- `METHOD_NEITHER` adds `Type3InputBuffer` and `UserBuffer`

This is the first place where the pipeline distinguishes trusted kernel mappings from
raw user pointers.

## Flow Candidate Recovery

`append_flow()` in `ks_taint.c` creates one `ks_flow_candidate` for each modeled sink
callsite that survives filtering. A flow candidate ties together:

- one recovered IOCTL
- one dispatch handler
- one sink callsite
- four sink argument expressions
- taint state for those expressions
- attribution mode from the dispatch-to-case analysis

Important candidate metadata comes directly from Stage 2:

- `case_reachability_exact`
- `case_reachability_shared_tail`
- helper-fallback attribution
- whether the dispatch selector was exact or heuristic

That metadata is not cosmetic. It directly controls later booleans such as
`path_reachable_exact`, `sink_exclusive_to_case`, and `path_shared_tail`.

## Candidate Classes

Before Stage 4 proves anything locally, Stage 3 already classifies each candidate with
`candidate_class_for_flow()`:

- `provisional_fingerprint`:
  the sink was kept as a low-confidence fingerprint candidate because the dispatch or
  dataflow story is incomplete.
- `modeled_sink_untainted`:
  the sink matches a modeled dangerous API, but no request-tainted operand survived to
  the sink callsite.
- `source_to_sink_exact`:
  an exact request-facing source such as `SystemBuffer`, `Type3InputBuffer`,
  `UserBuffer`, `MdlSystemAddress`, `InputBufferLength`, `OutputBufferLength`, or
  `IoControlCode` reaches a dangerous sink argument under exact case attribution.
- `source_to_sink_approximate`:
  the sink is still request-reachable, but some part of the provenance or case
  attribution lost exactness.

This class is later combined with Stage 4 proof results to decide whether a case is
`automatic_confirmation`, `guided_exploration`, or `static_triage`.

## Request Bindings and Witnesses

The important bridge between Stage 3 and Stage 4 is the request binding layer.

For each sink argument expression, Stage 3 tries to resolve a `ks_request_binding`
showing where the value came from:

- `IoControlCode`
- `InputBufferLength`
- `OutputBufferLength`
- a buffer load from a request-backed object
- a request-backed pointer

These bindings seed later `ks_arg_witness` records. A witness remembers:

- whether the argument is request-backed
- whether the mapping is controllable
- whether the mapping is exact
- whether precision was lost
- what request value corresponds to the sink value

That is why Stage 4 can still recover useful proof buckets even when the local proof is
not strong enough for `dynamic_ready`.

## Proof Language: `proof_kind` vs `proof_strength`

The repo uses two related but different labels:

- `proof_kind`:
  the result of the local structured symbolic proof in Stage 4
- `proof_strength`:
  the final grading label after local proof, fallback symbolic execution, and metadata
  lift are combined

`proof_kind` values come from `prove_flow_with_symbolic_engine()`:

- `structured_request_proof`
- `structured_request_rooted`
- `structured_request_binding`
- `ioctl_request_feasibility`

`proof_strength` is what most corpus-level summaries use. The full current set is:

- `local_exact`
- `local_approximate`
- `partial_local_exact`
- `partial_local_approximate`
- `local_rooted_exact`
- `local_rooted_approximate`
- `partial_local_rooted_exact`
- `partial_local_rooted_approximate`
- `fallback_symbolic_exact`
- `fallback_symbolic_approximate`
- `metadata_lift_exact`
- `metadata_lift_approximate`
- `request_binding_only`
- `request_binding_approximate`
- `candidate_only`

The buckets you called out are the ones the current corpus is dominated by:

- `local_exact`:
  every required sink argument is locally proven from request-controlled data, and the
  witnesses are exact with no precision loss.
- `local_approximate`:
  every required sink argument is still locally proven, but at least one witness lost
  exactness.
- `request_binding_only`:
  the pipeline can bind at least one sink argument back to request state, but it cannot
  prove the sink obligation locally or through fallback. This is still useful for guided
  dynamic execution.
- `request_binding_approximate`:
  same basic situation as `request_binding_only`, but with precision loss or a
  provisional/fallback path that reached only an approximate request-backed story.
- `candidate_only`:
  the sink candidate exists, but no request-backed witness survived strongly enough to
  claim even a request-binding proof.

The distinction between `request_binding_only` and `candidate_only` is important:

- `request_binding_only` means "we know which request field family matters"
- `candidate_only` means "we only know that this sink candidate is interesting"

## Output: `request_model.json`

`request_model.json` is the Stage 3 artifact that survives into the handoff package.
It is not yet a runnable dynamic manifest. It is the per-driver static summary of:

- recovered IOCTLs
- transfer-method-specific request sources
- minimum size hints
- exact/heuristic dispatch provenance

Stage 4 copies this file into the handoff root unchanged because the dynamic stage still
needs the request-side baseline when auditing or repairing manifests.

## Failure and Degradation Modes

Stage 3 usually fails by losing precision, not by crashing out completely.

Typical degradation paths are:

- exact case attribution becomes shared-tail attribution
- buffer loads are found, but only through approximate arithmetic or widening
- request roots are recovered, but concrete sink obligations are not proven
- a modeled sink is retained even though the final sink operand is not tainted

Those cases are still preserved. The pipeline prefers emitting a weaker bucket over
discarding a real driver path.
