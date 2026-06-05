---
title: "Rotating EULUMDAT photometric data azimutal with Python"
date: 2026-06-05
draft: false
tags: ["eulumdat", "ldt", "photometry", "rotation", "goniophotometer", "python", "pypi"]
description: "eulumdat-analysis v1.4.0 adds rotate — apply an azimuthal rotation to the intensity matrix of an EULUMDAT file to correct orientation or normalise C-plane alignment before archiving."
---

After a goniophotometric measurement, the photometric file may not be oriented
the way you need it. The C=0° direction defined in the file depends on how the
luminaire was mounted on the goniometer — and this may not match the reference
orientation required by a standard, a client, or your archiving convention.

[`eulumdat-analysis`](https://pypi.org/project/eulumdat-analysis/) v1.4.0 adds
`rotate` — a function that applies an azimuthal rotation of `alpha` degrees to
the intensity matrix of a `Ldt` object and returns a new corrected `Ldt`,
ready to be passed to the rest of the `eulumdat-*` ecosystem.

> For a field-by-field description of the EULUMDAT format and ISYM codes,
> see [The EULUMDAT file format — a complete technical reference](../eulumdat-format).
>
> For an introduction to the package and the `half_angle` function,
> see [Computing beam half-angles from EULUMDAT files with Python](../eulumdat-analysis).

---

## The C-plane convention in EULUMDAT

An EULUMDAT file stores the luminous intensity distribution as a matrix of
MC × NG values: MC azimuthal C-planes (0° to 360°) and NG polar γ-angles (0° to
180°). The C-plane labels in the header are fixed — they always run
0°, step°, 2·step°, … — but the physical direction they point to depends on
how the luminaire was mounted during measurement.

For a luminaire with a non-symmetric distribution — a single asymmetric
reflector, a linear LED strip, or a road lighting optic — the C=0° direction
in the photometric file should correspond to a clearly defined physical
reference: the longitudinal axis of a road, the main emission direction, or
the axis specified in the applicable standard (e.g. EN 13032-1 or CIE 121).

When this alignment is off, all downstream calculations (UGR tables, luminance
diagrams, polar plots) are computed against the wrong reference. `rotate`
corrects this directly in the intensity matrix.

---

## How the rotation works

A rotation by `alpha` degrees corresponds to a **circular shift** of the
C-plane profiles in the intensity matrix. The profile that was recorded at
C-plane $C_i$ is placed at C-plane $C_{i + \alpha}$:

$$I_{\text{rot}}[i] = I_{\text{orig}}\!\left[(i - n_{\text{shift}}) \bmod M_C\right]$$

where $n_{\text{shift}} = \operatorname{round}(\alpha / \Delta C) \bmod M_C$
and $\Delta C$ is the angular step between C-planes.

The C-plane labels in the header are not modified — they always remain
0°, $\Delta C$, $2\Delta C$, … The rotation only reorders the profiles that
are assigned to those labels.

---

## Installation

```bash
pip install eulumdat-analysis
```

---

## Quick start

```python
from pyldt import LdtReader, LdtWriter
from ldt_analysis import rotate

ldt = LdtReader.read("luminaire.ldt")

# Correct a 90° mounting offset
ldt_corrected = rotate(ldt, 90.0)

# Save the corrected file
LdtWriter.write(ldt_corrected, "luminaire_corrected.ldt")
```

---

## Valid rotation angles

`alpha` must be a **multiple of the angular step** $\Delta C$ stored in the
file. On a standard 15°×5° grid, valid rotations are …, −30°, −15°, 0°, 15°,
30°, 45°, 60°, 75°, 90°, …

```python
# Valid — 90° is a multiple of 15°
ldt_rot = rotate(ldt, 90.0)

# Valid — negative rotation supported
ldt_rot = rotate(ldt, -45.0)

# Valid — alpha=0° or alpha=360°: returns an identical copy
ldt_copy = rotate(ldt, 0.0)

# Invalid — raises ValueError
rotate(ldt, 7.0)
# ValueError: alpha=7.0° n'est pas un multiple du pas angulaire
# détecté (15.0°). Valeurs valides : …, -30, -15, 0, 15, 30, …
```

The tolerance for multiple-detection is ±1 × 10⁻⁹, so small floating-point
rounding errors in `alpha` are accepted.

---

## ISYM and the output file

`eulumdat-py` always expands the intensity matrix to the full MC × NG grid when
reading a file, regardless of the ISYM code stored in the header. `rotate`
therefore works on any input ISYM.

The output file always has **ISYM = 0** (full matrix), even if the source had a
symmetric ISYM. This is correct: after a rotation, the original symmetry is no
longer guaranteed to hold relative to the new C-plane orientation.

---

## Source object is never modified

`rotate` returns a new `Ldt` object. The original is never mutated:

```python
ldt_orig = LdtReader.read("luminaire.ldt")
ldt_rot  = rotate(ldt_orig, 90.0)

# ldt_orig.intensities is unchanged
assert ldt_orig.intensities[0] != ldt_rot.intensities[0]  # different profiles
```

---

## Practical use cases

**Correcting a mounting offset.** A luminaire measured with C=0° pointing
west instead of north needs a 90° correction before the file is delivered
to the client or archived:

```python
ldt_corrected = rotate(ldt_measured, 90.0)
LdtWriter.write(ldt_corrected, "luminaire_corrected.ldt")
```

**Normalising before further processing.** The C=0° direction should be the
main beam axis before computing a UGR catalogue, generating a polar diagram,
or calculating a luminance table:

```python
from ldt_analysis import rotate
from eulumdat_ugr import UgrCalculator

ldt_normalised = rotate(ldt_raw, offset_deg)
result = UgrCalculator.compute(ldt_normalised)
```

**Verifying rotational symmetry.** For a nominally symmetric luminaire, the
distributions at C=0° and C=90° should be identical after a 90° rotation:

```python
ldt_rot = rotate(ldt, 90.0)

# Compare C=0° profiles before and after rotation
original_c0 = ldt.intensities[0]
rotated_c0  = ldt_rot.intensities[0]

max_diff = max(abs(a - b) for a, b in zip(original_c0, rotated_c0))
print(f"Max deviation after 90° rotation: {max_diff:.3f} cd/klm")
```

---

## Return values and errors

| Case | Behaviour |
| ---- | --------- |
| `alpha` is a multiple of $\Delta C$ | Returns a new `Ldt` with ISYM=0 |
| `alpha = 0°` or `alpha = 360°` | Returns an identical copy |
| `alpha` not a multiple of $\Delta C$ | Raises `ValueError` with the detected step and example valid values |

---

## Resources

- [`eulumdat-analysis` on PyPI](https://pypi.org/project/eulumdat-analysis/)
- [Source code on GitHub](https://github.com/123VincentB/eulumdat-analysis)
- [Changelog v1.4.0](https://github.com/123VincentB/eulumdat-analysis/blob/main/CHANGELOG.md)
- [`eulumdat-py`](https://pypi.org/project/eulumdat-py/) — read/write EULUMDAT files
- [`eulumdat-symmetry`](https://pypi.org/project/eulumdat-symmetry/) — detect and apply ISYM symmetry
- [`eulumdat-plot`](https://pypi.org/project/eulumdat-plot/) — photometric polar diagrams
- [`eulumdat-ugr`](https://pypi.org/project/eulumdat-ugr/) — UGR catalogue (CIE 117/190)
