# Project Datum: Design

| Field     | Value                                      |
|-----------|--------------------------------------------|
| Author    | Chandler Camarena                          |
| Version   | 0.1                                        |
| Date      | 2026-10-24                                 |
| Status    | Draft. Architecture fixed, details open.   |

Items marked **Decided** are fixed for Phase 0 and Phase 1. Items marked
**Proposed** are the current intent and may change. Items marked **Open**
have no decision yet; they are collected in section 14.

---

## 1. Problem statement

The Linux gaming stack is a chain of independently developed layers:

```
game binary
  -> Wine / Proton runtime
  -> DXVK (D3D8 to D3D11) or VKD3D-Proton (D3D12), or native Vulkan / OpenGL
  -> Vulkan loader and layers
  -> userspace driver (Mesa: RADV, ANV, NVK; or NVIDIA proprietary)
  -> kernel GPU driver and DRM/KMS (amdgpu, i915/xe, nouveau, nvidia)
  -> GPU scheduler and hardware queues
  -> Wayland compositor (Gamescope, KWin, Mutter, Hyprland, wlroots based)
  -> KMS atomic commit, scanout, display
```

Most layers now emit traces of their own. None of those traces is joined
to the others per frame.

* Perfetto's ftrace data source records CPU scheduling, frequency, and idle
  state, and also GPU frequency, memory totals, and DRM scheduler events
  from the kernel [4].
* Mesa emits Perfetto GPU render stages and counters on Intel,
  Freedreno/Turnip, Panfrost/PanVK, and V3D/V3DV. The Mesa documentation
  lists no RADV or radeonsi support [2].
* A Mesa merge request adds Perfetto instrumentation to the Wayland WSI:
  acquire-to-present flows and a submission-to-presentation latency
  counter. Its merge status is unverified [3].
* `VK_EXT_present_timing`, merged for Mesa 26.1 (January 2026) for RADV,
  ANV, NVK, PanVK, and Turnip on X11 and Wayland, gives swapchain users
  presentation timing from the presentation engine [1].
* Wayland `wp_presentation` feedback gives display times to the client, but
  is rarely collected in performance work.
* MangoHud measures intervals between `vkQueuePresentKHR` calls. It
  reports that a frame was slow, not why, and not when it reached the
  display.
* DXVK and VKD3D-Proton have their own HUDs and logs, which see shader
  compilation but not what happened below them.

The events exist. What does not exist:

* **Frame identity.** Nothing assigns these events to application frames
  across layers.
* **Attribution.** Nothing says which layer made a given frame late, or by
  how much.
* **Statistics.** No shared methodology turns frametimes into results with
  stated uncertainty.
* **Coverage.** AMD GPUs have no Mesa render stage source.

When a game stutters, the cause may be in any layer or in the interaction
between layers: shader compilation, a CPU thread preempted at the wrong
moment, GPU work that overran its frame, explicit-sync waits between client
and compositor, the compositor missing a vblank deadline, VRR behavior,
thermal or power state changes, or memory pressure. Today, separating these
requires an expert who collects traces from several sources and correlates
them by hand. Most user reports never get that treatment, and the community
substitutes folklore for diagnosis.

Three recurring failures follow from this:

1. **Diagnosis.** Nobody can say why a specific frame was late.
2. **Reproducibility.** Two users on similar systems report contradictory
   results because shader cache state, kernel, driver, compositor, and
   governor differ silently, and no shared methodology exists.
3. **Configuration.** Distributions ship gaming defaults chosen by consensus
   rather than measurement, and users cannot tell what changed when an
   upgrade regresses.

All three share one root cause: no tool joins what each layer did into one
record per frame. Datum builds that record first and builds
reproducibility and configuration on top of it.

**Thesis.** Datum turns existing per-layer traces into per-frame causal
reports with honest statistics, and adds capture only where no source
exists.

## 2. Goals and non-goals

### 2.1 Goals

* **G1.** Collect events from every observable layer onto a single clock,
  from existing sources first and Datum capture where none exists, with
  enough identity information to assign each event to an application
  frame.
* **G2.** For each late frame, produce a ranked list of contributing factors
  with the evidence for each.
* **G3.** Report performance with statistically honest methodology:
  percentile metrics with confidence intervals, controlled warm-up, and
  explicit run counts.
* **G4.** Record traces in the format upstream developers already use
  (Perfetto) so Datum output can be attached to bug reports.
* **G5.** Record a hardware and software fingerprint with every result, so
  results are comparable and their staleness is known.
* **G6.** Eventually, apply configuration choices that are justified by
  measured data, with rollback and bisection.

### 2.2 Non-goals

* **Reverse engineering the NVIDIA proprietary driver.** NVIDIA systems are
  measured only through interfaces visible outside the driver: Vulkan,
  Wayland, KMS events, CPU scheduling. Datum states this limitation rather
  than working around it.
* **Anti-cheat compatibility.** A policy problem, not a measurement problem.
* **Windows support.**
* **A consumer GUI** in Phases 0 through 4.
* **Game library management or launching.** Steam, Lutris, and Heroic exist.
* **DRM circumvention or patching of commercial game binaries.** Datum
  observes games through public APIs and kernel interfaces only.
* **Always-on telemetry.** Capture runs only when a user starts it. Nothing
  is uploaded without explicit opt-in.
* **X11 desktop sessions and X11 compositors.** X11 clients under XWayland
  are in scope (section 6.2).

## 3. Audience

In order of priority:

1. **Graphics stack and compositor developers**: Mesa, the kernel DRM
   drivers, wlroots, Hyprland, KWin, Mutter, Gamescope, DXVK, VKD3D-Proton.
   They debug cross-layer problems today with ad hoc tooling. They are also
   the people who can validate that Datum's measurements mean what Datum
   claims.
2. **Gaming distribution maintainers**: Bazzite, CachyOS, Nobara,
   ChimeraOS, SteamOS, and gaming spins of general distributions. They need
   evidence for default configuration choices.
3. **Kernel graphics and scheduler maintainers.** They lack real-workload
   regression detection for gaming.
4. **Researchers and benchmark publishers**, who need sound methodology.

End users benefit indirectly, first through better upstream fixes and
later through Pillar III.

Design rule: **the author must want to run the Phase 1 tool on his own
machine.** A tool its own author does not use is not maintained.

## 4. Related work

* **Perfetto.** System-wide tracing: a tracing service, an ftrace collector
  (`traced_probes`), a C++ SDK for userspace producers, the
  `trace_processor` query engine (SQL plus C++), and a browser UI. Datum
  consumes it: Perfetto is Datum's trace format, Datum's capture
  components are Perfetto producers, and Datum's analysis is built on
  `trace_processor`.
* **Perfetto FrameTimeline (Android).** The closest prior art. On Android,
  the app and SurfaceFlinger emit an expected and an actual timeline for
  every frame, keyed by a shared frame token, and `trace_processor`
  classifies each janky frame by which side missed its deadline [6]. This
  works because Android controls the whole stack, from Choreographer to
  SurfaceFlinger to the display hardware. Datum does the equivalent across
  the uncoordinated Linux stack, where no component passes a frame token
  to the next and frame identity has to be reconstructed (section 8.2).
* **Mesa's Perfetto integration and `VK_EXT_present_timing`** (section 1).
  Datum ingests both where they exist. RADV render stages are a candidate
  upstream contribution.
* **MangoHud.** Datum records the same present-to-present interval, labels
  it as such, and adds display time and per-layer causes. MangoHud is also
  an upstream contribution target.
* **GPUVis.** A viewer for ftrace GPU and vblank events. It shows kernel
  activity on a timeline but does not assign it to application frames.
  Datum differs by joining kernel events to frames and attributing
  lateness.
* **P2949/stutter.** A Rust tool with Aya eBPF probes that measures
  scheduler runnable latency, aligns it with MangoHud frametimes, and runs
  counterbalanced A/B tuning experiments [5]. Its analysis is
  correlational and covers the CPU only: it does not trace the Vulkan
  layer, KMS, or the compositor. Datum differs in scope (all layers,
  per-frame attribution) and shares its concern with controlled A/B
  methodology.

## 5. Architecture: three pillars

```
+-------------------------------------------------------------+
|  Pillar III: Declarative Configuration        (Phase 5+)    |
|  spec -> measured profile -> apply -> verify -> rollback    |
+-------------------------------------------------------------+
|  Pillar II: Reproducibility                   (Phase 3+)    |
|  workload suite, fingerprints, methodology, results DB      |
+-------------------------------------------------------------+
|  Pillar I: Observability                      (Phase 0+)    |
|  trace sources -> clock alignment -> frame assignment       |
|  -> statistics -> causal engine -> reports and annotations  |
+-------------------------------------------------------------+
```

Each pillar depends only on the pillars below it. Each is useful alone:
Pillar I is a debugging tool, Pillar II is a benchmark methodology and
dataset, Pillar III is a configuration manager. Building Pillar III before
Pillar I would produce one more opinionated configuration with no evidence
behind it, so the order is fixed.

### 5.1 Pillar I: Observability

Trace sources write timestamped events into one Perfetto trace: existing
Perfetto data sources first, Datum capture components where no source
exists. The analysis engine aligns the events on one clock, assigns them
to frames, computes per-frame timing segments, and explains lateness.
Section 6 lists the sources. Section 8 describes the data flow.

GPU hang and reset diagnostics are part of Pillar I. A GPU hang is the
extreme case of a late frame, and the evidence needed is the same kind:
which job was running, what was in flight, what the kernel scheduler and
driver did, and what the application saw.

### 5.2 Pillar II: Reproducibility

* A curated workload suite, starting with games that ship built-in
  benchmarks, then scripted scenes for games without them.
* A hardware and software fingerprint schema (section 8.5).
* Warm-up and shader cache control: results are labeled cold-cache or
  warm-cache and never mixed.
* The statistical methodology in section 9.
* A public, queryable results database keyed by fingerprint and stack
  version tuple. The database is the deliverable; the harness exists to
  fill it.

### 5.3 Pillar III: Declarative Configuration

A specification of intent (hardware constraints, game class, priority among
smoothness, peak throughput, battery life, and compatibility, plus risk
tolerance) is resolved against Pillar II data into concrete settings:
kernel parameters, scheduler, CPU governor, driver and Proton version pins,
compositor settings, per-game launch options, sysctls. Every applied
configuration is versioned. A measured regression can be bisected against
configuration changes and rolled back.

Backends, in order: a NixOS module (Nix already models system state
declaratively and supports rollback), then an overlay approach for
image-based distributions such as Bazzite, then an audit-only mode for
traditional distributions.

## 6. Stack layers and observation points

Datum observes from outside each component wherever possible. External
observation needs no upstream changes, works on stock distributions, and
keeps Datum's code under Datum's license. Upstream changes are used only
where external observation cannot get the data, and are submitted to that
project under its license.

The table is a gap audit. For each need it lists the source that already
exists, if any, and what Datum adds. Datum writes new capture code only in
the rows where no source exists.

| Need | Existing source | Datum's job | Phase |
|---|---|---|---|
| CPU scheduling, frequency, idle | Perfetto ftrace (`sched`, `power`) | configure and ingest | 1 |
| DRM vblank and page flips, `gpu_scheduler`, `dma_fence` | Perfetto ftrace | configure, ingest, link to frames | 1 |
| GPU render stages | Mesa Perfetto on Intel, Freedreno, Panfrost, V3D; none on RADV | ingest; RADV render stages are a candidate upstream contribution | 2 |
| Display time | `VK_EXT_present_timing` (Mesa drivers); `wp_presentation` | layer reads present timing where available, `wp_presentation` otherwise, and records which | 1 |
| Submit, acquire, present, pipeline creation, device loss, on any driver including NVIDIA | none general; MangoHud covers present intervals | Datum Vulkan layer, emitting Perfetto events | 0 |
| Frame identity across layers | none | Datum core | 0 to 1 |
| Late-frame attribution | none (stutter is correlational, CPU only) | Datum core | 1 |
| Statistical methodology, fingerprints | none standard | Datum core | 0 |
| Compositor latch and deadline misses | partial (Mesa WSI timing, client side) | collaborator compositor | 2 |
| GPU hang and reset report | scattered (logs, tracepoints, devcoredump, `VK_EXT_device_fault`) | Datum gpu-health | 1 stretch, 2 |

### 6.1 Vulkan layer (Phase 0 core)

An implicit layer, enabled by an environment variable and installed through
a JSON manifest in the Vulkan loader's implicit layer search path. It sits
between the application (or DXVK or VKD3D-Proton) and the driver. It is a
Perfetto producer: it uses the Perfetto C++ SDK and emits its events as
Perfetto track events. It intercepts:

* `vkQueueSubmit` and `vkQueueSubmit2`: CPU timestamp, queue, and the
  semaphores and fences involved.
* `vkAcquireNextImageKHR`: CPU time spent blocked waiting for a swapchain
  image. Long acquire waits indicate back-pressure from the compositor or
  display.
* `vkQueuePresentKHR`: CPU timestamp, swapchain image index, present mode,
  and a present ID where `VK_KHR_present_id` is supported.
* Pipeline creation calls: duration of each, which exposes shader
  compilation stalls caused by DXVK or VKD3D-Proton without modifying
  either.
* Device loss: the first `VK_ERROR_DEVICE_LOST` return, plus
  `VK_EXT_device_fault` data where the driver supports it.

GPU execution time is measured by injecting timestamp queries at the start
and end of each frame's submitted work. GPU timestamps are converted to the
CPU clock using `VK_KHR_calibrated_timestamps` (or its EXT predecessor)
with the `CLOCK_MONOTONIC` time domain. Calibration is repeated during a
capture, because the two clocks drift.

Display time sources are in section 6.2. The layer may enable
`VK_EXT_present_timing` itself when the application did not.
TODO(verify): whether that requires swapchain creation changes the
application can observe.

The layer must add negligible overhead. Proposed budget: under 1 percent
change in mean frametime and no measurable change in 1 percent lows,
verified by A/B runs with the layer loaded and inactive versus active. The
active case includes the Perfetto SDK's own overhead; it is not measured
separately and subtracted.

**Important distinction.** The interval between two `vkQueuePresentKHR`
calls is when the application handed frames to the presentation engine. It
is not when frames reached the display. MangoHud style frametime reports the
first. Users experience the second. Datum records both and labels which is
which.

### 6.2 Compositor and presentation

On Wayland, the compositor decides when a client buffer is shown. Display
time is taken from the first available source, in this order, and every
frame records which source it used:

1. **`VK_EXT_present_timing`**, where the driver supports it (section 1).
   No Wayland-specific code is needed.
2. **`wp_presentation`**, otherwise. The protocol reports, per surface
   commit, the time the content was presented, the refresh interval, a
   display sequence counter, and flags such as whether the time came from
   hardware vblank.

Phase 1 collects either source on the client side, through the Vulkan
layer where the Wayland surface is accessible, or through a small helper.
Mesa's Wayland WSI timing work [3], if merged, supplies client-side
acquire-to-present data on Mesa drivers, and Datum ingests it. Phase 2
adds compositor-side tracing in one collaborator compositor: when the
compositor received the buffer, when it composed or scanned it out
directly, and whether it missed its commit deadline.

XWayland is a layer between the game and the compositor. Proton games run
as X11 clients by default, on Wayland desktops and under Gamescope, so the
Wayland client the compositor sees is XWayland, not the game.
`wp_presentation` feedback for those frames goes to XWayland, where the
game's Vulkan layer cannot reach it, so the fallback does not apply.
Display time for XWayland clients comes from `VK_EXT_present_timing`
through the X11 WSI. TODO(verify): how present
timing behaves for XWayland clients on the NVIDIA proprietary driver:
whether the extension is exposed on the X11 WSI there, and what its
reported times are derived from.

Gamescope is a special case: it is itself a Vulkan application and a
compositor. It can be instrumented from both sides.

### 6.3 Kernel: DRM/KMS and the GPU scheduler

Kernel events come from Perfetto's ftrace collector (`traced_probes`)
wherever it can supply them [4]. Datum ships the trace configuration that
enables the tracepoints it needs and ingests the result. Custom BPF is
written only where ftrace cannot provide what is needed, for example
in-kernel filtering or aggregation. BPF programs are CO-RE programs in C,
built with libbpf, so one build runs across kernel versions where the
underlying structures permit.

Sources:

* `drm` tracepoints for vblank events and their delivery to userspace.
* `gpu_scheduler` tracepoints for job queueing, execution start, and
  completion. Tracepoint names and fields have changed across kernel
  versions, so the tracepoint set (and any BPF probe set) is selected per
  kernel version, and the kernel version is part of every capture's
  fingerprint.
* `dma_fence` tracepoints for fence emission and signaling. Fences are the
  link between a submitted job, its completion, and a compositor waiting on
  it.
* Driver-specific tracepoints (nouveau first, amdgpu when hardware is
  available) for submission ioctls and ring activity.
* `sched:sched_switch` and `sched:sched_wakeup` for game, Proton, driver,
  and compositor threads: whether a thread was running, runnable but not
  scheduled, or blocked.

TODO(verify): which of the `drm` vblank, `dma_fence`, nouveau, and amdgpu
tracepoints `trace_processor` parses into tables, and which arrive only as
raw ftrace events. TODO(verify): which driver-specific tracepoints nouveau
exposes for submission and ring activity on the RTX 4080 (GSP firmware).

Clocks are covered in section 8.1.

### 6.4 GPU hang and reset diagnostics

When a GPU job does not complete, the kernel GPU scheduler's timeout fires,
the driver attempts recovery (soft recovery, queue reset, or full device
reset depending on driver and hardware), and the application eventually
sees `VK_ERROR_DEVICE_LOST`. The evidence is scattered: kernel log lines,
scheduler and driver tracepoints, a devcoredump in
`/sys/class/devcoredump/` if the driver produces one, and the application's
error.

Datum's GPU health component collects these into one report:

* the job that timed out, with its submitting process and context;
* the frames in flight and the jobs queued behind the hung job;
* the time from submission to timeout and the recovery path taken;
* the devcoredump contents, where present;
* what the application observed and when.

## 7. Components and language choices

### 7.1 Languages

**Decided.** Datum is written in C++ (restricted subset), with C for BPF
and kernel work and Nix for configuration.

| Component | Language | Rationale |
|---|---|---|
| Vulkan capture layer | C++, restricted subset | Reference Vulkan layers and the Perfetto SDK are C++. |
| DXVK / VKD3D-Proton instrumentation | C++, restricted subset | Both projects are C++; any upstream patch must match them. |
| Wayland tracing | C++ | Gamescope and KWin are C++. Patches inside a C compositor are written in C and live upstream. |
| Analysis engine | C++ | Built on Perfetto `trace_processor`, which is C++. |
| Fingerprinting, workload harness, results DB | C++ | One toolchain end to end. |
| BPF probes (gaps only) | C | BPF programs are written in restricted C and compiled by clang. libbpf is the reference loader. |
| Kernel patches | C | Kernel requirement. |
| Configuration | Nix | Declarative system state with built-in rollback. |

**Why one language.** Beyond the reasons in the table, one language gives
one toolchain, one build system, and one set of review habits from capture
to analysis, which is what one author can keep fluent in at the planned
pace.

**C++ subset (Decided).** RAII for every resource, `std::vector`,
`std::unordered_map`, `std::unique_ptr`, and standard library algorithms.
No custom templates, no inheritance hierarchies, no exceptions crossing the
Vulkan API boundary. The reasons are maintainability by a C-first author,
predictable code generation inside game processes, and easier review by
upstream developers.

**Memory safety (Decided).** CI builds and tests every component with
AddressSanitizer and UndefinedBehaviorSanitizer. Every parser of external
input (traces, protocol messages, devcoredumps, fingerprint files) has a
libFuzzer target. All statistics code has unit tests.

**Capture side and analysis side are separate processes.** Capture code
does the minimum: take a timestamp, write a fixed-shape Perfetto event,
return. All interpretation happens later in the C++ analysis engine. This
keeps capture overhead low and keeps complex logic out of game processes
and out of the kernel.

### 7.2 Repository layout

One monorepo for the open core. The capture sources and the analysis engine
share Datum's track and event naming conventions and must change together;
separate repositories would force version coordination for every naming
change. Separate repositories are used where the license or ownership
differs:

* commercial tooling (section 12), under a separate license;
* upstream patches, which live in the upstream projects.

The directory layout is in [README.md](README.md).

## 8. Data flow

```
 capture (per layer)                  analysis (offline, C++)
 ----------------------               ---------------------------------------
 Perfetto ftrace    --\
 Mesa Perfetto      ---+    one       1. load trace through trace_processor
 Datum Vulkan layer ---+--> Perfetto  2. align clocks
 Datum BPF (gaps)   ---+    trace     3. assign events to frames
 GPU health         --/               4. compute per-frame segments
 fingerprint        -------------->   5. statistics (section 9)
                                      6. causal engine (section 8.4)
                                      7. reports: text, JSON, and Perfetto
                                         annotations
```

Perfetto traces are the capture format. The analysis engine reads them
through `trace_processor`, using SQL queries for selection and joins and
C++ for frame assignment, statistics, and the causal engine.

### 8.1 Clock domain

Perfetto traces default to `CLOCK_BOOTTIME`. `trace_processor` expresses
every event in trace time and converts timestamps recorded in other clock
domains using the clock snapshots stored in the trace: packets that pair
simultaneous readings of the clocks the tracing service samples [7]. Datum
uses that model instead of defining its own:

* Capture components record each timestamp in its native clock, tagged
  with that clock's ID. Nothing is converted at capture time.
* All analysis is expressed in trace time: `CLOCK_BOOTTIME` nanoseconds,
  as converted by `trace_processor`.
* An event is merged only if the trace holds snapshots for its clock.

Per source:

* CPU side events in the layer: `clock_gettime(CLOCK_MONOTONIC)`, tagged
  `CLOCK_MONOTONIC`. TODO(verify): how the Perfetto SDK records an
  explicit clock ID on a track event timestamp.
* DRM page flip and vblank event timestamps: `CLOCK_MONOTONIC` on current
  kernels.
* Ftrace events from `traced_probes`: converted by `trace_processor`.
  TODO(verify): which clock `traced_probes` configures for ftrace, and
  whether it needs a snapshot conversion.
* BPF events, where they exist: `bpf_ktime_get_ns()`, tagged
  `CLOCK_MONOTONIC`.
* GPU timestamps: calibrated to `CLOCK_MONOTONIC` with calibrated
  timestamps, then converted to trace time like any other
  `CLOCK_MONOTONIC` event. Each calibration sample records its maximum
  deviation. The error bound on a converted GPU time is that deviation
  plus the snapshot conversion error.
* `VK_EXT_present_timing`: TODO(verify): which time domains the extension
  reports in, and which of them the trace holds snapshots for.
* `wp_presentation` feedback: the protocol announces its clock. Datum tags
  the feedback with that clock and merges it only if the trace holds
  snapshots for it.

TODO(verify): the exact conversion error bounds: how often the tracing
service takes snapshots, how `trace_processor` maps a timestamp that falls
between two snapshots, and the error for a clock pair that diverges during
a capture (`CLOCK_MONOTONIC` and `CLOCK_BOOTTIME` across a suspend).

Any event whose clock cannot be established is dropped from correlation,
not guessed.

### 8.2 Frame identity

A frame is identified by (swapchain, present ID). Where `VK_KHR_present_id`
is unavailable, Datum assigns its own monotonically increasing ID per
swapchain at the layer level.

Events are linked to frames through explicit keys where they exist and time
windows where they do not:

* Vulkan submits and GPU timestamp pairs: the layer knows which submits
  precede which present.
* Kernel GPU jobs: linked through the submitting process and, where
  observable, the fences the frame's submissions signal.
* Display time: linked through the present (present timing) or through the
  surface commit that carried the frame's buffer (`wp_presentation`).
* CPU scheduling: windowed by frame boundaries, filtered to known threads.

Every link records how it was made (explicit key or time window), so the
causal engine can weight evidence accordingly.

### 8.3 Per-frame segments

Each frame is decomposed into consecutive segments:

1. **CPU frame work**: previous present to this frame's first submit,
   broken into running, runnable but not scheduled, and blocked time for
   the frame's threads.
2. **Submission**: first submit to present call.
3. **Acquire wait**: time blocked in `vkAcquireNextImageKHR`.
4. **GPU queue wait**: submit to GPU execution start.
5. **GPU execution**: start to end timestamp.
6. **Compositor latency**: GPU completion to the display time reported by
   `VK_EXT_present_timing` or `wp_presentation`.

The frame's **display interval** (time between this frame's and the
previous frame's display) is the user-visible quantity. A frame is **late**
when its display interval exceeds the target interval (refresh period, or
the frame pacing target under VRR or an FPS cap) by a threshold.

### 8.4 Causal engine

**Proposed for Phase 1: rules plus baseline comparison.** For each late
frame:

1. Compare each segment with that segment's distribution over the
   capture's on-time frames.
2. Compute each segment's excess over its baseline median.
3. Attach co-occurring evidence: pipeline creation during the frame, the
   game's render thread preempted or migrated, a CPU frequency drop, a
   fence wait in the compositor, a missed vblank, a GPU scheduler event.
4. Rank contributors by excess time, and report each with its evidence and
   how strongly that evidence is linked to the frame (section 8.2).

Example of the intended output:

```
frame 4812  displayed 41.6 ms after previous (target 16.7 ms)
  1. CPU frame work      +22.1 ms  render thread runnable, not running, 19.8 ms
                                   (preempted by kworker on CPU 3)
  2. pipeline creation   +3.4 ms   vkCreateGraphicsPipelines x2 during frame
  3. compositor latency  +0.9 ms   within normal range
```

This is a heuristic, not proof of causation. Reports state that.
Phase 2 and later may replace it with a model learned from controlled
experiments (inject a known delay into one layer, check that Datum
attributes it correctly). Those injection experiments are also how the
Phase 1 engine is validated.

### 8.5 Fingerprint

Recorded with every capture: CPU model and microcode version, GPU model and
PCI ID, kernel version and command line, Mesa or proprietary driver
version, Vulkan loader version, Proton and DXVK or VKD3D-Proton versions,
compositor and version, display mode and VRR state, CPU governor and
scheduler (including sched_ext scheduler if any), relevant sysctls, power
profile, and shader cache state (cold or warm). The (kernel, driver,
compositor, Mesa) tuple is part of each result's identity, and results
become stale when that tuple moves.

### 8.6 Trace format

**Decided:** Perfetto is the canonical trace format. Upstream developers
can open a Perfetto trace in a browser without installing anything, and
the existing per-layer sources already write it.

* Datum defines its own track and event naming conventions (track names,
  event names, argument keys, and their units) and documents them in
  `docs/`. The conventions are versioned.
* Analysis results (per-frame segments and causal reports) are written as
  text, as JSON, and back into a Perfetto trace as annotations, so they
  show in the Perfetto UI next to the events they explain.

## 9. Statistical methodology

Bad statistics would permanently damage the project's credibility, so the
methodology is defined before the first public result.

### 9.1 Metrics

* **Mean frametime** and **mean FPS**, reported but never alone.
* **Frametime percentiles**: P50, P95, P99, P99.9.
* **"1% low"**: the term has two common definitions (the 99th percentile
  frametime expressed as FPS, or the mean of the worst 1 percent of
  frametimes expressed as FPS). **Proposed:** Datum reports the 99th
  percentile definition, names it explicitly in every report, and never
  compares its numbers with tools that use the other definition.
* **Late frame count** relative to the display target (section 8.3), which
  is closer to perceived stutter than any percentile.

Metrics are computed on display intervals when display time is available,
and on present-to-present intervals otherwise, labeled accordingly.

### 9.2 Confidence intervals

Frametimes within one run are autocorrelated: a slow frame is often
followed by more slow frames. Treating them as independent samples and
bootstrapping individual frames gives confidence intervals that are too
narrow.

**Proposed:**

* The unit of replication is the **run**. Each configuration is measured
  with at least N independent runs (N = 5 initially, revised after
  measuring between-run variance).
* A metric's confidence interval is computed across runs (percentile
  bootstrap over run-level values, or a t-interval when the run-level
  distribution supports it).
* Within-run uncertainty, when reported, uses a block bootstrap with block
  length chosen from the measured autocorrelation.

### 9.3 Controls

* Shader caches are either cleared before every run (cold) or primed by a
  discarded warm-up run (warm). The state is recorded and the two are never
  pooled.
* A fixed warm-up period is discarded from every run.
* Thermal state is recorded at run start; runs starting above a threshold
  are delayed.
* Background load is recorded from the scheduler trace, not assumed.
* Run order is interleaved across configurations (A B A B, not A A B B) so
  slow drift is not mistaken for a configuration effect.
* When many configurations are compared, the report says how many
  comparisons were made, and Pillar II applies a multiple-comparisons
  correction.

## 10. Phases and milestones

Planning pace: **8 to 12 hours per week**, alongside employment.
[ROADMAP.md](ROADMAP.md) breaks Phase 0 into ordered work items.

### 10.1 Current status (2026-10-24)

* Work started: 2026-10-24.
* Design: this document, version 0.1.
* Vulkan fundamentals: working through the Vulkan Tutorial; validation
  layers reached.
* Datum code: none yet.
* Upstream bug reports and patches: none submitted yet.
* Hardware: primary machine is an NVIDIA RTX 4080 desktop on Arch Linux
  with Hyprland, on the proprietary driver. A nouveau/NVK boot entry is not
  yet set up.

The Phase 0 Vulkan layer prototype works on the proprietary NVIDIA driver,
because the layer sits above the driver. An open kernel GPU driver is
required from Phase 1, for kernel GPU events. nouveau with NVK on the
existing RTX 4080 qualifies. AMD (amdgpu, RADV) remains desirable for
Steam Deck relevance.

### 10.2 Phase 0: on-ramp

| Milestone | Exit criterion |
|---|---|
| Vulkan foundations | Rendering loop with two frames in flight, written and understood: every semaphore and fence in it can be explained. |
| Read the real tools | MangoHud built from source with one custom metric added; Gamescope built from source with its present-timing path identified. |
| Open kernel driver | nouveau/NVK boot entry working on the primary machine. |
| Ecosystem standing | Three bug reports with full traces, repro steps, and hardware details filed upstream; three patches merged (MangoHud, a compositor or Gamescope, and one more). |
| Prototype | Vulkan layer emitting Perfetto events for present-to-present intervals, display time (where available), and GPU execution time for one game; a C++ analyzer on `trace_processor` producing the section 9 metrics with confidence intervals; tests, sanitizer CI, one fuzz target, and a README a stranger can follow. |
| Writeup | Three games on three configurations, published with methodology; at least one reply from a practitioner. |
| Collaborator choice | Phase 1 upstream collaborator project chosen from the D1 candidates (section 14). |

### 10.3 Phase 1: frame causality

| Milestone | Exit criterion |
|---|---|
| Second layer | Perfetto ftrace capture of CPU scheduling and DRM/KMS events, aligned with the Vulkan layer in Perfetto trace time (section 8.1). |
| Display time | Display time collected from `VK_EXT_present_timing` or `wp_presentation`, source recorded; display intervals reported. |
| Kernel GPU events | GPU scheduler and fence events linked to frames on nouveau (amdgpu when hardware is available). |
| Causal engine v0 | Section 8.4 engine produces ranked late-frame reports, validated by delay-injection experiments in at least two layers. |
| Perfetto annotations | Captures open in the Perfetto UI with all layers on one timeline and Datum's per-frame segments and reports shown as annotations. |
| GPU health v0 (stretch) | One report combining a scheduler timeout, the recovery path, frames in flight, and the application's device-lost error. |
| Success | Reliable per-frame causal report on three games. |

### 10.4 Later phases

| Phase | Deliverable | Success criterion |
|---|---|---|
| 2 | Compositor-side tracing in one collaborator project; translation layer events; first public release | An upstream developer uses a Datum trace to file or fix a bug. |
| 3 | Workload suite, fingerprint schema v1, results database (Pillar II) | Two independent users produce comparable results on the same workload. |
| 4 | Regression detection across kernel, driver, and compositor versions; CI integration with one upstream project | A real regression is caught before a stable release. |
| 5 | NixOS module (Pillar III) | A non-technical user installs a Datum profile and benefits measurably, with rollback. |
| 6 | Distribution integrations, broader workload coverage, methodology paper | A distribution maintainer cites Datum data in a configuration decision. |
| 7 | Open implementations of features that are proprietary today (latency reduction, frame pacing), validated by Datum | A measurably competitive open implementation of one such feature. |

## 11. Licensing plan

**Decided.** Three license zones in the open core:

| Zone | License | Reason |
|---|---|---|
| Default | `MIT` | Fits the ecosystem Datum works in: Mesa, MangoHud, Wayland, wlroots, and libdrm are MIT licensed (Mesa mostly). MIT is GPL-2.0 compatible. |
| `capture/bpf/` | `GPL-2.0-only OR MIT` | The kernel only lets BPF programs that declare a GPL-compatible license call GPL-only helpers, which tracing needs. The kernel accepts `Dual MIT/GPL` but not plain `MIT`, so the default zone cannot be used. |
| `kernel-patches/` | `GPL-2.0-only` | Kernel requirement. |

The Perfetto SDK is Apache-2.0, which is compatible with the default zone
(obligations in [NOTICE](NOTICE)). Code written inside other projects is
under those projects' licenses ([OWNERSHIP.md](OWNERSHIP.md)).

Contributions use the DCO, not a CLA: the permissive license already allows
any downstream use of contributions, including by the project owner.

**Commercial tier (Proposed).** Any future commercial tooling lives in
separate repositories under a separate license. Rules:

* The open core never depends on commercial code.
* Anything an upstream developer needs to diagnose a bug stays in the open
  core. Moving diagnostic capability behind a paywall would break the
  adoption strategy in section 3.
* Commercial tooling may depend on the open core like any other user.

The commercial license is **Open** (section 14). The intent is free use by
individual developers and paid use by companies. PolyForm Small Business
does not implement that intent: it is free for any organization under its
size and revenue thresholds, including small studios. A proprietary
license with an individual-use grant fits better. Either choice requires
legal review.

## 12. Commercial direction

No commercial activity is conducted at this time. This section records the
intended long-term direction so the open core is designed to support it.

The open core produces measurement capability and public data. Commercial
value comes from applying that capability to a specific customer's problem:

* **Performance engineering services** for studios shipping on Linux and
  Linux-based handhelds such as the Steam Deck: diagnosing frame pacing,
  stutter, and GPU stability problems in their titles across the stack,
  and fixing them in the game, the translation layer, or upstream.
* **Commercial tooling** built on the open core for teams that need what
  individuals do not: continuous performance regression tracking in a
  studio's build pipeline, private results databases for unreleased titles,
  and fleet-level configuration management.
* **Long term**, a development environment that makes Linux the easiest
  platform to build, test, profile, and ship games on, with Datum's
  measurement substrate underneath.

The ordering is fixed: the open substrate first, adoption by upstream
developers second, commercial work third.

## 13. Risks

* **Adoption.** Infrastructure projects attract attention and few users.
  Mitigation: build in dialogue with one or two upstream collaborators from
  Phase 1, and write output that opens in a viewer graphics developers
  already use (the Perfetto UI).
* **Dependency on Perfetto.** Datum's trace format, capture SDK, and
  analysis input all come from one project. Mitigation: Perfetto is widely
  used and open; Datum's analysis logic is isolated behind its own frame
  model, so the input source could change.
* **NVIDIA visibility.** Much Linux gaming pain is NVIDIA specific and the
  proprietary driver is opaque. Mitigation: measure everything visible
  outside the driver and state the gap.
* **Moving targets.** Explicit sync, HDR, scheduler work, and tracepoint
  definitions change monthly. Mitigation: version tuples in every result,
  per-kernel tracepoint and probe selection, and staleness in the data
  model.
* **Methodology errors.** Precise-looking wrong numbers. Mitigation:
  section 9 is in force from the first prototype.
* **Measurement overhead.** A tool that changes what it measures.
  Mitigation: overhead budget and A/B verification (section 6.1).
* **Scope creep.** Launchers, GUIs, store integration, anti-cheat.
  Mitigation: section 2.2.
* **Author time.** A side project at 8 to 12 hours per week. Mitigation:
  small phases with concrete exit criteria, and a tool the author uses
  himself.

## 14. Open decisions

| ID | Question | Current lean |
|---|---|---|
| D1 | Phase 1 upstream collaborator | Gamescope (measurement-friendly, Steam Deck relevance), or Hyprland or KWin. Decide at the end of Phase 0. |
| D3 | Confidence interval method | Run-level bootstrap (section 9.2). Validate with measured variance. |
| D4 | Open-driver hardware | nouveau/NVK on the existing RTX 4080 for Phase 1; a used AMD RDNA2 or RDNA3 card when available, for Steam Deck relevance. |
| D5 | Commercial tier license | Proprietary with individual-use grant, pending legal review. |
| D6 | Layer dispatch | Hand-written dispatch, or the helpers in Vulkan-Utility-Libraries. Decide at prototype start. |

## 15. Revision history

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-24 | First published version. |

## 16. References

1. Mesa merges `VK_EXT_present_timing` for Mesa 26.1 (Phoronix):
   https://www.phoronix.com/news/Mesa-Merges-Present-Timing
2. Mesa documentation, Perfetto tracing:
   https://docs.mesa3d.org/perfetto.html
3. Mesa merge request 28634, Perfetto instrumentation for the Wayland WSI
   (`MESA_VK_WSI_DEBUG=wltiming`), merge status unverified:
   https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/28634
4. Perfetto documentation, GPU data sources:
   https://perfetto.dev/docs/data-sources/gpu
5. P2949/stutter:
   https://github.com/P2949/stutter
6. Perfetto documentation, Android FrameTimeline:
   https://perfetto.dev/docs/data-sources/frametimeline
7. Perfetto documentation, clock synchronization:
   https://perfetto.dev/docs/concepts/clock-sync
