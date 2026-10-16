# WHM implementation in `halofit.f90`

This document explains how the Web-Halo Model (WHM) described in
[arXiv:2508.10902](https://arxiv.org/abs/2508.10902) is implemented in
[`WHM-CAMB/fortran/halofit.f90`](../WHM-CAMB/fortran/halofit.f90).

The file contains both the original HMcode/Halofit-style models and the WHM
extensions. The WHM path is selected through `halofit_version` and is then
handled mainly by `p_1h`.

## 1. Selecting a WHM variant

The public selectors are defined near the start of
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L64):

| Public selector | Meaning |
| ---: | --- |
| `11` | Baseline WHM |
| `13` | WHM cross-term variant |
| `14` | Tweaked WHM |
| `15` | Tweaked WHM cross-term variant |
| `39` | Tweaked WHM with thin-web profiles |

These values are mapped to the internal `imead` variable in
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L654):

| `halofit_version` | `imead` | Internal meaning |
| ---: | ---: | --- |
| `11` | `10` | Baseline WHM |
| `13` | `11` | WHM cross version |
| `14` | `12` | Tweaked WHM |
| `15` | `13` | Tweaked cross version |
| `39` | `12` | Tweaked WHM with thin-web profiles |

Thus, `halofit_version=14` activates `imead=12`, not `imead=14`.

The `ibrieden` variable controls diagnostic components. `ibrieden=0`
selects the complete WHM expression; other values isolate halo, filament,
sheet, sphere, or cross terms. The mapping is in
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L693).

The profile selector `ivar_profile` chooses between the default
homogeneous cylinder/slab profiles and the concentration-dependent thin-web
profiles. This is configured at
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L709).

## 2. Overall calculation

The `halomod` routine computes the two terms and combines them:

```text
p1h   = p_1h(k, ...)
p2h   = p_2h(k, ...)
pfull = p2h + p1h       [for WHM]
```

This happens in [`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L1064).
The WHM result is therefore

\[
\Delta^2_{\rm WHM}(k)
=
\Delta^2_{\rm 2h}(k)
+
\Delta^2_{\rm 1h}(k).
\]

## 3. The two-halo term

For active WHM branches, `p_2h` is simply the linear power spectrum:

```fortran
p_2h = plin
```

See [`p_2h`](../WHM-CAMB/fortran/halofit.f90#L1990). Damping and
de-wiggling expressions remain as commented alternatives, but are not active
for WHM. Hence

\[
\Delta^2_{\rm 2h}(k)=\Delta^2_{\rm lin}(k).
\]

## 4. Web components and multiplicity functions

The WHM one-halo calculation starts in
[`p_1h`](../WHM-CAMB/fortran/halofit.f90#L2029). For each peak-height value
\(\nu\), it constructs multiplicity functions for three cosmic-web
components:

```fortran
g  = gnu(nu, 4, ...)
gf = gnu(nu, 2, ...)
gs = gnu(nu, 1, ...)
```

They represent halo, filament, and sheet populations respectively. The same
`gnu` family is used with different collapsed-dimension arguments; these
are not three unrelated empirical mass functions.

The halo term can instead use an alternative halo mass function through
`WHM_hmf`:

```text
WHM_hmf = 0   gnu-based default
WHM_hmf = 1   Warren et al. halo mass function
WHM_hmf = 2   Despali et al. halo mass function
```

The filament and sheet terms continue to use `gnu`. The resolved fractions
`sum_halo`, `sum_filament`, and `sum_sheet` are computed near
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2054).

## 5. Window functions

Each component has a Fourier-space window function.

For halos:

```fortran
wk   = win(...)
wk_2 = wk * wk
```

`win` is the truncated NFW-like halo window.

For the default filament and sheet profiles:

```fortran
wkf = win_alt(..., 2, ...)
wks = win_alt(..., 1, ...)
```

The code comments identify these as homogeneous cylinder and slab
approximations. Their squared and cross windows are constructed with
`win_alt_2w` and `win_alt_cross`.

For `ivar_profile=3`, the code instead calls `win_alt_nfw` and
`win_alt_2wc`. These thin-web profiles depend on the halo concentration
`c(M)`; see [`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2105).

## 6. Baseline WHM

For `imead=10` and `ibrieden=0`, the integrand is

```fortran
integrand = (g  * (wk_2  - wkf_2) &
           + gf * (wkf_2 - wks_2)) * m
```

Equivalently,

\[
I_{\rm baseline}
=
g\left(W_h^2-W_f^2\right)
+
g_f\left(W_f^2-W_s^2\right).
\]

The subtractions are intentional. WHM does not add three independent powers:

- the halo term occupies the part beyond the filament profile;
- the filament term occupies the part beyond the sheet profile;
- the baseline model has no explicit sheet term.

This branch is at
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2150).

## 7. Tweaked WHM

For `imead=12`, normally selected by `halofit_version=14`, the integrand
becomes

```fortran
integrand = (g  * (wk_2  - wkf_2) &
           + gf * (wkf_2 - wks_2) &
           + gs * (wks_2 - wks_2 * WindowTophat(k, rnl, 1.)**2)) * m
```

In equation form,

\[
I_{\rm tweaked}
=
g\left(W_h^2-W_f^2\right)
+
g_f\left(W_f^2-W_s^2\right)
+
g_sW_s^2\left[1-W_{\rm TH}^2(k,r_{\rm nl})\right].
\]

The sheet contribution is therefore suppressed through
\(1-W_{\rm TH}^2(k,r_{\rm nl})\), so sheets contribute only beyond the
nonlinear transition. This is implemented at
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2154).

## 8. Integration and dimensionless power

After integrating over the tabulated values of \(\nu\), the code divides by
the cosmic matter density and converts to dimensionless power:

\[
\Delta^2_{\rm 1h}(k)
=
\frac{k^3}{2\pi^2}
\frac{1}{\bar\rho}
\int d\nu\,m(\nu)I(\nu,k).
\]

The final conversion is performed by

```fortran
p_1h = sum * k**3 / (2.*pi**2)
```

near [`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2317). For WHM
branches with massive neutrinos, the integrand is additionally multiplied by
`(1-f_nu)**2`; see
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2209).

## 9. Low-mass completion

The numerical table may not contain all matter. The code therefore adds the
missing fractions at the lowest mass-bin profile:

```text
missing halo fraction     = 1 - sum_halo
missing filament fraction = 1 - sum_filament
missing sheet fraction    = 1 - sum_sheet
```

This makes the result less dependent on the lower mass limit of the input
table. The correction is implemented around
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2211).

## 10. Cross-term variants

The cross variants (`imead=11` and `imead=13`) calculate bias-weighted
quantities such as `sum_halo_cross`, `sum_filament_cross`,
`sum_sheet_cross`, `sum_ff`, and `sum_ss`.

Their final expression is nested:

```fortran
p_1h = sum*k**3/(2.*pi**2) &
     + ((plin*sum_sheet_cross**2 &
         + sum_ss*k**3/(2.*pi**2)) * sum_filament_cross**2 &
        + sum_ff*k**3/(2.*pi**2)) * sum_halo_cross**2
```

This represents sequential sheet-filament-halo embedding rather than the
simple additive full-WHM expression. It is implemented near
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2293).

## 11. Practical reading guide

For the normal tweaked WHM calculation:

```text
halofit_version = 14
        |
        v
imead = 12, ibrieden = 0
        |
        +--> p_2h = linear power
        |
        +--> p_1h = halo beyond filament
                    + filament beyond sheet
                    + sheet beyond nonlinear scale
        |
        v
pfull = p_2h + p_1h
```

The most important code locations are:

1. Selector mapping: [`halofit.f90#L654`](../WHM-CAMB/fortran/halofit.f90#L654)
2. Two-halo term: [`halofit.f90#L1990`](../WHM-CAMB/fortran/halofit.f90#L1990)
3. Multiplicity functions and profiles: [`halofit.f90#L2054`](../WHM-CAMB/fortran/halofit.f90#L2054)
4. Baseline and tweaked integrands: [`halofit.f90#L2150`](../WHM-CAMB/fortran/halofit.f90#L2150)
5. Integration and final power: [`halofit.f90#L2211`](../WHM-CAMB/fortran/halofit.f90#L2211)
6. Cross-term combination: [`halofit.f90#L2293`](../WHM-CAMB/fortran/halofit.f90#L2293)
