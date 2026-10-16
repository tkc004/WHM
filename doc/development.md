# Development guide

## Source of truth

The WHM implementation is in:

```text
WHM-CAMB/fortran/halofit.f90
```

The source contains the public `halofit_version` constants, the `THalofit`
configuration type, the `WHM_hmf` and `WHM_thinweb` inputs, and the numerical
branches for halo, filament, and sheet calculations.

Notebook plotting and validation code is in:

```text
WHM-notebooks/WHMcode.ipynb
WHM-notebooks/WHMcode_LPT_scatter.ipynb
WHM-notebooks/whmcode_windowfunctions.ipynb
```

## Modifying Fortran code

After changing `halofit.f90` or another Fortran source, rebuild the local
extension:

```bash
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk \
MACOSX_DEPLOYMENT_TARGET=15.0 \
.venv/bin/pip install -e ./WHM-CAMB --no-build-isolation
```

Confirm that the new build exposes the WHM constants:

```bash
.venv/bin/python - <<'PY'
import camb
from camb.nonlinear import halofit_brieden2025_tweaked
print(camb.__file__)
print(halofit_brieden2025_tweaked)
PY
```

Avoid changing the numeric values of existing `halofit_version` selectors
without updating all `.ini` files, notebooks, and documentation.

## Validation checklist

For a source or dependency change:

1. Rebuild the local CAMB package.
2. Run `WHM-notebooks/env/import_smoketest.py`.
3. Run a small CAMB calculation with `halofit_version=14`.
4. Run the cached window-function notebook.
5. Run the cached main notebook if the change affects spectra or interfaces.

For plotting-only changes, use cached data and avoid full recomputation.

## Generated files

Expected generated outputs include:

- `WHM-notebooks/figures/*.png` from the paper notebooks;
- cached spectra under `WHM-notebooks/data/`;
- CAMB build products such as `camblib.so`, object files, and module files;
- executed notebooks written by `nbconvert`.

Build products and virtual environments are ignored by Git. Write executed
notebooks to `/tmp` unless they are intentionally part of a commit.

## Data and reproducibility

The notebooks distinguish between recomputation and writing of cached results.
For headless execution:

- `WHM_RECOMPUTE=all` recomputes all configured numerical inputs;
- `WHM_WRITE=all` writes the recomputed results to `data/`;
- `WHM_SAMPLE` selects `BACCO`, `EE2`, or `CosmicEmu`;
- `WHM_NCOSMO` limits the number of cosmologies for a short test.

Full regeneration is expensive. Keep cached inputs when the purpose is only to
inspect a figure or test plotting logic.

## Repository hygiene

- Do not commit `.venv/` or `*.venv/`.
- Do not commit temporary notebook outputs from `/tmp`.
- Avoid changing cached data and figures unless regeneration is intentional.
- Keep documentation synchronized with public model switches.
- Record dependency or compiler workarounds in `doc/troubleshooting.md`.
