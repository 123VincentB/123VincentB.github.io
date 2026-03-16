---
title: "Symmetrising EULUMDAT photometric files with Python"
date: 2026-03-16
draft: false
tags: ["eulumdat", "ldt", "photometry", "symmetry", "python", "pypi"]
description: "A practical guide to symmetrising EULUMDAT .ldt photometric files using the eulumdat-symmetry Python package — including automatic ISYM detection."
math: true
---

EULUMDAT files store luminous intensity distributions as a matrix of C-planes
and γ-angles. When a luminaire has a symmetric distribution, the file can
declare this via the `ISYM` field — allowing lighting design software to
reconstruct the full distribution from a reduced set of C-planes.

In practice, raw goniophotometer measurements are never perfectly symmetric
due to measurement noise, lamp positioning tolerances, and minor luminaire
asymmetries. Symmetrising the data enforces the intended symmetry by
averaging mirror-plane pairs, producing cleaner files that import correctly
into DIALux, Relux, and similar tools.

This article covers the [`eulumdat-symmetry`](https://pypi.org/project/eulumdat-symmetry/)
Python package, which provides both deterministic symmetrisation and
automatic ISYM detection.

> For a field-by-field description of the EULUMDAT format and ISYM codes,
> see [The EULUMDAT file format — a complete technical reference](../eulumdat-format).

---

## ISYM modes

The EULUMDAT format defines five symmetry codes:

| ISYM | Description              | Averaging rule                             |
| ---- | ------------------------ | ------------------------------------------ |
| 0    | No symmetry              | None — raw values                          |
| 1    | Full rotational symmetry | All C-planes averaged to a single profile  |
| 2    | Symmetry about C0–C180   | $I_{sym}(C) = \dfrac{I(C) + I(360°-C)}{2}$ |
| 3    | Symmetry about C90–C270  | $I_{sym}(C) = \dfrac{I(C) + I(180°-C)}{2}$ |
| 4    | Quadrant symmetry        | Mean of all 4 quadrant mirror planes       |

Symmetrising a file sets its ISYM field to the chosen mode and replaces the
intensity matrix with the averaged values. The file becomes smaller and the
distribution becomes exactly symmetric by construction.

---

## Installation

```bash
pip install eulumdat-symmetry
```

Requires [`eulumdat-py`](https://pypi.org/project/eulumdat-py/) >= 1.0.0.

---

## Symmetrising with a known ISYM mode

When the symmetry of the luminaire is known in advance, use `LdtSymmetriser`
directly:

```python
from pyldt import LdtReader, LdtWriter
from ldt_symmetry import LdtSymmetriser

ldt = LdtReader.read("luminaire.ldt")
ldt_sym = LdtSymmetriser.symmetrise(ldt, isym=2)
LdtWriter.write(ldt_sym, "luminaire_ISYM2.ldt")
```

`symmetrise()` returns a new `Ldt` object — the original is never modified.

If the file already declares a non-zero ISYM (the distribution was already
symmetrised at measurement time), `symmetrise()` returns an unchanged copy
by default. Use `force=True` to re-symmetrise regardless:

```python
ldt_sym = LdtSymmetriser.symmetrise(ldt, isym=2, force=True)
```

---

## Automatic ISYM detection

When the symmetry mode is not known in advance, `LdtAutoDetector` analyses
the intensity distribution and returns the most plausible ISYM code:

```python
from pyldt import LdtReader, LdtWriter
from ldt_symmetry import LdtAutoDetector, LdtSymmetriser

ldt = LdtReader.read("luminaire.ldt")

detector = LdtAutoDetector()
isym = detector.detect(ldt)

if isym != 0:
    ldt_sym = LdtSymmetriser.symmetrise(ldt, isym=isym)
    LdtWriter.write(ldt_sym, f"luminaire_ISYM{isym}.ldt")
else:
    print("No plausible symmetry detected — file left unchanged.")
```

Detection returns `0` if no symmetric mode is plausible for the distribution.

### How detection works

Detection runs in two stages.

**Stage 1 — Shape analysis**

The intensity distribution $I(C, \gamma)$ is projected onto the floor
($\gamma < 90°$) and ceiling ($\gamma > 90°$) planes using the standard
point-source illuminance formula:

$$E = \frac{I \cdot \cos^3(\gamma)}{d^2}$$

This gives the shape of the illuminance footprint on the floor (or ceiling)
as seen from directly above. The shape of this footprint is characteristic
of the luminaire symmetry:

- A **circular** footprint → rotational symmetry (ISYM 1)
- A square, rectangular, or elongated **centred** footprint → quadrant symmetry (ISYM 4)
- A footprint **offset along a single axis** (C0 or C90) → single mirror plane (ISYM 2 or 3)
- A footprint **offset along both axes** → no plausible symmetry (ISYM 0)

The floor and ceiling components are evaluated independently (a luminaire
may emit only downward, only upward, or both), then combined.

**Stage 2 — Score veto**

The shape stage proposes a candidate mode. A score veto then measures the
actual asymmetry of the distribution for that candidate using a relative RMS:

$$\text{score} = \frac{\text{RMS}(\text{diff})}{\text{RMS}(\text{signal})}$$

where diff is the element-wise difference between mirror-plane pairs and
signal is their mean absolute value. A score of 0 means perfect symmetry.
If the score exceeds the threshold (default: 0.23), the candidate is
rejected and ISYM 0 is returned.

Both stages must agree for a mode to be accepted. This two-stage approach
avoids false positives: a luminaire whose footprint *looks* symmetric but
whose distribution is actually irregular will be correctly classified as
ISYM 0.

### Diagnostic output

`return_diag=True` exposes the internal metrics — useful for understanding
why a particular mode was accepted or rejected:

```python
isym, diag = detector.detect(ldt, return_diag=True)

print(f"Detected ISYM  : {diag['isym_detected']}")
print(f"Candidate ISYM : {diag['isym_candidate']}  (shape stage)")
print(f"Accepted       : {diag['accepted']}")
print(f"Scores         : {diag['scores']}")
# {'scores': {1: 0.12, 2: 0.04, 3: 0.38, 4: 0.38}}
# score_threshold = 0.23 → ISYM 2 accepted, ISYM 3 and 4 vetoed
```

---

## Batch processing

Processing a folder of files in one pass:

```python
from pathlib import Path
from pyldt import LdtReader, LdtWriter
from ldt_symmetry import LdtAutoDetector, LdtSymmetriser

detector = LdtAutoDetector()

for path in Path("ldt_files").glob("*.ldt"):
    ldt = LdtReader.read(path)
    isym = detector.detect(ldt)
    if isym != 0:
        ldt_sym = LdtSymmetriser.symmetrise(ldt, isym=isym)
        out = path.with_stem(path.stem + f"_ISYM{isym}")
        LdtWriter.write(ldt_sym, out)
        print(f"{path.name} → ISYM {isym} → {out.name}")
    else:
        print(f"{path.name} → no symmetry detected")
```

---

## Resources

- [`eulumdat-symmetry` on PyPI](https://pypi.org/project/eulumdat-symmetry/)
- [Source code on GitHub](https://github.com/123VincentB/ldt_symmetry)
- [DOI: 10.5281/zenodo.19047884](https://doi.org/10.5281/zenodo.19047884)
- [`eulumdat-py`](https://pypi.org/project/eulumdat-py/) — read/write EULUMDAT files
