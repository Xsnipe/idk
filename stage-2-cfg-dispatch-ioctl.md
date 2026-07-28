# Stage 2 - CFG, Dispatch, and IOCTL Recovery

## Purpose

Stage 2 recovers the control-flow and entry surfaces that matter for user-reachable driver behavior.

For the current release target, that means x64 WDM dispatch recovery with emphasis on `IRP_MJ_DEVICE_CONTROL`.

## Main Work

The core implementation is spread across:

- `static/src/ks_cfg.c`
- `static/src/ks_wdm.c`
- `static/src/ks_x64.c`

This stage:

1. builds a CFG over relevant functions;
2. finds driver entry and dispatch registration patterns;
3. resolves major-function handlers;
4. identifies device-control front doors;
5. extracts IOCTL constants and nearby request-shape hints.

## Why This Stage Matters

Dynamic confirmation only works when the static side gives it a plausible entry path. If dispatch recovery is wrong, everything after it is wasted effort.

The stage is therefore biased toward:

- exact handler recovery when possible
- explicit fallbacks when exact recovery is not possible
- stable, reviewable artifacts instead of opaque heuristics

## Outputs

The stage feeds later work with:

- recovered handler RVAs
- front-door summaries
- device-control surfaces
- IOCTL candidates
- CFG artifacts that local verification reuses

## Precision

This is where the first large precision split appears:

- exact dispatch recovery yields the best later proof grades
- approximate or helper-based recovery still allows candidate generation, but usually degrades to guided or observability-first dynamic cases
