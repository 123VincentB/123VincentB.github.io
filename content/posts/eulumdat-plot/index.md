---
title: "Generating photometric polar diagrams from EULUMDAT files with Python"
date: 2026-03-18
draft: false
tags: ["eulumdat", "ldt", "photometry", "polar-diagram", "svg", "python", "pypi"]
description: "A practical guide to generating publication-ready Lumtopic-style photometric polar diagrams from EULUMDAT .ldt files using the eulumdat-plot Python package."
math: true
---

Every luminaire photometric file contains a candela distribution — a table of
intensities indexed by C-plane and γ-angle. Visualising that distribution as a
polar diagram is one of the most basic tasks in lighting documentation, yet doing
it correctly requires handling several non-trivial details: symmetry expansion,
CIE angular conventions, radial autoscaling, and consistent visual presentation.

[`eulumdat-plot`](https://pypi.org/project/eulumdat-plot/) is a Python package
that handles all of this and produces **publication-ready SVG polar diagrams** in
the style of the Lumtopic software — suitable for product datasheets, PDF
documentation, and online publication.

> This package is designed for **publication output**, not interactive exploration.
> For scientific plots with matplotlib (axis labels, legends, custom styling),
> see the [eulumdat-py polar diagram example](https://github.com/123VincentB/eulumdat-py/blob/main/examples/02_polar_diagram.md).

> For a field-by-field description of the EULUMDAT format and ISYM codes,
> see [The EULUMDAT file format — a complete technical reference](../eulumdat-format).

---

## What the diagram shows

The output is a square SVG image with:

- A **top banner**: "LED" on the left, an optional distribution code in the
  centre, "cd / klm" on the right.
- A **polar plot** showing the candela distribution in cd/klm:
  - C0/C180 as a **solid** curve
  - C90/C270 as a **dotted** curve
- A **dynamic radial scale**: 3–6 concentric circles with round values,
  automatically computed from the data maximum.
- **Scale labels** placed on the dominant hemisphere (upper or lower,
  detected automatically).

The CIE photometric convention is applied throughout: γ = 0° points toward
nadir (directly below the luminaire), γ = 180° toward zenith.

<img src="https://raw.githubusercontent.com/123VincentB/eulumdat-plot/main/docs/img/sample_01.png" width="300">

---

## Installation

```bash
pip install eulumdat-plot
```

With raster export (PNG / JPEG):

```bash
pip install "eulumdat-plot[export]"
```

With cubic spline interpolation:

```bash
pip install "eulumdat-plot[cubic]"
```

---

## Quick start

```python
from eulumdat_plot import plot_ldt

svg = plot_ldt("luminaire.ldt")
# → luminaire.svg  (next to the source file)

svg = plot_ldt("luminaire.ldt", code="D53")
# → luminaire.svg  with "D53" in the banner centre
```

That is the complete API for the common case. The SVG is written next to the
source file with the same base name.

---

## Architecture

The package is split into three modules with clear responsibilities:

**`plot.py`** — public entry point. Reads the LDT file via
[`eulumdat-py`](https://pypi.org/project/eulumdat-py/), extracts the four
principal C-planes by nearest-angle lookup, optionally resamples I(γ), converts
each plane to NAT coordinates, and calls the renderer.

**`renderer.py`** — pure SVG generation. Knows nothing about LDT files.
Receives lists of `(x, y)` points and produces an SVG via `svgwrite`.
Contains the `Layout` dataclass and the `make_svg()` function.

**`export.py`** — raster conversion. SVG → PNG or JPEG via
`vl-convert-python` (a compiled Rust/resvg renderer bundled in the wheel —
no native DLL required on any platform).

This separation means the renderer can be used independently of the LDT
pipeline, and the export layer can be omitted entirely if only SVG output
is needed.

---

## Symmetry handling

EULUMDAT files declare a symmetry code (`ISYM` 0–4) that determines how many
C-planes are actually stored. Rather than reimplementing symmetry expansion,
`eulumdat-plot` delegates entirely to
[`eulumdat-py`](https://pypi.org/project/eulumdat-py/): `LdtReader.read()`
always returns a **full** `[MC × NG]` intensity matrix regardless of ISYM.
Extracting C0, C90, C180, and C270 then reduces to a nearest-angle lookup
on `ldt.header.c_angles` — no ISYM-specific logic needed in the plot pipeline.

---

## CIE angular conventions

The polar diagram uses the NAT coordinate system (y pointing upward, θ = 0°
upward, clockwise positive). The CIE γ → NAT θ mapping is:

$$\theta_{right}(C_0, C_{90}) = 180° - \gamma$$
$$\theta_{left}(C_{180}, C_{270}) = 180° + \gamma$$

This places γ = 0° (nadir) at the bottom of the diagram and γ = 180° (zenith)
at the top, which is the standard orientation for photometric polar plots.

---

## Interpolation

EULUMDAT files typically store I(γ) at 5° steps. Resampling to 1° before
plotting produces visibly smoother curves:

```python
# Linear interpolation — default, always available
svg = plot_ldt("luminaire.ldt", interpolate=True, interp_method="linear")

# Cubic spline — smoother result, requires scipy
# pip install "eulumdat-plot[cubic]"
svg = plot_ldt("luminaire.ldt", interpolate=True, interp_method="cubic")
```

The cubic spline uses `scipy.interpolate.CubicSpline` with natural boundary
conditions. Output is clamped to `[min, max]` of the input to prevent
overshoot at the boundaries.

---

## Proportional scaling

All visual parameters — stroke widths, font sizes, margins, dash patterns —
are defined in the `Layout` dataclass and scale proportionally from the
1181 px reference via a single factory method:

```python
from eulumdat_plot import plot_ldt, Layout

# 600 × 600 px for a compact datasheet
svg = plot_ldt("luminaire.ldt", layout=Layout.for_size(600))

# 2362 × 2362 px for high-resolution print
svg = plot_ldt("luminaire.ldt", layout=Layout.for_size(2362))
```

`Layout.for_size(1181)` is identical to the default `Layout()`.
Individual parameters can be adjusted after construction:

```python
layout = Layout.for_size(800)
layout.stroke_curve_solid = 12.0
```

---

## Raster export

```python
from eulumdat_plot import plot_ldt, Layout
from eulumdat_plot.export import svg_to_png, svg_to_jpg

svg = plot_ldt("luminaire.ldt", layout=Layout.for_size(1181))

# Export at any size — independent of the SVG canvas size
for size in [300, 600, 1181]:
    svg_to_png(svg, f"diagram_{size}px.png", size_px=size)

svg_to_jpg(svg, "diagram.jpg", size_px=600, quality=95)
```

The export uses `vl-convert-python`, which bundles a compiled Rust SVG
renderer (resvg) inside the wheel. No Cairo, no GTK runtime, no system
library installation required on Windows, Linux, or macOS.

---

## Batch processing

```python
from pathlib import Path
from eulumdat_plot import plot_ldt, Layout
from eulumdat_plot.export import svg_to_png

input_dir  = Path("ldt_files")
output_dir = Path("output")
output_dir.mkdir(exist_ok=True)

layout = Layout.for_size(1181)

for ldt_file in sorted(input_dir.glob("*.ldt")):
    svg = plot_ldt(
        ldt_file,
        output_dir / ldt_file.with_suffix(".svg").name,
        layout=layout,
    )
    svg_to_png(svg, output_dir / ldt_file.with_suffix(".png").name, size_px=600)
    print(f"✓  {ldt_file.name}")
```

---

## Debug mode

The `debug=True` flag colour-codes each C-plane to validate the symmetry
expansion visually: C0 → red, C180 → blue, C90 → green, C270 → orange.
Useful when processing files from manufacturers whose ISYM declarations are
inconsistent.

```python
svg = plot_ldt("luminaire.ldt", debug=True)
```

---

## Resources

- [`eulumdat-plot` on PyPI](https://pypi.org/project/eulumdat-plot/)
- [Source code on GitHub](https://github.com/123VincentB/eulumdat-plot)
- [DOI: 10.5281/zenodo.19096110](https://doi.org/10.5281/zenodo.19096110)
- [`eulumdat-py`](https://pypi.org/project/eulumdat-py/) — read/write EULUMDAT files
- [`eulumdat-symmetry`](https://pypi.org/project/eulumdat-symmetry/) — symmetrise EULUMDAT files
