# Physical Memory Readiness Plan

This file tracks the current remediation plan for the `physical_memory_mapping`
family in the x64 WDM static-to-dynamic pipeline.

## Current verified issues

1. Static proof still treats physical mapping as a single-mask obligation.
   - `MmMapIoSpace` currently collapses address control and length validity into
     `required_arg_mask = 0x03`.
   - Stage 3 still treats the family as `dangerous = 0x0f`.

2. Physical execution values are not modeled correctly.
   - Exact physical cases often serialize SAT witness literals like `0x1000`
     into the request instead of a provider-backed guest physical address.
   - The generic manifest path would also use `address_of:` if a nested
     allocation were materialized, which is wrong for physical addresses.

3. Physical queue/readiness is being misread.
   - The current `dynamic_ready` formula requires sink-specific and self-tested
     oracles.
   - Physical mapping is currently an observer-backed lane, so the short-term
     target is `observer_ready`, not automatic effect-ready promotion.

4. `HalTranslateBusAddress` promotion is currently unsound.
   - The taint path can copy the translation input bus address into a later
     `MmMapIoSpace` address operand without modeling the translated output.

5. Plain mapping is not separated from downstream read/write use.
   - The current physical records mix map-only and stronger read/write effects.

## Implementation order

1. Add a physical audit report and physical-specific metrics.
   - Emit `physical-memory-funnel.json`.
   - Count address provenance, length class, path exactness, provider
     compatibility, and dynamic packaging blockers.

2. Split physical control and value obligations.
   - Add control/value/constant masks to the sink contract.
   - Record controlled/value-valid masks in verification output.
   - Require address control, but only value-valid length/cache operands.

3. Fix physical manifest/runtime materialization.
   - Materialize physical arg0 as a provider-backed `physical_test_page`.
   - Use `physical_address_of:<allocation-id>` for the address operand.
   - Use `allocation_size_of:<allocation-id>` for request-backed size only when
     the request actually carries size.
   - Stop serializing generic witness literals as execution inputs.

4. Remove exact over-credit from `HalTranslateBusAddress`.
   - Immediately stop promoting exact address provenance from the translation
     input argument.
   - Reintroduce only after output-slot provenance exists.

5. Add physical address-domain modeling.
   - Preserve exact 32-bit physical address control as a below-4-GB domain.
   - Track extension, masks, alignment, and bias.
   - Suppress cases when the provider GPA cannot fit the controlled domain.

6. Split map-only from map-plus-read/write cases.
   - Track mapped pointer provenance from `MmMapIoSpace` return values.
   - Classify `physical_map_only`, `physical_map_read`, `physical_map_write`,
     `physical_map_copy_to_user`, and `physical_map_copy_from_user`.

7. Surface physical readiness separately from generic `dynamic_ready`.
   - Add `physical_manifest_complete`.
   - Add `physical_provider_compatible`.
   - Add `physical_observer_ready`.
   - Add `physical_effect_ready`.

## Immediate acceptance goals

- No physical recipe should use `address_of:` for `MmMapIoSpace` arg0.
- No exact physical recipe should use a generic SAT witness literal as its
  runtime physical address.
- No exact physical result should be derived from the current
  `HalTranslateBusAddress` input-copy shortcut.
- The next physical rerun should be evaluated against `observer_ready`, not
  against the current sink-specific `dynamic_ready` ceiling.

## Implementation status

Completed in the current patch set:

1. Physical proof now requires address control instead of collapsing address and
   size into a single required mask.
2. Physical manifests now emit provider-backed `mapped_page` allocations and
   use `physical_address_of:<allocation-id>` for `MmMapIoSpace` arg0.
3. Exact physical requests no longer serialize generic SAT witness literals as
   runtime physical addresses.
4. The `HalTranslateBusAddress` input-promotion shortcut has been removed from
   the exact provenance path.
5. `physical-memory-funnel.json` is emitted per driver so the physical queue can
   be audited separately from the generic dynamic queue.

Verified smoke result:

- `dbutil_2_3.sys` now emits a physical dynamic case with:
  - `request.allocations[0].kind = "mapped_page"`
  - `request.allocations[0].provider = "ks_test_pci_v1"`
  - `request.pre_call_fixups[*].value = "physical_address_of:arg0_nested"`
- `ADV64DRV.sys` now reports the physical obligation as
  `physical_memory.address_control` with `required_arg_mask = 1`.

Still outside this patch set:

1. Physical address-domain fitting and below-4-GB compatibility checks.
2. Splitting plain map-only cases from map-plus-read/write or copy-backed
   effects.
3. Any change to the global `dynamic_ready` gate. Physical cases still remain
   guided when installation or device-path exactness is unresolved.
