---
title: "Computing luminance tables and polar diagrams from EULUMDAT files with Python"
date: 2026-03-27
draft: false
tags: ["eulumdat", "ldt", "photometry", "luminance", "ugr", "polar-diagram", "svg", "python", "pypi"]
description: "A practical guide to computing luminance tables (cd/m²) and generating print-ready polar luminance diagrams from EULUMDAT .ldt files using the eulumdat-luminance Python package."
math: true
---

Luminous intensity tells you how much light a luminaire emits in each direction.
Luminance tells you how *bright* it appears from that direction — the quantity
directly responsible for glare. Converting from intensity to luminance requires
knowing the projected luminous area of the luminaire as seen from each C-plane
and γ-angle, which depends on the physical geometry declared in the EULUMDAT
header.

[`eulumdat-luminance`](https://pypi.org/project/eulumdat-luminance/) is a Python
package that computes luminance tables (cd/m²) from EULUMDAT files, generates
**polar luminance diagrams** showing all 24 C-planes simultaneously, and provides
**arbitrary-angle interpolation** for downstream UGR calculations.

> This package is the prerequisite for `eulumdat-ugr` (UGR per CIE 117/CIE 190),
> currently in development. `LuminanceResult.at()` is the interface that bridges
> the two.

> For a field-by-field description of the EULUMDAT format and ISYM codes,
> see [The EULUMDAT file format — a complete technical reference](../eulumdat-format).

---

## Installation

```bash
pip install eulumdat-luminance
```

Dependencies: [`eulumdat-py`](https://pypi.org/project/eulumdat-py/), numpy,
scipy, vl-convert-python (SVG rasterisation), Pillow (JPG conversion).

---

## Quick start

```python
from pyldt import LdtReader
from eulumdat_luminance import LuminanceCalculator, LuminancePlot, PolarStyle

ldt    = LdtReader.read("luminaire.ldt")
result = LuminanceCalculator.compute(ldt)

print(f"{result.luminaire_name} — {result.maximum:.0f} cd/m²")

plot = LuminancePlot(result)
plot.polar("polar.svg")
plot.polar("polar.png")

# Print-ready: 10 cm at 150 dpi, fonts equivalent to Arial 9pt
plot.polar("polar_report.png",
           style=PolarStyle.for_print(width_cm=10, dpi=150, font_scale=2.11))
```

<img src="https://raw.githubusercontent.com/123VincentB/eulumdat-luminance/main/examples/polar_sample04_word.png" width="400">

*Sample 04 — linear luminaire 1480 × 63 mm, 12 334 lm —
`PolarStyle.for_print(width_cm=10, dpi=150, font_scale=2.11)`*

---

## Physical model

### The luminance formula

Luminance is intensity divided by the projected luminous area as seen from the
direction $(C, \gamma)$:

$$L(C, \gamma) = \frac{I(C, \gamma)}{A_\text{proj}(C, \gamma)}$$

where $I$ is in cd (not cd/klm — the package applies the lamp flux
automatically from the EULUMDAT header).

### Projected area

The luminous area of a luminaire has a bottom face (the emitting surface) and
lateral faces (the luminaire housing depth). Their combined projection onto a
plane perpendicular to the observation direction is:

$$A_\text{proj}(C, \gamma) = A_\text{bottom} \cdot \cos(\gamma) + A_\text{side}(C) \cdot \sin(\gamma)$$

For **rectangular** luminaires, $A_\text{bottom} = \ell \times w$.
For **circular** luminaires (`width_lum_area = 0`), $A_\text{bottom} = \pi \cdot ({\ell}/{2})^2$.

The lateral contribution $A_\text{side}(C)$ combines the projected housing
heights along the C-plane direction. The four header fields
`h_lum_c0`, `h_lum_c90`, `h_lum_c180`, `h_lum_c270` (in mm) map to the four
quadrants as follows:

| Quadrant | C range   | Long axis height | Short axis height |
|----------|-----------|-----------------|------------------|
| Q0       | 0°–90°    | h_c0            | h_c90            |
| Q1       | 90°–180°  | h_c180          | h_c90            |
| Q2       | 180°–270° | h_c180          | h_c270           |
| Q3       | 270°–360° | h_c0            | h_c270           |

For flush-mounted or recessed luminaires where all heights are zero,
$A_\text{side} = 0$ throughout — no special handling is needed because
$\gamma = 90°$ falls outside the UGR grid (max 85°).

### Flux and multi-set LDT files

Some EULUMDAT files declare multiple lamp sets (alternative configurations).
Only the **first set** is used for the calculation, consistent with the
behaviour of Relux and DIALux:

$$I = I_\text{cd/klm} \times \frac{n_\text{lamps}[0] \times \Phi_\text{lamp}[0]}{1000}$$

### `full` mode

By default, `LuminanceCalculator.compute()` returns a UGR grid:

- **C-axis**: 0°, 15°, 30°, …, 345° (24 planes)
- **γ-axis**: 65°, 70°, 75°, 80°, 85° (5 angles)

This is the grid required by CIE 117 / CIE 190 for UGR calculation. When the
native LDT resolution does not match (e.g. files stored at 2.5° steps),
bilinear interpolation is applied automatically.

For detailed photometric analysis, `full=True` returns the native LDT grid
without any resampling:

```python
result_ugr  = LuminanceCalculator.compute(ldt)              # UGR grid
result_full = LuminanceCalculator.compute(ldt, full=True)   # native grid
```

---

## The polar luminance diagram

The luminance polar diagram differs fundamentally from the intensity polar
diagram produced by [`eulumdat-plot`](../eulumdat-plot):

| | `eulumdat-plot` | `eulumdat-luminance` |
|---|---|---|
| Quantity | cd/klm (intensity) | cd/m² (luminance) |
| C-planes shown | C0/C90/C180/C270 only | **All 24 C-planes** |
| γ-angles shown | Full range (0°–180°) | UGR range (65°–85°) |
| Colour encoding | Solid / dotted lines | **Blue gradient by γ** |
| Threshold marker | None | Red dashed circle at 3 000 cd/m² |

All 24 C-planes are drawn as curves, colour-coded from dark blue (γ = 65°,
closest to horizontal) to light blue (γ = 85°, most grazing). The colour
gradient makes it immediately apparent whether glare is concentrated in
specific C-planes or distributed uniformly around the luminaire.

The red dashed circle at 3 000 cd/m² marks the UGR-sensitive threshold
(above this value a luminaire contributes measurably to glare). The threshold
can be adjusted or disabled via `PolarStyle`:

```python
from eulumdat_luminance import LuminancePlot, PolarStyle

plot = LuminancePlot(result)

# Disable threshold circle
style = PolarStyle(threshold=None)
plot.polar("polar.svg", style=style)

# Custom threshold
style = PolarStyle(threshold=5000)
plot.polar("polar.svg", style=style)
```

### Angular convention

The diagram uses the standard trigonometric convention:
C = 0° to the right, C = 90° upward, anticlockwise positive. This is distinct
from the CIE photometric nadir-down convention used in `eulumdat-plot` — the
luminance diagram is oriented around the C-plane system, not the γ-angle system.

---

## Print-ready output

`PolarStyle.for_print()` computes all internal dimensions from a physical
target size, ensuring the output is exactly the right size for insertion into
a PDF or Word document:

```python
# 10 cm wide at 150 dpi — fonts equivalent to Arial 9pt
style = PolarStyle.for_print(width_cm=10, dpi=150, font_scale=2.11)

# 8 cm wide at 300 dpi — fonts equivalent to Arial 10pt
style = PolarStyle.for_print(width_cm=8, dpi=300, font_scale=2.6)

plot.polar("report_figure.png", style=style)
```

The `font_scale` parameter scales all font sizes and text reservation zones
proportionally without affecting the diagram radius or stroke widths. The
formula to match a specific font size is:

$$\text{font\_scale} = \frac{p \cdot \text{dpi}}{72 \cdot 10 \cdot k}$$

where $p$ is the target font size in points and $k$ is the ratio of the
target canvas width to the 665 px reference canvas.

Reference values at **150 dpi**:

| `width_cm` | Output pixels | `diagram_r` |
|---|---|---|
| 8  | ~472 px | 177 |
| 10 | ~591 px | 222 |
| 12 | ~709 px | 266 |

Reference values at **300 dpi**:

| `width_cm` | Output pixels | `diagram_r` |
|---|---|---|
| 6  | ~709 px  | 266 |
| 8  | ~945 px  | 355 |
| 12 | ~1417 px | 533 |

Note: when `font_scale > 1`, the canvas becomes slightly wider than
`width_cm` because the text reservation zones expand. The diagram itself
(radius, strokes, legend bar) is unaffected.

### SVG output

The SVG is generated in pure SVG — no Vega-Lite dependency. The internal
layout uses named `<g>` layers (`grid`, `curves`, `ring-labels`,
`angle-labels`, `title`, `legend`) each in its own coordinate system centred
on the diagram origin. The `grid` and `curves` layers are clipped to the
diagram circle; text layers are not clipped and can extend into the padding
zone. `vl-convert-python` is used only for SVG → PNG rasterisation.

---

## Arbitrary-angle interpolation

`LuminanceResult.at()` evaluates the luminance at any $(C, \gamma)$ pair via
bilinear interpolation — the interface needed by `eulumdat-ugr`:

```python
# Scalar input → float
lum = result.at(c_deg=12.0, g_deg=67.0)

# Array input → np.ndarray (vectorised, same shapes required)
import numpy as np
lums = result.at(
    c_deg=np.array([0.0, 12.0, 90.0]),
    g_deg=np.array([65.0, 67.0, 75.0]),
)
```

The interpolator uses `scipy.interpolate.RegularGridInterpolator` (bilinear
method) and is built on first use then cached. The C-axis is extended to 360°
before construction (copying the C = 0° profile) so that interpolation
between C = 345° and C = 360° works correctly.

For the best interpolation accuracy, use the native LDT grid:

```python
result = LuminanceCalculator.compute(ldt, full=True)
lum = result.at(c_deg=12.0, g_deg=67.0)
```

---

## CSV and JSON export

```python
result.to_csv("luminance.csv")
result.to_json("luminance.json")
```

Both exports include the full table, the C-axis and γ-axis values, the
maximum luminance, and the luminaire name.

---

## Validation

The package was validated against **Relux Desktop** on 10 representative LDT
samples covering all ISYM symmetry modes (0–4), rectangular and circular
luminaire shapes, files with 5° and 2.5° angular resolution, and luminaires
with non-zero housing heights. All 10 samples pass within the tolerance band
of 3% for values above 10 cd/m² and ±10 cd/m² for lower values. Typical
discrepancies are below 0.3%; the largest (sample 03, circular luminaire) is
2.5% and is explained by rounding in the Relux reference report.

The polar diagrams were validated visually for all 10 samples against the
expected output style.

---

## Resources

- [`eulumdat-luminance` on PyPI](https://pypi.org/project/eulumdat-luminance/)
- [Source code on GitHub](https://github.com/123VincentB/eulumdat-luminance)
- [DOI: 10.5281/zenodo.19223062](https://doi.org/10.5281/zenodo.19223062)
- [`eulumdat-py`](https://pypi.org/project/eulumdat-py/) — read/write EULUMDAT files
- [`eulumdat-plot`](https://pypi.org/project/eulumdat-plot/) — intensity polar diagrams
- [`eulumdat-symmetry`](https://pypi.org/project/eulumdat-symmetry/) — symmetrise EULUMDAT files
