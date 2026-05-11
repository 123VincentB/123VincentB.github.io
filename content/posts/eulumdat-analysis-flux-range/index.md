---
title: "Integrating luminous flux over any angular cone from EULUMDAT files with Python"
date: 2026-05-11
draft: false
tags: ["eulumdat", "ldt", "photometry", "flux", "beam angle", "python", "pypi"]
description: "eulumdat-analysis v1.3.0 adds luminous_flux_range — integrate the luminous flux over any gamma window [g_min, g_max] from an EULUMDAT file, with exact handling of partial zones."
math: true
---

Measuring the total luminous flux of a luminaire is straightforward. But for
practical photometric analysis, you often need the flux emitted within a
specific angular zone: the main beam of a downlight, the UGR-relevant upper
hemisphere of an indirect luminaire, or any arbitrary cone that a standard
characterises.

[`eulumdat-analysis`](https://pypi.org/project/eulumdat-analysis/) v1.3.0 adds
`luminous_flux_range` — a function that integrates the luminous flux over any
gamma window $[\gamma_{\min}, \gamma_{\max}]$ using the same CIE 190
trapezoidal method as the existing `luminous_flux` function, with correct
handling of partial angular zones.

> For a field-by-field description of the EULUMDAT format and ISYM codes,
> see [The EULUMDAT file format — a complete technical reference](../eulumdat-format).
>
> For the beam half-angle function and an introduction to the package,
> see [Computing beam half-angles from EULUMDAT files with Python](../eulumdat-analysis).

---

## The CIE 190 trapezoidal integral

`luminous_flux` and `luminous_flux_range` both implement the trapezoidal
approximation defined in CIE 190:2010. The intensity matrix is first averaged
across all C-planes to obtain a mean intensity $\bar{I}(\gamma)$ for each
γ-angle. The flux is then summed over consecutive γ-zones:

$$\phi = \sum_{k} \bar{I}_k \cdot 2\pi \left| \cos\gamma_k - \cos\gamma_{k+1} \right|$$

where $\bar{I}_k = \frac{\bar{I}(\gamma_k) + \bar{I}(\gamma_{k+1})}{2}$ is the
trapezoid mean and $2\pi|\cos\gamma_k - \cos\gamma_{k+1}|$ is the exact solid
angle of the zone in steradians. Intensities are in cd/klm, so the result is
in lm/klm — normalised to the lamp flux declared in the file header.

`luminous_flux_range` applies this formula only to zones that overlap
$[\gamma_{\min}, \gamma_{\max}]$.

---

## Installation

```bash
pip install eulumdat-analysis
```

---

## Quick start

```python
from pyldt import LdtReader
from ldt_analysis import luminous_flux, luminous_flux_range, dff_computed

ldt = LdtReader.read("luminaire.ldt")

# Total flux (full sphere)
phi_total = luminous_flux(ldt)                   # e.g. 943.7 lm/klm

# Flux within the 0°–60° cone (main beam of a downlight)
phi_cone = luminous_flux_range(ldt, 0.0, 60.0)  # e.g. 756.2 lm/klm

# Fraction of total flux captured by the cone
efficiency = phi_cone / phi_total * 100          # e.g. 80.1 %

# Lower hemisphere — equivalent to dff_computed but in lm/klm
phi_down = luminous_flux_range(ldt, 0.0, 90.0)

# Any arbitrary zone
phi_zone = luminous_flux_range(ldt, 30.0, 75.0)
```

---

## Handling partial angular zones

Most EULUMDAT files store intensities at 5° steps. The requested boundaries
$\gamma_{\min}$ and $\gamma_{\max}$ will not always fall exactly on a grid
angle. `luminous_flux_range` handles this by splitting the overlapping zone at
the boundary angle.

For a zone $[\gamma_k, \gamma_{k+1}]$ where $\gamma_k < \gamma_{\max} < \gamma_{k+1}$,
the intensity at the boundary is estimated by linear interpolation:

$$\bar{I}(\gamma_{\max}) = \bar{I}(\gamma_k) + \bigl(\bar{I}(\gamma_{k+1}) - \bar{I}(\gamma_k)\bigr) \cdot \frac{\gamma_{\max} - \gamma_k}{\gamma_{k+1} - \gamma_k}$$

The trapezoid mean for the partial zone is then
$\frac{\bar{I}(\gamma_k) + \bar{I}(\gamma_{\max})}{2}$, and the solid angle is
computed exactly over $[\gamma_k, \gamma_{\max}]$. The same applies at the
lower boundary $\gamma_{\min}$.

This ensures that `luminous_flux_range(ldt, 0.0, 180.0)` is always
**exactly equal** to `luminous_flux(ldt)`, and that the additivity property
holds to second-order accuracy:

$$\phi[0°, 90°] \approx \phi[0°, 45°] + \phi[45°, 90°]$$

(A small second-order interpolation error may appear when 45° is not a grid
angle, which is why the additivity tolerance is 1e-4 rather than 1e-6.)

---

## Relationship to `dff_computed`

The downward flux fraction (DFF) was already computed by `dff_computed` in
v1.2.0. In v1.3.0, `dff_computed` is refactored to delegate to
`luminous_flux_range`:

```python
def dff_computed(ldt):
    phi_total = luminous_flux(ldt)
    phi_down = luminous_flux_range(ldt, 0.0, 90.0)
    return phi_down / phi_total * 100.0
```

This is a pure refactoring — numerically identical on all EULUMDAT files where
90° is an angular grid point, which is the case for all standard files (5°,
2.5°, and 1° grids).

`luminous_flux_range(ldt, 0.0, 90.0)` gives the same DFF numerator as
`dff_computed`, but **in lm/klm** rather than as a percentage — useful when
you need the absolute downward flux rather than the fraction.

---

## Return values

| Case | Return value |
| ---- | ------------ |
| Window overlaps the gamma grid | `float` — flux in lm/klm over $[\gamma_{\min}, \gamma_{\max}]$ |
| Window does not overlap the gamma grid | `0.0` — not an error |
| $\gamma_{\min} \geq \gamma_{\max}$ | `None` |
| $\gamma_{\min} < 0°$ or $\gamma_{\max} > 180°$ | `None` |
| Intensity matrix or gamma grid empty | `None` |

The function never raises an unhandled exception regardless of input quality.

---

## Practical use cases

**Beam cone efficiency.** For a downlight with a narrow beam, the fraction of
flux within 0°–60° (the typical useful cone for task lighting) is a standard
performance indicator:

```python
phi_total = luminous_flux(ldt)
phi_60    = luminous_flux_range(ldt, 0.0, 60.0)
cone_eff  = phi_60 / phi_total * 100
print(f"Cone efficiency (0–60°): {cone_eff:.1f} %")
```

**UGR-relevant upward flux.** For indirect or semi-indirect luminaires, the
flux above 90° contributes to the unified glare rating. It can be isolated
as the complement of the downward flux:

```python
phi_up = luminous_flux_range(ldt, 90.0, 180.0)
```

**Zonal flux breakdown.** Decompose the total output into standard photometric
zones — useful for luminaire classification and reporting:

```python
zones = [(0, 30), (30, 60), (60, 90), (90, 120), (120, 150), (150, 180)]
for g_min, g_max in zones:
    phi = luminous_flux_range(ldt, float(g_min), float(g_max))
    print(f"  γ {g_min:3d}°–{g_max:3d}°: {phi:7.2f} lm/klm")
```

---

## Resources

- [`eulumdat-analysis` on PyPI](https://pypi.org/project/eulumdat-analysis/)
- [Source code on GitHub](https://github.com/123VincentB/eulumdat-analysis)
- [Changelog v1.3.0](https://github.com/123VincentB/eulumdat-analysis/blob/main/CHANGELOG.md)
- [`eulumdat-py`](https://pypi.org/project/eulumdat-py/) — read/write EULUMDAT files
- [`eulumdat-plot`](https://pypi.org/project/eulumdat-plot/) — photometric polar diagrams
- [`eulumdat-luminance`](https://pypi.org/project/eulumdat-luminance/) — luminance tables
- [`eulumdat-ugr`](https://pypi.org/project/eulumdat-ugr/) — UGR catalogue (CIE 117/190)
