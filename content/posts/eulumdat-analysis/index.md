---
title: "Computing beam half-angles from EULUMDAT files with Python"
date: 2026-04-07
draft: false
tags: ["eulumdat", "ldt", "photometry", "beam angle", "half angle", "python", "pypi"]
description: "A practical guide to computing the half-angle at half maximum (HAHM) from EULUMDAT .ldt files using the eulumdat-analysis Python package — including multi-peak detection, ISYM handling, and FWHM."
math: true
---

Characterising the angular spread of a luminaire is one of the most
common tasks in photometric analysis. The standard metric is the
**half-angle at half maximum (HAHM)**: the angle at which the luminous
intensity drops to 50 % of its peak value. It is directly related to
the beam angle specified in product datasheets and is used for
luminaire classification, glare assessment, and lighting calculation
input.

[`eulumdat-analysis`](https://pypi.org/project/eulumdat-analysis/) is a
Python package that computes this and other practical photometric
quantities from EULUMDAT `.ldt` files. This article covers the `half_angle`
function, the first function of the package.

> For a field-by-field description of the EULUMDAT format and ISYM codes,
> see [The EULUMDAT file format — a complete technical reference](../eulumdat-format).

---

## What the function returns

For a given C-plane, `half_angle` returns the **absolute γ-angle** (measured
from nadir) where the intensity first drops to 50 % of the C-plane maximum,
searching from the peak angle γ_max toward 90°:

$$\text{half\_angle} = \gamma_{\text{root}} \quad \text{where} \quad I(\gamma_{\text{root}}) = \frac{I_{\text{max}}}{2}$$

The result is an **absolute angle**, not a distance from the peak. For a
Lambertian distribution (peak at γ = 0°), the half-angle is 60°. For an
off-axis luminaire with a peak at γ = 20° and a crossing at γ = 39°, the
half-angle is 39°, not 19°.

The measurement is **unilateral**: it looks only toward 90° from the peak,
not toward nadir. This is intentional — combining two complementary C-planes
gives the Full Width at Half Maximum (FWHM), as shown below.

---

## Installation

```bash
pip install eulumdat-analysis
```

---

## Quick start

```python
from pyldt import LdtReader
from ldt_analysis import half_angle

ldt = LdtReader.read("luminaire.ldt")

result = half_angle(ldt, [0.0, 90.0, 180.0, 270.0])
print(result)
# {0.0: 35.4, 90.0: 36.1, 180.0: 35.8, 270.0: 36.0}
```

`half_angle` accepts any list of C-plane angles and returns a dict mapping
each angle to its half-angle in degrees, or `None` if undefined.

---

## Algorithm

The function operates on the intensity profile $I(\gamma)$ of a single
C-plane, restricted to $\gamma \in [0°, 90°]$.

**Step 1 — Find γ_max**

The peak angle is the γ where $I(\gamma)$ is maximum. In case of ties,
the smallest γ is used (closest to nadir).

**Step 2 — Build the search domain**

Only the range $[\gamma_{\text{max}}, 90°]$ is considered. The intensity
never rises again toward nadir — that direction is excluded by definition.

**Step 3 — Locate the crossing**

A pair of consecutive discrete angles bracketing the half-maximum threshold
is identified. A `CubicSpline` is then fitted on the search domain, and
`scipy.optimize.brentq` finds the exact crossing within the bracket.

The combination of cubic interpolation and root-finding gives sub-degree
accuracy even on 5° angular grids, which are standard in EULUMDAT files.

---

## Multi-peak distributions

Some luminaires produce a bi-directional or bat-wing distribution — two
distinct lobes at different angles. For these, the concept of a single
half-angle is not meaningful.

`half_angle` detects multi-peak distributions automatically using a
**prominence criterion**: a local maximum (other than the global peak) is
considered a real secondary lobe if its prominence toward the global maximum
exceeds 5 % of $I_{\text{max}}$:

$$\text{prominence} = I_{\text{peak}} - \min\bigl(I(\gamma) \text{ on path to global max}\bigr)$$

If one or more significant secondary peaks exist, the function returns `None`.

This threshold correctly handles the two common failure modes:

- **Measurement noise** — tiny oscillations of 0.1–1 % prominence are ignored.
- **Real secondary lobes** — prominences of 10 % and above are correctly flagged.

---

## Uplight luminaires

A purely upward-emitting luminaire has zero intensity in the $[0°, 90°]$
hemisphere. `half_angle` returns `None` for all C-planes in this case — the
half-angle in the lower hemisphere is undefined.

---

## ISYM = 1: full rotational symmetry

For rotationally symmetric luminaires (`ldt.header.isym == 1`), only one
C-plane is stored in the file. `half_angle` handles this automatically:
any requested C-plane angle is resolved to the single available profile.

```python
ldt = LdtReader.read("symmetric.ldt")  # ISYM=1, one C-plane stored

result = half_angle(ldt, [0.0, 45.0, 90.0, 180.0])
# All four planes return the same value
# {0.0: 35.2, 45.0: 35.2, 90.0: 35.2, 180.0: 35.2}
```

---

## FWHM

`half_angle` is a unilateral measurement. To compute the Full Width at Half
Maximum (FWHM) of a beam, combine two complementary C-planes:

```python
ha_c0   = result[0.0]
ha_c180 = result[180.0]

if ha_c0 is not None and ha_c180 is not None:
    fwhm = ha_c0 + ha_c180
    print(f"FWHM (C0/C180) = {fwhm:.1f}°")
```

For a rotationally symmetric luminaire:

```python
ha = result[0.0]
if ha is not None:
    fwhm = 2 * ha
    print(f"FWHM = {fwhm:.1f}°")
```

---

## Return values

| Case | Return value |
| ---- | ------------ |
| Normal beam | `float` — crossing angle in degrees |
| C-plane not found in file (±0.01°) | `None` |
| `I_max = 0` (dark or inactive plane) | `None` |
| Intensity never drops to half-max within [γ_max, 90°] | `None` |
| Multi-peak distribution (prominence > 5 % of I_max) | `None` |

The function never raises an unhandled exception regardless of input quality.

---

## Cubic vs linear interpolation

EULUMDAT files typically store intensities at 5° angular steps. On a smooth,
single-lobe distribution, the difference between cubic spline and simple linear
interpolation between the two bracketing points is small — typically under 0.3°.
The cubic spline is preferred because it captures the curvature of the intensity
profile, giving more accurate results on sharply peaked distributions.

---

## Resources

- [`eulumdat-analysis` on PyPI](https://pypi.org/project/eulumdat-analysis/)
- [Source code on GitHub](https://github.com/123VincentB/eulumdat-analysis)
- [`eulumdat-py`](https://pypi.org/project/eulumdat-py/) — read/write EULUMDAT files
- [`eulumdat-symmetry`](https://pypi.org/project/eulumdat-symmetry/) — symmetrise EULUMDAT files
- [`eulumdat-plot`](https://pypi.org/project/eulumdat-plot/) — photometric polar diagrams
- [`eulumdat-luminance`](https://pypi.org/project/eulumdat-luminance/) — luminance tables
