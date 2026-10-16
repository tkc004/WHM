# Web-Halo Model (WHM)

This repository contains the Web-Halo Model (WHM) implementation associated
with Brieden, Beutler, and Pellejero-Ibañez,
[arXiv:2508.10902](https://arxiv.org/abs/2508.10902).

WHM extends the standard halo model by including matter collapsed into sheets
and filaments in addition to haloes. These contributions provide a smoother
description of the transition between perturbative two-halo scales and the
non-linear one-halo regime. The implementation is integrated into a modified
CAMB distribution and is used together with 1-loop Lagrangian perturbation
theory in the paper notebooks.

## What is included

This repository contains:

- A modified CAMB source tree with WHM support.
- WHM halo, filament, and sheet model variants.
- Alternative halo mass functions and sheet/filament profile choices.
- Paper-figure notebooks and cached spectra.
- Example CAMB `.ini` files.
- An import smoke test for the complete Python environment.

The main WHM implementation is in
[`WHM-CAMB/fortran/halofit.f90`](WHM-CAMB/fortran/halofit.f90).

There is no separate WHM-specific CLASS source tree in this checkout. The
notebooks use an external CLASS 3.3.4 installation through `classy` for
linear spectra and comparison calculations.

## Repository layout

```text
.
├── WHM-CAMB/
│   ├── camb/                  Python CAMB wrapper
│   ├── fortran/halofit.f90   WHM and HMcode implementation
│   └── inifiles/              Example CAMB configurations
├── WHM-notebooks/
│   ├── WHMcode.ipynb         Main paper-figure workflow
│   ├── whmcode_windowfunctions.ipynb
│   ├── WHMcode_LPT_scatter.ipynb
│   ├── data/                  Input data and cached spectra
│   ├── figures/               Generated figures
│   └── env/                   Environment setup and smoke test
├── skills.md                 Repository development guide
└── README.md
```

## Requirements

The core CAMB build requires:

- Python 3.11 or 3.12.
- A Fortran compiler, such as `gfortran`.
- A C/C++ compiler for CLASS if `classy` is installed.
- NumPy, SciPy, and SymPy.

The full notebook workflow additionally uses:

- Jupyter and Matplotlib.
- `euclidemu2`.
- `baccoemu` and its emulator data.
- `velocileptors`.
- `FlamingoBaryonResponseEmulator`.
- CLASS 3.3.4 through `classy`.
- A LaTeX installation because the figures use Matplotlib's `usetex` mode.

## Installation

### Recommended: local Python environment

From the repository root:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip 'setuptools<81' wheel
.venv/bin/pip install numpy scipy matplotlib jupyter cython pyfftw
```

The `setuptools<81` constraint is needed because the local CAMB build imports
`pkg_resources` during compilation.

Build and install the modified CAMB rather than the published PyPI package:

```bash
.venv/bin/pip install -e ./WHM-CAMB --no-build-isolation
```

On macOS, if the linker reports `library 'System' not found`, retry with the
active Command Line Tools SDK:

```bash
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk \
MACOSX_DEPLOYMENT_TARGET=15.0 \
.venv/bin/pip install -e ./WHM-CAMB --no-build-isolation
```

Install the notebook dependencies:

```bash
.venv/bin/pip install euclidemu2 baccoemu \
  'git+https://github.com/sfschen/velocileptors' \
  'git+https://github.com/FLAMINGOSIM/FlamingoBaryonResponseEmulator'
```

The first `baccoemu` initialization downloads its emulator data. Network
access is therefore needed both during installation and on the first use.

### Conda setup

The repository also provides a Conda environment specification:

```bash
conda env create -f WHM-notebooks/env/environment.yml
conda activate whm
pip install -e ./WHM-CAMB --no-build-isolation
```

The automated setup helper is:

```bash
bash WHM-notebooks/env/setup_whm_env.sh
```

The helper is primarily targeted at native Apple Silicon and installs the
local CAMB build plus the notebook dependencies.

## CLASS / classy

The main notebook imports `classy`. Build CLASS 3.3.4 separately:

```bash
git clone --depth 1 --branch v3.3.4 \
  https://github.com/lesgourg/class_public /tmp/class_public

make -C /tmp/class_public -j4 class
make -C /tmp/class_public libclass.a

CC=/usr/bin/clang CXX=/usr/bin/clang++ \
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk \
MACOSX_DEPLOYMENT_TARGET=15.0 \
.venv/bin/pip install /tmp/class_public --no-build-isolation
```

`classy` is optional for the basic CAMB smoke test, but required for the full
`WHMcode.ipynb` workflow.

## macOS native libraries

EuclidEmulator2 may require GSL and OpenBLAS:

```bash
brew install gsl openblas
```

Because OpenBLAS is usually keg-only, expose its libraries at runtime:

```bash
export DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib
```

Using `DYLD_FALLBACK_LIBRARY_PATH` is preferred here; putting OpenBLAS in
`DYLD_LIBRARY_PATH` can interfere with NumPy's Accelerate-linked libraries.

## Verify the installation

Run the smoke test from the repository root:

```bash
MPLCONFIGDIR=/tmp/whm-cache \
XDG_CACHE_HOME=/tmp/whm-cache \
DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib \
.venv/bin/python WHM-notebooks/env/import_smoketest.py
```

To require CLASS during the test:

```bash
WHM_REQUIRE_CLASSY=1 \
MPLCONFIGDIR=/tmp/whm-cache \
XDG_CACHE_HOME=/tmp/whm-cache \
DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib \
.venv/bin/python WHM-notebooks/env/import_smoketest.py
```

A successful full test reports `SMOKETEST PASSED` and confirms:

- local CAMB is being imported from `WHM-CAMB`;
- the Brieden WHM modes are available;
- `baccoemu` can instantiate its matter-power emulator;
- `velocileptors`, EuclidEmulator2, and the Flamingo emulator import;
- `classy` is available when `WHM_REQUIRE_CLASSY=1` is set.

## Running the WHM model with CAMB

The WHM model is selected with `halofit_version`. Important modes defined in
`halofit.f90` include:

| Value | Meaning |
|---:|---|
| `11` | Baseline Brieden 2025 WHM model |
| `14` | Tweaked Brieden 2025 WHM model used by the main workflow |
| `39` | Tweaked model with thin-web sheet/filament profiles |

Additional WHM switches are:

| Parameter | Values | Meaning |
|---|---|---|
| `WHM_hmf` | `0` | Sheth & Tormen mass function, baseline |
| `WHM_hmf` | `1` | Warren et al. (2006) mass function |
| `WHM_hmf` | `2` | Despali et al. (2016) mass function |
| `WHM_thinweb` | `0` | Standard constant-density sheet/filament profiles |
| `WHM_thinweb` | `1` | Thin-web sheet/filament profiles |

Run an example configuration:

```bash
cd WHM-CAMB
../.venv/bin/camb inifiles/planck_2018_WebHaloModel2000.ini
```

The Python API exposes the same settings through
`NonLinearModel.set_params`, for example:

```python
import camb
from camb.nonlinear import halofit_brieden2025_tweaked

pars = camb.CAMBparams()
pars.set_cosmology(H0=67.5, ombh2=0.0224, omch2=0.12)
pars.NonLinear = camb.model.NonLinear_both
pars.NonLinearModel.set_params(
    halofit_version=halofit_brieden2025_tweaked,
    WHM_hmf=0,
    WHM_thinweb=0,
)

results = camb.get_results(pars)
```

## Running the notebooks

Run notebooks from `WHM-notebooks`, because their relative paths assume that
working directory:

```bash
cd WHM-notebooks
../.venv/bin/jupyter lab
```

The notebooks are:

- `WHMcode.ipynb` — main WHM spectra and most paper figures.
- `whmcode_windowfunctions.ipynb` — analytic sheet, filament, and halo
  window functions.
- `WHMcode_LPT_scatter.ipynb` — emulator comparison and LPT scatter plots.

### Cached headless execution

The default notebook settings load the cached spectra under `data/` and are
the fastest way to verify the workflow:

```bash
cd WHM-notebooks

MPLCONFIGDIR=/tmp/whm-cache \
XDG_CACHE_HOME=/tmp/whm-cache \
DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib \
../.venv/bin/jupyter nbconvert --to notebook --execute \
WHMcode.ipynb --output /tmp/WHMcode_run.ipynb
```

The window-function notebook can be run independently:

```bash
../.venv/bin/jupyter nbconvert --to notebook --execute \
whmcode_windowfunctions.ipynb --output /tmp/whm_windowfunctions_run.ipynb
```

### Regenerating spectra

For headless runs, set the sample and recomputation flags through environment
variables:

```bash
WHM_SAMPLE=BACCO \
WHM_RECOMPUTE=all \
WHM_WRITE=all \
../.venv/bin/jupyter nbconvert --to notebook --execute \
WHMcode.ipynb --output /tmp/WHMcode_BACCO_run.ipynb
```

Valid samples are `BACCO`, `EE2`, and `CosmicEmu`. Use `WHM_NCOSMO` to limit
the number of cosmologies while testing:

```bash
WHM_SAMPLE=BACCO WHM_NCOSMO=2 \
WHM_RECOMPUTE=all WHM_WRITE=all \
../.venv/bin/jupyter nbconvert --to notebook --execute \
WHMcode.ipynb --output /tmp/WHMcode_test_run.ipynb
```

Full regeneration can take several hours because it runs CAMB, velocileptors,
and emulator calculations for many cosmologies. Cached execution is much
faster.

## Outputs and cached data

Generated figures are written to `WHM-notebooks/figures/`. Cached spectra and
intermediate data are stored in `WHM-notebooks/data/`, including:

- `PkWHMhf/` — CAMB WHM spectra;
- `PkLPT/` — 1-loop LPT spectra;
- `Pklin/` — linear spectra;
- `PkBacco/`, `PkEE2/`, and `CosmicEmu/` — emulator data;
- `knl/` — non-linear scale data.

Executed notebooks should normally be written outside the repository, such as
under `/tmp`, unless a notebook result is intentionally being versioned.

## Development notes

- Keep WHM implementation changes in
  `WHM-CAMB/fortran/halofit.f90`.
- Preserve the public `halofit_version`, `WHM_hmf`, and `WHM_thinweb`
  interfaces when possible.
- Rebuild the local CAMB package after Fortran changes.
- Run `import_smoketest.py` after dependency or build changes.
- Use cached data for plotting-only changes.
- Do not commit `.venv/`, compiler artifacts, emulator caches, or temporary
  executed notebooks. The local virtual environment is ignored by Git.

More repository-specific operating instructions are in
[`skills.md`](skills.md), and the notebook-specific guide is in
[`WHM-notebooks/README.md`](WHM-notebooks/README.md).
