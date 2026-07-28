# Stage 1 - PE Intake and Fingerprinting

## Purpose

Stage 1 turns a raw `.sys` into a stable analysis subject.

It is responsible for:

- opening the image safely
- hashing it
- validating that it is a driver-shaped PE
- collecting architecture and section metadata
- emitting a stable per-driver identity for later stages

## Inputs

- one driver binary, or a directory containing many binaries

## Main Work

The intake code lives in `static/src/ks_intake.c` with shared declarations under `static/include/ks/`.

At this stage the pipeline:

1. parses the PE headers;
2. records image layout and machine type;
3. computes the SHA-256 identity used everywhere else in the pipeline;
4. rejects obviously unsupported shapes before heavier analysis starts.

The fingerprint data is important because every later artifact is rooted in the driver hash. That gives the pipeline a stable directory layout and makes resume-safe operation simple.

## Outputs

Stage 1 produces the minimal image metadata that later stages need to:

- select the x64 path,
- build per-driver output directories,
- compare expected and actual driver hashes during dynamic execution.

## Failure Model

If intake fails, the rest of the pipeline stops for that driver. This is deliberate. Kernel Sieve does not try to recover from malformed or unsupported images by guessing its way forward.
