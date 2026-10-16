# Model and code architecture

## Physical model

The standard halo model divides the non-linear matter power spectrum into a
two-halo term and a one-halo term. WHM retains this broad organization but
recognizes the intermediate collapse stages of the cosmic web:

1. Collapse along one axis produces sheets.
2. Collapse along two axes produces filaments.
3. Collapse along three axes produces haloes.

The additional sheet and filament terms regularize the perturbative breakdown
before final halo collapse and improve the two-halo to one-halo transition.
The paper combines the WHM correction with 1-loop LPT.

## Main source

The implementation is concentrated in:

```text
WHM-CAMB/fortran/halofit.f90
```

The Python wrapper and CAMB integration are in:

```text
WHM-CAMB/camb/
WHM-CAMB/fortran/classes.f90
WHM-CAMB/fortran/model.f90
```

`halofit.f90` contains both the existing HMcode variants and the WHM
extensions. The WHM code is not a standalone Python model; it is compiled into
CAMB's non-linear power-spectrum library.

## Public model selectors

The Fortran constants include the following primary selectors:

| `halofit_version` | Model |
|---:|---|
| `11` | `halofit_brieden2025`, baseline WHM |
| `12` | Brieden 2025 feedback variant |
| `13` | Brieden 2025 cross variant |
| `14` | `halofit_brieden2025_tweaked`, main tweaked WHM |
| `15` | Cross tweaked variant |
| `17` | Halo-only diagnostic |
| `18` | Filament-only diagnostic |
| `19` | Sheet-only diagnostic |
| `23` | Halo-minus-sphere diagnostic |
| `24` | Filament-minus-sphere diagnostic |
| `25` | Sheet-minus-sphere diagnostic |
| `39` | Tweaked WHM with thin-web profiles |

The complete list, including profile and bias diagnostics, is defined near the
top of `WHM-CAMB/fortran/halofit.f90`.

## WHM switches

These switches are read from CAMB `.ini` files and are also available through
the Python `NonLinearModel.set_params` interface:

| Parameter | Values | Meaning |
|---|---|---|
| `WHM_hmf` | `0` | Sheth & Tormen mass function, baseline |
| `WHM_hmf` | `1` | Warren et al. (2006) mass function |
| `WHM_hmf` | `2` | Despali et al. (2016) mass function |
| `WHM_thinweb` | `0` | Default constant-density sheet/filament profiles |
| `WHM_thinweb` | `1` | Thin-web sheet/filament profiles |

`WHM_thinweb=1` overrides the profile choice for Brieden modes and sets the
internal `ivar_profile=3` branch. The profile options are documented in the
Fortran type definition:

- `0`: constant-density profile;
- `1`: infinitely thin line/disk;
- `2`: NFW sphere with scale radius equal to the initial radius;
- `3`: thin-web limit multiplied by the NFW window of the corresponding halo.

## Configuration examples

The paper-oriented examples are:

```text
WHM-CAMB/inifiles/planck_2018_WHMcode2023.ini
WHM-CAMB/inifiles/planck_2018_WebHaloModel2000.ini
```

The latter selects the tweaked WHM mode with `halofit_version=14`. The former
selects the baseline Brieden mode with `halofit_version=11`.

## Python API mapping

The Python constants are imported from `camb.nonlinear`:

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

For a thin-web diagnostic, change `WHM_thinweb` to `1` or use the dedicated
thin-web selector exposed by the local wrapper.
