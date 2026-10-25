# Contributing

## Sign-off

Every commit needs a `Signed-off-by:` trailer certifying the Developer
Certificate of Origin 1.1 in [DCO](DCO), as in the Linux kernel and Mesa.
There is no CLA. Use `git commit -s`.

The name and email must match the commit author; anonymous sign-offs are not
accepted. Sign-off is not a cryptographic signature; `git commit -S` is
encouraged but not required.

By signing off you license your contribution under the file's SPDX header
(zones listed in [NOTICE](NOTICE)). Do not submit code you lack the right to
license: code your employer claims, code from incompatibly licensed projects,
or code generated from proprietary sources.

## SPDX headers

Every source file starts with an SPDX identifier and a copyright line:

```c
// SPDX-License-Identifier: MIT
// Copyright (c) 2026 Chandler Camarena
```

Use `#` comments for Nix, shell, Python, TOML, and YAML. Files under
`capture/bpf/` use `GPL-2.0-only OR MIT`, and each BPF object must declare a
GPL-compatible license string so the kernel permits GPL-only helpers:

```c
char LICENSE[] SEC("license") = "Dual MIT/GPL";
```

Add your own copyright line below the existing ones when you substantially
change a file.

## Patches

* One logical change per commit, kernel-style message: `component: summary`,
  blank line, then why the change is needed and how it was tested.
* Measurement changes state what was measured, on what hardware, and how
  many runs. Unverified performance claims are not merged.
* Changes must pass the sanitizer CI, new parsers of external input come with
  a libFuzzer target, and statistics changes come with unit tests
  ([DESIGN.md](DESIGN.md) section 7.1).
