# Sink Matrix

## Purpose

This document is the sink catalog for the current x64 WDM pipeline. It answers five
questions for each sink class:

1. what the sink actually is;
2. how it can be abused;
3. how Kernel Sieve detects it statically;
4. what the current provenance buckets mean for that sink;
5. what the dynamic stage actually tries to observe.

The authoritative code paths are:

- static classification and ranking:
  `static/src/ks_taint.c`
- sink obligations and runtime oracle assignment:
  `static/src/ks_verify.c`
- dynamic sink registry and lane support:
  `dynamic/src/ks_dynamic/support.py`

## How to Read the Provenance Labels

Two details matter before reading the sink-by-sink sections.

First, the current static verifier does not give every sink a fully semantic operand
model. The strongest specialized Stage 4 obligations today are:

- `section_view_unmap`
- `copy_memory`

Most other sink kinds currently use a coarser Stage 4 obligation:

- choose the primary dangerous argument from `dangerous_arg_mask`
- require it to be request-controlled
- model the local formula as `nonzero`

That means the dynamic oracle often carries more of the semantic burden than the local
proof for:

- physical-memory mapping
- port I/O
- control-register access
- MSR access
- process-object sinks
- registry and file sinks

Second, the important `proof_strength` buckets mean this:

- `local_exact`:
  the sink's required operand set was locally proven from request-backed values with no
  precision loss
- `local_approximate`:
  same basic proof, but some witness widened or lost exactness
- `fallback_symbolic_exact`:
  the local proof was incomplete, but bounded handler-to-sink symbolic execution reached
  the sink and recovered exact request-backed control
- `fallback_symbolic_approximate`:
  same as above, but with approximation
- `metadata_lift_exact`:
  the sink was not directly closed locally, but provenance metadata still recovered exact
  request-backed control of the missing sink operand
- `metadata_lift_approximate`:
  same with approximation
- `request_binding_only`:
  Kernel Sieve can name the request field family that feeds the sink, but it cannot yet
  prove the sink obligation
- `request_binding_approximate`:
  same, but some part of the request-to-sink mapping is widened or indirect
- `candidate_only`:
  the sink callsite is interesting, but no request-backed operand survived strongly
  enough to claim binding

For sink-specific reading:

- on strong sinks, `local_exact` is meaningful evidence
- on coarse sinks, `local_exact` usually means "the primary dangerous operand is exactly
  request-backed," not "the full API semantics are modeled"
- `request_binding_only` means "this request shape still deserves dynamic execution"
- `candidate_only` means "static triage only"

## Dynamic Behavior Modes

The dynamic stage does not test every sink the same way. There are four current
evidence modes:

| Evidence mode | What the runner proves | Typical sinks |
| --- | --- | --- |
| exact effect oracle | a sink-specific canary side effect happened in the guest | `section_view_unmap`, `section_view_map`, exact process sinks, exact registry sinks, exact file sinks, `copy_memory` |
| family canary oracle | the candidate reached a real object family target, but the exact API-specific effect is still generalized | generic `process_object_access`, `registry_open`, `object_registry_access` |
| host-side observability oracle | instrumentation or an observer device saw the privileged behavior, but the effect is not normalized as a stable guest artifact | `physical_memory_mapping`, `port_io`, `control_register_access`, `msr_access` |
| no automatic oracle | the case is kept for guided/manual execution because no built-in success oracle exists yet | `file_open`, `sanitizer_or_buffer_map`, `irp_completion` |

That distinction matters when reading provenance:

- `local_exact` plus an exact effect oracle is close to production-grade evidence
- `local_exact` plus an observability-only oracle means "controlled privileged behavior
  was observed," not yet "full end impact was semantically proven"
- `request_binding_only` plus a family oracle is still useful because the dynamic stage
  can often decide whether the candidate deserves promotion or manual review

## Summary Matrix

| Static sink kind | What it is and how it is abused | Static detection and Stage 4 proof model | What strong provenance means here | Dynamic behavior under test | Current execution class |
| --- | --- | --- | --- | --- | --- |
| `section_view_unmap` | unmap an attacker-chosen section view in the current or target process | exact API match, `+85`, `section_view.base_address_control` on `arg1`, `nonzero` | `local_exact` means the unmap base address is exactly request-backed | `mm_section_unmap` -> `section_view_unmapped` | automatic |
| `section_view_map` | map an attacker-chosen section into privileged memory | exact API match, `+85`, all args dangerous, current proof reduced to primary dangerous map operand | `local_exact` means exact request control of the primary map operand, not full map semantics | `mm_section_map` -> `section_view_mapped` | automatic or guided |
| `physical_memory_mapping` | map chosen physical memory or MMIO into kernel space | `MmMapIoSpace` family, `+90`, all args dangerous, coarse primary-operand `nonzero` proof | `local_exact` means the primary mapping operand is exactly request-backed | `mm_map_observer` -> `mapped_test_page_observed` | observability-only |
| `port_io` | read or write privileged I/O ports | privileged-register family, `+90`, all args dangerous, coarse primary-operand proof | `local_exact` means the request exactly controls the key port operand | `io_port_trace` -> `port_io_observed` | observability-only |
| `control_register_access` | write CPU control registers such as CR0/CR4 | privileged-register family, `+105`, all args dangerous, coarse primary-operand proof | `local_exact` means the request exactly controls the key CR operand | `control_register_trace` -> `control_register_write_observed` | observability-only |
| `msr_access` | read or write model-specific registers | privileged-register family, `+105`, all args dangerous, coarse primary-operand proof | `local_exact` means the request exactly controls the key MSR operand | `msr_trace` -> `msr_access_observed` | observability-only |
| `process_handle_open` | open a handle to a chosen process | exact promotion from `process_object`, `+80`, coarse `process_object.target_control` proof | `local_exact` means the target-process operand is exactly request-backed | `process_open` -> `canary_process_handle_opened` | automatic |
| `process_lookup` | resolve a chosen PID to a process object | exact promotion from `process_object`, `+80`, coarse `process_object.target_control` proof | `local_exact` means the target PID/object operand is exactly request-backed | `process_lookup` -> `canary_process_object_referenced` | automatic |
| `process_handle_reference` | convert a chosen handle into a kernel process reference | exact promotion from `process_object`, `+80`, coarse `process_object.target_control` proof | `local_exact` means the handle/object operand is exactly request-backed | `process_handle_reference` -> `canary_process_handle_referenced` | automatic |
| `process_object_access` | generic process-object abuse, including terminate-like paths | `process_object` family, `+80`, exact terminate promoted, otherwise coarse primary-operand proof | `local_exact` means the primary process target is exactly request-backed | exact terminate: `process_termination`; generic: `process_handle_canary` | automatic for terminate, guided for generic |
| `registry_create` | create privileged registry keys | exact promotion from `registry_operation`, `+55`, coarse `object_registry.target_control` proof | `local_exact` means the primary registry target is exactly request-backed | `registry_create` -> `canary_registry_key_created` | automatic |
| `registry_set_value` | write privileged registry values | exact promotion from `registry_operation`, `+55`, coarse `object_registry.target_control` proof | `local_exact` means the key/value target is exactly request-backed | `registry_set_value` -> `canary_registry_value_written` | automatic |
| `registry_delete` | delete privileged registry keys | exact promotion from `registry_operation`, `+55`, coarse `object_registry.target_control` proof | `local_exact` means the delete target is exactly request-backed | `registry_delete` -> `canary_registry_key_deleted` | automatic |
| `registry_open` | open or touch a sensitive registry/object-manager target | non-exact registry/object path, coarse `object_registry.target_control` proof | `local_exact` means the primary object target is exactly request-backed, not that the final effect is fully modeled | `object_registry_canary` -> `canary_object_touched` | guided |
| `file_create` | create a file in a privileged location | exact promotion from `file_operation`, `+55`, coarse `file.target_control` proof | `local_exact` means the primary file target is exactly request-backed | `file_create` -> `canary_file_created` | automatic |
| `file_write` | overwrite file contents from kernel context | exact promotion from `file_operation`, `+55`, coarse `file.target_control` proof | `local_exact` means the file target or write operand is exactly request-backed | `file_write` -> `canary_file_contents_written` | automatic |
| `file_delete` | delete a file from kernel context | exact promotion from `file_operation`, `+55`, coarse `file.target_control` proof | `local_exact` means the delete target is exactly request-backed | `file_delete` -> `canary_file_deleted` | automatic |
| `file_open` | open-like file primitive without a stronger create/write/delete effect | non-exact file path, coarse `file.target_control` proof | `local_exact` means the open target is exactly request-backed, but no automatic effect oracle exists | no dedicated oracle today | often unsupported |
| `object_registry_access` | generic object-manager or registry touching primitive | family-level object/registry bucket, coarse target-control proof | `local_exact` means the primary object/registry target is exactly request-backed | `object_registry_canary` -> `canary_object_touched` | guided |
| `copy_memory` | controlled kernel copy, overwrite, or overflow primitive | specialized copy analysis, exact dst/src/len taint scoring, proof via `size_overflow`, `size_control`, `dst_control`, or `src_control` | `local_exact` means the selected copy obligation was exactly proven; on `size_overflow` it means an actual overflow witness exists | `buffer_canary_diff` -> `canary_buffer_corrupted` | automatic |
| `sanitizer_or_buffer_map` | probe, pin, or map a user buffer in a way that can enable later misuse | low-score sanitizer family, `+10`, reachability-oriented today | strong buckets are rare; even `local_exact` is weaker here because no rich sink semantic is modeled | no sink-specific oracle today | guided |
| `irp_completion` | complete an IRP with attacker-influenced state or timing | completion family, `+5`, reachability-oriented today | strong buckets are rare; provenance mostly means "this completion path is worth deeper review" | no sink-specific oracle today | guided |

## Memory Mapping

### `section_view_unmap`

What it is:

- a driver-controlled call into `ZwUnmapViewOfSection` or `NtUnmapViewOfSection`

How it can be abused:

- unmap a chosen section view in the current or target process
- invalidate mapped payloads or code/data ranges
- tear down memory a privileged component assumed would remain mapped

How static detection works:

- Stage 3 matches the exact sink name
- `classify_flow_candidate()` sets `sink_kind = section_view_unmap`
- the sink gets a high static score bias of `+85`
- the dangerous operand is hard-coded to `arg1`

What exact provenance means here:

- `local_exact` means the Stage 4 verifier proved that the base-address argument
  (`arg1`) is exactly request-backed and satisfies the `nonzero` obligation
- `fallback_symbolic_exact` means the local proof needed the bounded handler-to-sink
  slice to recover that same base-address control

What weaker provenance means here:

- `request_binding_only` means Stage 3/4 knows which request field feeds the base
  address, but it did not fully prove the local obligation
- `candidate_only` means an unmap candidate exists on the recovered path, but the base
  address is not yet tied to request state

In practice:

- `request_binding_only` is still a valid dynamic lead because the unmap target field is
  usually the key exploit variable
- `candidate_only` should be read as "unmap-shaped path exists, but request control of
  the base address is still missing"

What the dynamic stage tests:

- oracle: `mm_section_unmap`
- behavior: `section_view_unmapped`

### `section_view_map`

What it is:

- a driver-controlled call into `ZwMapViewOfSection` or `NtMapViewOfSection`

How it can be abused:

- map a chosen section into a privileged address space
- expose attacker-chosen content inside a trusted process
- turn a section handle primitive into code/data placement

How static detection works:

- Stage 3 matches the exact sink name
- `classify_flow_candidate()` marks all four arguments dangerous and biases the score by
  `+85`

Important modeling note:

- the current Stage 4 model does not encode the full `MapViewOfSection` contract
- it reduces the local proof to control of the primary dangerous operand with a
  `nonzero` obligation, then relies on the dynamic oracle to check that a canary section
  was actually mapped

What exact provenance means here:

- `local_exact` means the primary dangerous map operand is exactly request-backed
- it does not yet mean every map parameter is semantically modeled

What weaker provenance means here:

- `request_binding_only` means the request appears to steer an important map operand,
  but the current verifier did not close the full local proof
- `candidate_only` means the map path is present and interesting, but request-backed
  control of the important operand was not recovered strongly enough

What the dynamic stage tests:

- oracle: `mm_section_map`
- behavior: `section_view_mapped`
- exact dynamic support still depends on required runtime allocations such as
  `section_handle` and `mapping_slot`

### `physical_memory_mapping`

What it is:

- a driver-controlled physical-memory or MMIO mapping primitive, currently centered on
  `MmMapIoSpace`

How it can be abused:

- map attacker-chosen physical memory into kernel space
- access device MMIO regions directly
- turn a kernel bug into raw machine-state manipulation

How static detection works:

- Stage 3 assigns `sink_kind = physical_memory_mapping`
- all four arguments are treated as dangerous
- the candidate gets a `+90` score bias

What exact provenance means here:

- `local_exact` means the primary dangerous physical-mapping operand is exactly
  request-backed
- because the local obligation is coarse, the dynamic oracle is what upgrades this from
  "address control exists" to "a controlled physical mapping happened"

What weaker provenance means here:

- `request_binding_only` means the request still appears to steer the physical address,
  length, or mapping operand, which is enough to justify observer execution
- `candidate_only` means the mapping API was recovered but request control is still too
  weak to claim a meaningful primitive

What the dynamic stage tests:

- Stage 4 oracle: `mm_map_observer`
- success condition: `mapped_test_page_observed`
- Stage 5 support class: `observability_only`
- selected lane: `tcg_test_device`
- required extra capability: hardware model `ks_test_pci_v1`

## Privileged Register and Port I/O

### `port_io`

What it is:

- direct I/O port reads or writes via `READ_PORT_*` / `WRITE_PORT_*`

How it can be abused:

- touch chipset or device ports directly
- reprogram hardware state from a user-reachable path
- interact with privileged buses or firmware-facing interfaces

How static detection works:

- Stage 3 puts the callsite in `privileged_register_io`
- port-like operations get `sink_kind = port_io`
- all four arguments are marked dangerous
- score bias is typically `+90`

What provenance means here:

- `local_exact` means the primary dangerous port operand is exactly request-backed
- `request_binding_only` means the request appears to steer the port number or port
  operand, but the local proof is not complete
- `candidate_only` means the port I/O callsite is interesting, but the request-to-port
  binding is not yet strong enough to claim user-steered hardware access

What the dynamic stage tests:

- oracle: `io_port_trace`
- behavior: `port_io_observed`
- execution class: `observability_only`
- lane: `tcg_instrumented`

### `control_register_access`

What it is:

- a modeled control-register access path

How it can be abused:

- modify CR0/CR4-style machine controls
- disable or weaken write-protect / SMEP / SMAP-like protections
- redirect execution assumptions at the CPU level

How static detection works:

- the sink stays under `privileged_register_io`
- `sink_kind = control_register_access`
- all four arguments are marked dangerous
- score bias is `+105`, making these `critical` candidates very quickly

What provenance means here:

- `local_exact` means the primary dangerous control-register operand is exactly
  request-backed
- this is still a coarse local proof; the dynamic trace supplies the stronger evidence
- `request_binding_only` means the request likely steers the CR selector or value, but
  the verifier did not prove the full local obligation
- `candidate_only` means the CR access exists on a reachable path, but request control
  of the critical operand is still unresolved

What the dynamic stage tests:

- oracle: `control_register_trace`
- behavior: `control_register_write_observed`
- execution class: `observability_only`
- lane: `tcg_instrumented`

### `msr_access`

What it is:

- a `WRMSR` or `RDMSR` style path

How it can be abused:

- rewrite CPU model-specific registers
- alter low-level kernel, virtualization, or hardware behavior

How static detection works:

- `sink_kind = msr_access`
- all four arguments are marked dangerous
- score bias is `+105`

What provenance means here:

- `local_exact` means the primary MSR operand is exactly request-backed
- weaker buckets still mean the request likely steers the MSR number or value, but the
  proof is not complete
- `candidate_only` means the MSR path is interesting, but the request-backed operand is
  still too weakly recovered for a control claim

What the dynamic stage tests:

- oracle: `msr_trace`
- behavior: `msr_access_observed`
- execution class: `observability_only`
- lane: `tcg_instrumented`

## Process Object and Lifecycle

### Exact process sinks

These are the process sinks with sink-specific dynamic oracles:

- `process_handle_open`
- `process_lookup`
- `process_handle_reference`
- `process_object_access` when the exact sink is `ZwTerminateProcess` / `NtTerminateProcess`

What they are:

- privileged process-handle opens
- process-object lookups by PID
- handle-to-object conversions
- process termination

How they can be abused:

- target protected or privileged processes from user space
- gain handles or object references the caller should not have
- terminate or later tamper with sensitive processes

How static detection works:

- Stage 3 starts with the broader `process_object` family
- API name then selects:
  - `process_handle_open`
  - `process_lookup`
  - `process_handle_reference`
  - or generic `process_object_access`
- all four args are treated as dangerous
- score bias is `+80`

What provenance means here:

- `local_exact` means the primary process-target operand is exactly request-backed
- because the local obligation is still coarse, the dynamic oracle is what confirms
  whether a canary process was really opened, referenced, or terminated
- `request_binding_only` means the request still appears to steer the target PID,
  handle, or object reference, which is enough to keep the case runnable
- `candidate_only` means a process-sensitive path exists, but the target operand is not
  yet tied tightly enough to request state

What the dynamic stage tests:

- `process_open` -> `canary_process_handle_opened`
- `process_lookup` -> `canary_process_object_referenced`
- `process_handle_reference` -> `canary_process_handle_referenced`
- `process_termination` -> `canary_process_exited`

### Generic `process_object_access`

What it is:

- the generic process-object bucket when the sink is clearly process-related but not one
  of the exact specialized cases above

How it can be abused:

- target arbitrary processes without a sink-specific semantic claim yet

How static detection works:

- same `process_object` family recovery as above
- sink-specific API mapping was not strong enough to promote the case to an exact
  specialized process sink

What provenance means here:

- `local_exact` still only means the primary process-target operand is exactly
  request-backed
- this is why generic process-object cases are usually `guided_supported`, not automatic
- `request_binding_only` means "the request likely chooses the process target, but we do
  not yet have an exact API-specific effect claim"
- `candidate_only` means "process-object path recovered, but targeting control is still
  too weak for a meaningful dynamic contract"

What the dynamic stage tests:

- family oracle: `process_handle_canary`
- behavior: `canary_process_targeted`

## Registry and Generic Object Access

### Exact registry sinks

Exact registry sinks currently promoted to sink-specific dynamic oracles are:

- `registry_create`
- `registry_set_value`
- `registry_delete`

What they are:

- create, modify, or delete operations on kernel-visible registry objects

How they can be abused:

- persistence
- service or driver configuration tampering
- policy or ACL manipulation
- sabotage by deleting keys or values

How static detection works:

- Stage 3 starts from the `registry_operation` family
- exact API names promote the sink to `registry_create`, `registry_set_value`, or
  `registry_delete`
- all four args are treated as dangerous
- score bias is `+55`

What provenance means here:

- `local_exact` means the primary registry-target operand is exactly request-backed
- it does not mean the full `OBJECT_ATTRIBUTES` or `UNICODE_STRING` semantics are
  statically modeled end to end
- `request_binding_only` means the request still appears to choose the registry key,
  value name, or payload, but the verifier did not fully prove the local obligation
- `candidate_only` means the registry path exists, but the exact request-to-target
  binding is not yet strong enough for an automatic claim

What the dynamic stage tests:

- `registry_create` -> `canary_registry_key_created`
- `registry_set_value` -> `canary_registry_value_written`
- `registry_delete` -> `canary_registry_key_deleted`

The exact registry oracles also need context fields such as:

- `native_path`
- `win32_path`
- `value_name`
- `expected_bytes`

Missing context is what downgrades an otherwise strong case to `guided_supported`.

### `registry_open` and `object_registry_access`

What they are:

- broader open/touch operations over keys, sections, or related object-manager targets

How they can be abused:

- open privileged object-manager targets as a step toward later abuse
- gain a reachable object handle or section primitive without yet proving the exact
  follow-on effect

How static detection works:

- `registry_open` comes from the `registry_operation` family when the API is not one of
  create/set/delete
- `object_registry_access` is a broader static bucket for object/registry sinks

What provenance means here:

- `local_exact` means the primary object or registry target operand is exactly
  request-backed
- because these are family-level cases, `request_binding_only` is common and still
  meaningful
- `candidate_only` means the object-touch path is real, but the target provenance is
  still too weak to treat it as a runnable automatic case

What the dynamic stage tests:

- family oracle: `object_registry_canary`
- behavior: `canary_object_touched`
- execution class: usually `guided_supported`

## File Operations

### Exact file sinks

Exact file sinks currently promoted to sink-specific dynamic oracles are:

- `file_create`
- `file_write`
- `file_delete`

What they are:

- file creation, content overwrite, or deletion from a kernel path

How they can be abused:

- drop or replace files in privileged locations
- overwrite configs, scripts, or binaries
- delete files used for security or service operation

How static detection works:

- Stage 3 starts from the `file_operation` family
- API name then selects `file_create`, `file_write`, `file_delete`, or `file_open`
- all four args are treated as dangerous
- score bias is `+55`

What provenance means here:

- `local_exact` means the primary file-target operand is exactly request-backed
- the local proof is still coarse compared with a full `CreateFile` semantic model, so
  the dynamic oracle is what proves the canary file effect
- `request_binding_only` means the request still appears to choose the target path or
  written bytes, which is enough to keep the case in the exact-file family
- `candidate_only` means the file path exists, but target control is not yet proven
  strongly enough for an exact handoff

What the dynamic stage tests:

- `file_create` -> `canary_file_created`
- `file_write` -> `canary_file_contents_written`
- `file_delete` -> `canary_file_deleted`

The exact file oracles need path context such as:

- `native_path`
- `dos_path`
- `expected_bytes` for write cases

### `file_open`

What it is:

- an open-like file sink that was not promoted to create/write/delete

How it can be abused:

- often as a step toward later file-object use rather than as a complete final effect

Current state:

- Stage 3 can recover `file_open`
- there is no dedicated dynamic sink descriptor for it today
- these cases are often static-only or require manual promotion to a richer file oracle

What provenance means here:

- `local_exact` means the open target operand is exactly request-backed
- `request_binding_only` means the request still appears to choose the opened path or
  handle target, but there is no exact file-effect oracle yet
- `candidate_only` means the open path is only a static lead

## Controlled Copy

### `copy_memory`

What it is:

- copy-style sinks such as `memcpy`, `memmove`, `RtlCopyMemory`, `RtlMoveMemory`,
  `MmCopyMemory`, and `MmCopyVirtualMemory`

How it can be abused:

- arbitrary or semi-arbitrary kernel write
- attacker-directed source read
- overflow by over-controlling the copy length
- corruption of guarded kernel buffers

How static detection works:

- Stage 3 puts the sink in `controlled_copy`
- `copy_memory` gets a specialized dangerous-arg analysis:
  - `arg0` if the destination looks request-tainted
  - `arg1` if the source looks request-tainted
  - `arg2` if the length looks request-tainted
- score is assembled from those exact conditions, so copy cases can land anywhere from
  medium to critical depending on which operands are controlled

This is the most specialized non-section local obligation in the current repo.

Stage 4 chooses among:

- `copy_memory.size_overflow`
- `copy_memory.size_control`
- `copy_memory.dst_control`
- `copy_memory.src_control`

What exact provenance means here:

- `local_exact` means the chosen copy obligation was proven exactly
- for `size_overflow`, the verifier recovered a destination capacity and proved that the
  controlled length can exceed it
- for the other cases, the verifier proved exact control over the selected destination,
  source, or length operand

What weaker provenance means here:

- `request_binding_only` means the request feeds a copy operand, but the verifier could
  not prove the overflow or exact corruption story yet
- `candidate_only` means a copy primitive exists on the path, but request-backed control
  of the key operand was not recovered strongly enough

This sink is different from most other families:

- `local_exact` on `copy_memory` is one of the strongest static results in the current
  pipeline
- `request_binding_only` is still valuable because copy primitives often become exact
  once dynamic setup supplies cleaner concrete sizes or buffers

What the dynamic stage tests:

- oracle: `buffer_canary_diff`
- behavior: `canary_buffer_corrupted`

## Buffer Probe or Map

### `sanitizer_or_buffer_map`

What it is:

- sanitizer or user-buffer mapping operations such as `ProbeForRead`,
  `ProbeForWrite`, `MmGetSystemAddressForMdlSafe`, `MmProbeAndLockPages`, and
  `MmMapLockedPagesSpecifyCache`

How it can be abused:

- usually not as the final vulnerability by itself
- more often as the enabling step for unsafe user-buffer lifetime, remapping, or stale
  trust in a user-controlled pointer

How static detection works:

- Stage 3 identifies the `buffer_probe_or_map` family
- the sink is marked as a sanitizer-style path, not as a direct dangerous-arg sink
- score bias is only `+10`

Important provenance note:

- these cases currently have no strong Stage 4 dangerous-arg obligation
- the verifier mostly treats them as reachability-oriented candidates
- `local_exact` is therefore uncommon and less meaningful here than on copy or unmap
- `request_binding_only` usually means "the request can reach the buffer-mapping step"
- `candidate_only` usually means "this is an enabling path, not a finished exploit
  claim"

What the dynamic stage tests:

- there is currently no sink-specific automatic oracle
- execution class is `guided_supported`

## IRP Completion

### `irp_completion`

What it is:

- `IoCompleteRequest` / `IofCompleteRequest` style completion paths

How it can be abused:

- complete a request with corrupted state
- trigger lifetime or race bugs around IRP ownership
- produce follow-on bugs rather than a single direct privileged side effect

How static detection works:

- Stage 3 identifies the `completion` family
- score bias is only `+5`
- there is no strong dangerous-arg model yet

Important provenance note:

- these are currently reachability-heavy triage sinks
- a `candidate_only` or `request_binding_only` result is still useful, but it should be
  read as "follow this completion path more closely," not "the vulnerability is already
  proven"
- even a rare `local_exact` here should be read conservatively because the sink model is
  still much weaker than for copy or exact-effect oracles

What the dynamic stage tests:

- there is currently no sink-specific automatic oracle
- execution class is `guided_supported`

## Practical Reading Rules

If you need a short operational rule set:

1. trust `local_exact` the most on:
   `section_view_unmap` and `copy_memory`
2. trust dynamic behavior more than local semantics on:
   physical memory, port I/O, CR/MSR, process, registry, and file sinks
3. treat `request_binding_only` as a real lead for:
   copy, process, registry, file, and object sinks
4. treat `candidate_only` on:
   sanitizer and completion sinks as "interesting path, not yet a proof"
5. expect:
   `observability_only` for physical memory and privileged register sinks,
   `guided_supported` for generic family sinks,
   and `automatic_supported` mostly for exact section, process, registry, file, and copy cases
