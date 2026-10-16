# Running workflows

## Run CAMB from an `.ini` file

From the repository root:

```bash
cd WHM-CAMB
../.venv/bin/camb inifiles/planck_2018_WebHaloModel2000.ini
```

The output prefix is controlled by `output_root` in the configuration. The
example outputs are stored under `WHM-CAMB/fortran/output/` when run with the
repository's existing paths.

## Run CAMB from Python

Use the local package, not a globally installed CAMB:

```bash
which python
.venv/bin/python -c "import camb; print(camb.__file__)"
```

A correct path points into this checkout, typically:

```text
.../WHM-CAMB/camb/__init__.py
```

Minimal WHM power-spectrum example:

```python
import camb
from camb.nonlinear import halofit_brieden2025_tweaked

pars = camb.CAMBparams()
pars.set_cosmology(H0=67.5, ombh2=0.0224, omch2=0.12)
pars.InitPower.set_params(As=2.1e-9, ns=0.965)
pars.set_matter_power(redshifts=[0.0], kmax=10.0)
pars.NonLinear = camb.model.NonLinear_both
pars.NonLinearModel.set_params(
    halofit_version=halofit_brieden2025_tweaked,
    WHM_hmf=0,
    WHM_thinweb=0,
)

results = camb.get_results(pars)
kh, z, pk = results.get_matter_power_spectrum(
    minkh=1e-3, maxkh=10.0, npoints=200
)
```

## Notebook working directory

Notebook paths are relative to `WHM-notebooks`. Always change directory before
launching Jupyter or `nbconvert`:

```bash
cd WHM-notebooks
../.venv/bin/jupyter lab
```

Running from the repository root can cause missing-file errors because the
notebooks refer to paths such as `data/Pklin/` and `figures/`.

## Cached notebook execution

The default flags in `WHMcode.ipynb` load existing cached spectra. This is the
recommended first run:

```bash
cd WHM-notebooks
mkdir -p /tmp/whm-cache
MPLCONFIGDIR=/tmp/whm-cache \
XDG_CACHE_HOME=/tmp/whm-cache \
DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib \
../.venv/bin/jupyter nbconvert --to notebook --execute \
WHMcode.ipynb --output /tmp/WHMcode_run.ipynb
```

The main notebook writes paper figures to `figures/`. The executed notebook is
written outside the repository in `/tmp` to avoid accidental large notebook
diffs.

The independent window-function notebook is faster:

```bash
../.venv/bin/jupyter nbconvert --to notebook --execute \
whmcode_windowfunctions.ipynb --output /tmp/whm_windowfunctions_run.ipynb
```

## Full regeneration

The main notebook supports environment overrides:

```bash
WHM_SAMPLE=BACCO \
WHM_RECOMPUTE=all \
WHM_WRITE=all \
../.venv/bin/jupyter nbconvert --to notebook --execute \
WHMcode.ipynb --output /tmp/WHMcode_BACCO_run.ipynb
```

Valid values for `WHM_SAMPLE` are `BACCO`, `EE2`, and `CosmicEmu`.
`WHM_NCOSMO` limits the number of cosmologies and is useful for a small test:

```bash
WHM_SAMPLE=BACCO WHM_NCOSMO=2 \
WHM_RECOMPUTE=all WHM_WRITE=all \
../.venv/bin/jupyter nbconvert --to notebook --execute \
WHMcode.ipynb --output /tmp/WHMcode_test_run.ipynb
```

Full regeneration can take several hours. It runs per-cosmology CAMB and
velocileptors calculations and may download emulator data.

## Notebook outputs

The notebooks produce:

- `halomassfunctions.png` — mass-function comparison;
- `WHMcode_wk_v3.png` — analytic window functions;
- `WHMcode_spectra_comp_*.png` — model spectra comparisons;
- `w0waMnu_LPT_*.png` — LPT plus WHM comparisons;
- `theo_vs_measured_p1h_*.png` — one-halo term comparisons;
- `HMF_cumulative_integrand_comparison.png` — cumulative HMF diagnostics;
- emulator comparison and LPT scatter figures from the scatter notebook.

Cached numerical data are under `WHM-notebooks/data/`, especially `Pklin/`,
`PkLPT/`, `PkWHMhf/`, `PkBacco/`, `PkEE2/`, and `knl/`.
