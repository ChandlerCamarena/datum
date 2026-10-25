# Roadmap

Phase 0 work items, in order. Exit criteria for the phase are in
[DESIGN.md](DESIGN.md) section 10; this file does not change them. M0.2 to
M0.4 can overlap M0.1. Estimates assume the section 10 pace.

Items are checked off in the commits that complete them, and a milestone's
completion date is added to its heading. Dates are recorded, not predicted.

Phase 0 started 2026-10-24.

### M0.1 Vulkan foundations (30 to 50 h)

- [ ] Vulkan Tutorial through swapchain creation.
- [ ] Rendering and presentation.
- [ ] Frames in flight.
- [ ] Textures: optional, after M0.3.

Exit: every semaphore and fence in a two-frames-in-flight loop can be
explained: what it orders, who signals it, and who waits on it.

### M0.2 Read MangoHud (15 to 25 h)

- [ ] Build from source on Arch.
- [ ] Trace the path from its `vkQueuePresentKHR` hook to the frametime it
      displays and logs.
- [ ] Notes committed in `docs/notes/`.
- [ ] One candidate patch identified.

### M0.3 Layer skeleton (30 to 50 h)

`capture/vulkan-layer/` v0.

- [ ] Meson build, to match Mesa and MangoHud.
- [ ] Chain `vkCreateInstance` and `vkCreateDevice` through the loader.
- [ ] Hook `vkQueuePresentKHR`.
- [ ] Emit Perfetto events with `CLOCK_MONOTONIC` timestamps.
- [ ] Implicit layer manifest, enabled by environment variable.
- [ ] Decide D6 and record it in DESIGN.md.

Exit: a Vulkan application run with the variable set produces a trace with
one event per present in the Perfetto UI.

### M0.4 First upstream contribution (10 to 20 h)

- [ ] Implement the M0.2 candidate patch.
- [ ] Test it, recording what was tested and on what hardware.
- [ ] Submit it to MangoHud.

### M0.5 GPU timing and display time (30 to 50 h)

DESIGN.md section 6.1.

- [ ] GPU timestamp queries around each frame's submitted work.
- [ ] Calibrate to `CLOCK_MONOTONIC`, recalibrate during capture, record the
      deviation bound.
- [ ] Display time from `VK_EXT_present_timing`, falling back to
      `wp_presentation`, with the source recorded per frame.

Exit: a trace from one game shows present intervals, GPU execution time, and
labeled display time per frame.

### M0.6 Analyzer v0 (40 to 60 h)

`analysis/` v0.

- [ ] Load layer traces through `trace_processor`.
- [ ] Section 9 metrics with run-level confidence intervals.
- [ ] Unit tests for all statistics code.
- [ ] Sanitizer CI and one libFuzzer target.

Exit: given N runs per configuration, the analyzer prints the section 9
metrics with confidence intervals.

### M0.7 Read Gamescope (15 to 25 h)

- [ ] Build from source.
- [ ] Notes on its present-timing path committed in `docs/notes/`.

### M0.8 Upstream standing (30 to 60 h)

- [ ] File the remaining bug reports and submit the remaining patches, and
      follow each through review.

Exit: the "Ecosystem standing" row of DESIGN.md section 10.2 is met.

### M0.9 Writeup (30 to 50 h)

- [ ] Three games, three configurations, captured with the section 9.3
      controls and analyzed with the M0.6 analyzer.
- [ ] Published with full methodology in `docs/`.

### M0.10 Open driver and collaborator (10 to 20 h)

Phase 1 needs kernel GPU events the proprietary NVIDIA driver cannot provide.

- [ ] nouveau/NVK boot entry working on the primary machine (D4).
- [ ] Phase 1 collaborator chosen (D1).

Exit: both decisions recorded in DESIGN.md section 14.

## Later phases

Defined in DESIGN.md section 10; broken into work items here when Phase 0
completes.
