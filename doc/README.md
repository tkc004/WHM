# WHM documentation

This directory contains detailed documentation for the Web-Halo Model (WHM)
repository. Start with the [installation guide](installation.md), then use
the [model guide](model.md) for WHM configuration and the
[workflow guide](workflows.md) for CAMB and notebook execution.

## Documentation map

- [Installation and dependencies](installation.md) — local virtualenv,
  Conda, CAMB, CLASS, macOS libraries, and emulator data.
- [Model and code architecture](model.md) — WHM concepts, source locations,
  model switches, and parameter mappings.
- [Running workflows](workflows.md) — CAMB `.ini` files, Python API, cached
  notebooks, full regeneration, and outputs.
- [Troubleshooting](troubleshooting.md) — common build, linker, dependency,
  notebook, and data-download failures.
- [Development guide](development.md) — source changes, validation, generated
  files, and repository hygiene.

## Scope

The WHM implementation in this checkout is integrated into CAMB. CLASS is an
external dependency used by the notebooks through `classy`; there is no
separate WHM-CLASS source tree here.

## References

- Brieden, Beutler, and Pellejero-Ibañez, [Web-Halo Model (WHM),
  arXiv:2508.10902](https://arxiv.org/abs/2508.10902).
- Root project guide: [`../README.md`](../README.md).
- Repository operating notes: [`../skills.md`](../skills.md).
