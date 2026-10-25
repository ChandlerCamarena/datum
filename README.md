# Project Datum

Per-frame causal reports for the Linux gaming graphics stack, from the game
down to the display. Datum joins existing per-layer data (Perfetto ftrace,
Mesa, `VK_EXT_present_timing`) into one record per frame, explains why late
frames were late, and reports results with sound statistics. Its output opens
in the Perfetto UI.

**Status: pre-alpha, design stage.** Nothing is usable yet. See
[DESIGN.md](DESIGN.md) for the architecture and [ROADMAP.md](ROADMAP.md) for
current work.

## Layout

| Path                    | Component                               |
|-------------------------|-----------------------------------------|
| `capture/vulkan-layer/` | Vulkan layer (Perfetto producer)        |
| `capture/dxvk-hooks/`   | D3D translation layer instrumentation   |
| `capture/bpf/`          | BPF probes for ftrace gaps              |
| `capture/wayland/`      | Compositor and protocol tracing         |
| `capture/gpu-health/`   | GPU hang and reset diagnostics          |
| `analysis/`             | Trace model, correlation, causal engine |
| `results-db/`           | Public results database                 |
| `workloads/`            | Workload suite and fingerprinting       |
| `config/nix/`           | Declarative configuration               |
| `kernel-patches/`       | Copies of upstream kernel work          |
| `docs/`                 | Writeups and methodology                |

## Contributing and license

See [CONTRIBUTING.md](CONTRIBUTING.md). MIT, with exceptions listed in
[NOTICE](NOTICE). Ownership: [OWNERSHIP.md](OWNERSHIP.md).

Copyright (c) 2026 Chandler Camarena
