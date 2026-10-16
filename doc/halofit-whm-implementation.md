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

```math
\Delta^2_{\rm WHM}(k)
=
\Delta^2_{\rm 2h}(k)
+
\Delta^2_{\rm 1h}(k).
```

## 3. The two-halo term

For active WHM branches, `p_2h` is simply the linear power spectrum:

```fortran
p_2h = plin
```

See [`p_2h`](../WHM-CAMB/fortran/halofit.f90#L1990). Damping and
de-wiggling expressions remain as commented alternatives, but are not active
for WHM. Hence

```math
\Delta^2_{\rm 2h}(k)=\Delta^2_{\rm lin}(k).
```

## 4. Web components and multiplicity functions

The WHM one-halo calculation starts in
[`p_1h`](../WHM-CAMB/fortran/halofit.f90#L2029). For each peak-height value
$\nu$, it constructs multiplicity functions for three cosmic-web
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

### Explicit window formulas

It is useful to identify the variables in `p_1h` as

```math
W_h(k,M) = wk,
\qquad
W_f(k,M) = wkf,
\qquad
W_s(k,M) = wks,
\qquad
W_{\rm TH}(k,M) = wkTH.
```

More precisely, in the WHM loop the arguments are

```math
W_h=\mathrm{win}\left(k\nu^{\eta},r_v,c\right),
\qquad
W_f=\mathrm{win\_alt}\left(k,r_v,r_r,2,\mathrm{ivar\_profile}\right),
```

```math
W_s=\mathrm{win\_alt}\left(k,r_v,r_r,1,\mathrm{ivar\_profile}\right),
\qquad
W_{\rm TH}=W_{\rm TH}(k,r_r;1).
```

Here `r_v=lut%rv` is the halo radius, `r_r=lut%rr` is the web-object
radius, `c=lut%c` is the halo concentration, and `eta` is the profile
scaling exponent returned by `this%eta`. The filament and sheet arguments use
`nu**0`, so their profile wavenumber is simply `k`.

The halo window is the normalized truncated NFW Fourier transform. Defining

```math
k_h=k\nu^{\eta},
\qquad
x = \frac{k_h r_v}{c},
\qquad
m(c)=\ln(1+c)-\frac{c}{1+c},
```

the code evaluates

```math
W_h(k,M)
=
\frac{
\cos(x)\left[\mathrm{Ci}((1+c)x)-\mathrm{Ci}(x)\right]
+\sin(x)\left[\mathrm{Si}((1+c)x)-\mathrm{Si}(x)\right]
-\frac{\sin(cx)}{(1+c)x}
}{m(c)}.
```

This is implemented by `winnfw`, called from `win`, in
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2944).

The spherical top-hat window used for the sphere term is

```math
W_{\rm TH}(k,R;\Delta_v)
=
\frac{3\Delta_v}{(kR)^3}
\left[
\sin\left(\frac{kR}{\Delta_v^{1/3}}\right)
-\frac{kR}{\Delta_v^{1/3}}
\cos\left(\frac{kR}{\Delta_v^{1/3}}\right)
\right].
```

For the ordinary sphere window, `Delta_v=1`, so this reduces to

```math
W_{\rm TH}(k,R)
=
3\frac{\sin(kR)-kR\cos(kR)}{(kR)^3}.
```

The implementation is [`WindowTophat`](../WHM-CAMB/fortran/halofit.f90#L3340).

For the default constant-density web profiles, define

```math
D_v=\left(\frac{r_r}{r_v}\right)^3,
\qquad
d_f=\sqrt{\frac{2\pi}{3}}\frac{r_r}{D_v^{1/3}},
\qquad
d_s=\frac{4}{3}\frac{r_r}{D_v^{1/3}}.
```

Then the filament and sheet windows used for the cross terms are

```math
W_f(k,M)
=
-\frac{3\sqrt{\pi}}{d_f^2 k^3\left(\sqrt{2}d_f-3\sqrt{\pi}r_r\right)}
\left[
-d_f k\sqrt{2\pi}\cos\left(\frac{d_f k}{\sqrt{2\pi}}\right)
+2\pi\sin\left(\frac{d_f k}{\sqrt{2\pi}}\right)
-d_f^2k^2\mathrm{Si}\left(\frac{d_f k}{\sqrt{2\pi}}\right)
+d_f^2k^2\mathrm{Si}(kr_r)
\right],
```

and

```math
W_s(k,M)
=
\frac{24\left[d_s k\cos(kr_r)-2\sin(d_s k/2)\right]}
{k^3\left(d_s^3-12d_s r_r^2\right)}.
```

These formulas are implemented by `WindowFilament` and `WindowSheet`,
selected by `win_alt`; see
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2958) and
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L3222).

### Important distinction: `wkf_2` and `wks_2`

In the WHM integrand, `wkf_2` and `wks_2` are not generally calculated as
`wkf**2` and `wks**2`. They are dedicated second-moment functions:

```math
W_f^{(2)}(k,M)
=
\frac{2\left[-1+\cos(2kr_r)+2kr_r\mathrm{Si}(2kr_r)\right]}
{(2kr_r)^2}
\left|W_{\rm TH}\left(k,\sqrt{\frac{5}{6}}d,1\right)\right|^2,
```

where `d=r_r/D_v^(1/3)` in `win_alt_2w`, and

```math
W_s^{(2)}(k,M)
=
\frac{2-2H(kr_r)}{(kr_r)^2}
\left|W_{\rm TH}\left(k,\frac{2\sqrt{5}}{3\sqrt{3}}d,1\right)\right|^2,
```

where the code defines

```math
H(q)=\mathrm{Re}\left[e^{-2iq}
{}_1F_1\left(\frac{3}{2};3;4iq\right)\right].
```

The corresponding routines are `WindowFilament_2f` and `WindowSheet_2s` in
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L3147). For thin-web profiles,
the analogous quantities are `WindowFilament_2fc` and `WindowSheet_2sc`, which
replace the top-hat transverse factor by a concentration-dependent NFW factor.

### Cross windows

The cross variants use dedicated filament/sphere and sheet/sphere windows. With
`q=kr_r`, the code implements

```math
W_{f\times h}(k,M)
=
\frac{3\left[\sin(q)-q\cos(q)\right]\mathrm{Si}(q)}{r_r^4 k^4}
W_{\rm TH}(k,d,1)^2,
```

and

```math
W_{s\times h}(k,M)
=
\frac{12r_rk\sin^2(q/2)\left[\sin(q)-q\cos(q)\right]}{q^6}
W_{\rm TH}(k,d,1)^2.
```

They are implemented by `WindowFilament_cross_Sphere` and
`WindowSheet_cross_Sphere` in
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L3186).

For `ivar_profile=3`, the unsquared `W_f` and `W_s` are instead the
concentration-dependent `WindowFilamentNFW` and `WindowSheetNFW` expressions,
selected by `win_alt_nfw`; their full formulas are in
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L3080).

## 6. Baseline WHM

For `imead=10` and `ibrieden=0`, the integrand is

```fortran
integrand = (g  * (wk_2  - wkf_2) &
           + gf * (wkf_2 - wks_2)) * m
```

Equivalently,

```math
I_{\rm baseline}
=
g\left(W_h^2-W_f^2\right)
+
g_f\left(W_f^2-W_s^2\right).
```

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

```math
I_{\rm tweaked}
=
g\left(W_h^2-W_f^2\right)
+
g_f\left(W_f^2-W_s^2\right)
+
g_sW_s^2\left[1-W_{\rm TH}^2(k,r_{\rm nl})\right].
```

The sheet contribution is therefore suppressed through
$1-W_{\rm TH}^2(k,r_{\rm nl})$, so sheets contribute only beyond the
nonlinear transition. This is implemented at
[`halofit.f90`](../WHM-CAMB/fortran/halofit.f90#L2154).

## 8. Integration and dimensionless power

After integrating over the tabulated values of $\nu$, the code divides by
the cosmic matter density and converts to dimensionless power:

```math
\Delta^2_{\rm 1h}(k)
=
\frac{k^3}{2\pi^2}
\frac{1}{\bar\rho}
\int d\nu\,m(\nu)I(\nu,k).
```

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
