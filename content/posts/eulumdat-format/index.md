---
title: "The EULUMDAT file format — a complete technical reference"
date: 2026-02-25
draft: false
tags: ["eulumdat", "ldt", "photometry", "file-format", "python"]
description: "A field-by-field description of the EULUMDAT .ldt photometric file format, including symmetry types, lamp sets, intensity data and real-world variations."
---

EULUMDAT is the standard file format for luminaire photometric data in Europe.
Despite being widely used — every goniophotometer lab produces `.ldt` files,
every lighting simulation tool (DIALux, Relux, AGi32) consumes them — the format
is remarkably poorly documented in English. This article is an attempt to fix that.

## Background

The format was defined by the **LiTG** (Deutsche Lichttechnische Gesellschaft) in
1990. It predates XML, JSON and every modern data format. It is a plain-text,
positional file: each piece of data occupies a specific line number, and a single
missing or empty line shifts everything that follows, producing a corrupt file.

Despite its age, EULUMDAT remains dominant in Europe. IES files (IESNA LM-63)
cover the same purpose and are standard in North America, but most European
manufacturers distribute `.ldt` files.

## File structure

A EULUMDAT file is a sequence of lines. There are no section markers, no
delimiters other than newlines. The parser must count lines.

The first 26 lines (plus 6 lines per lamp set) form the **header**. The rest
contains the photometric data.

### Header fields (lines 1–26)

| Line | Field | Type | Description |
|------|-------|------|-------------|
| 1 | `COMPANY` | string | Manufacturer name |
| 2 | `ITYP` | int | Luminaire type (0=point, 1=linear, 2=area) |
| 3 | `ISYM` | int | Symmetry code (see below) |
| 4 | `MC` | int | Number of C-planes |
| 5 | `DC` | float | Angular step between C-planes (°) |
| 6 | `NG` | int | Number of γ-angles per C-plane |
| 7 | `DG` | float | Angular step between γ-angles (°) |
| 8 | `REPORT_NO` | string | Measurement report number |
| 9 | `LUMINAIRE_NAME` | string | Luminaire description |
| 10 | `LUMINAIRE_NO` | string | Luminaire catalogue number (may be empty) |
| 11 | `FILE_NAME` | string | Original filename |
| 12 | `DATE_USER` | string | Date and operator |
| 13 | `LENGTH` | float | Luminaire length (mm) |
| 14 | `WIDTH` | float | Luminaire width (mm) |
| 15 | `HEIGHT` | float | Luminaire height (mm) |
| 16 | `LENGTH_LUM` | float | Luminous area length (mm) |
| 17 | `WIDTH_LUM` | float | Luminous area width (mm) |
| 18 | `H_C0` | float | Height of luminous area at C=0° (mm) |
| 19 | `H_C90` | float | Height of luminous area at C=90° (mm) |
| 20 | `H_C180` | float | Height of luminous area at C=180° (mm) |
| 21 | `H_C270` | float | Height of luminous area at C=270° (mm) |
| 22 | `DFF` | float | Downward flux fraction (%) |
| 23 | `LORL` | float | Light output ratio luminaire (%) |
| 24 | `CONV_FACTOR` | float | Conversion factor |
| 25 | `TILT` | float | Tilt angle (°) |
| 26 | `N_SETS` | int | Number of lamp sets |
| 27 | `NUM_LAMPS` | int | Number of lamps — set 1 |
| 28 | `LAMP_TYPE` | string | Lamp type description — set 1 |
| 29 | `LAMP_FLUX` | float | Total luminous flux (lm) — set 1 |
| 30 | `CCT` | string | Colour temperature — set 1 |
| 31 | `CRI` | string | Colour rendering index — set 1 |
| 32 | `LAMP_WATT` | float | Power (W) — set 1 |
| … | *(repeat 6 lines per additional set)* | | |

### Lamp sets (6 lines per set)

Starting at line 27, each lamp set occupies exactly **6 lines**, one value per line.
With N sets, lamp data spans lines 27 to 26+6N. Direct ratios start at line 27+6N.
```
NUM_LAMPS      ← number of lamps (integer; -1 = lamp not specified)
LAMP_TYPE      ← free text, e.g. "LED" or "1 x T5 35W"
LAMP_FLUX      ← rated luminous flux (lm)
CCT            ← correlated colour temperature, e.g. "4000K"
CRI            ← colour rendering index, e.g. "80"
LAMP_WATT      ← rated power (W)
```

Most files contain a single lamp set (`N_SETS = 1`). Files with multiple sets
describe luminaires with interchangeable lamp configurations.

> ⚠️ **Common mistake**: some early parsers read all 6 values from a single line.
> This is wrong. Each value occupies its own line. The single-line interpretation
> produces silently incorrect results on any file where `LAMP_TYPE` contains spaces.

### Angular data and intensity values

After the lamp sets come:

- **10 direct ratio values** (one per line or space-separated)
- **MC C-plane angles** (degrees)
- **NG γ-angles** (degrees)
- **Intensity values** in cd/klm, organised by C-plane then γ-angle

The intensity unit is **cd/klm** (candela per kilolumen of lamp flux), not absolute
candela. To convert:
```
I [cd] = I [cd/klm] × Φ_lamp [lm] / 1000
```

## Symmetry (ISYM)

The most subtle part of the format. ISYM allows the file to store only a subset
of C-planes, reducing file size and smoothing the distribution.

| ISYM | Description | Planes stored | Averaging rule |
|------|-------------|---------------|----------------|
| 0 | No symmetry | All MC planes | None — raw values |
| 1 | Full rotational | 1 plane (C=0°) | Mean of all planes |
| 2 | About C0–C180 | MC/2 + 1 planes | I_sym = [I(C) + I(360°−C)] / 2 |
| 3 | About C90–C270 | MC/2 + 1 planes | I_sym = [I(C) + I(180°−C)] / 2 |
| 4 | Quadrant | MC/4 + 1 planes | Mean of 4 symmetric planes |

**ISYM=3 storage order**: planes are stored from C=270° down to C=90°
(decreasing order). This is the opposite of what most people expect, and several
parsers get it wrong. DIALux and Relux require this exact order to import the file
correctly.

The values stored in the file are **already averaged** for ISYM 1–4. ISYM=0 is
the only case that contains raw, unaveraged measurement data.

## Real-world variations

Testing against files from 10 manufacturers revealed several deviations from the
spec that a robust parser must handle:

- **Empty `LUMINAIRE_NO`** (line 10): perfectly valid, must not be skipped
- **`NUM_LAMPS = -1`**: used by some manufacturers (e.g. ERCO) to indicate that
  the lamp is not specified — common for LED modules
- **BOM UTF-8** prefix: some files (ERCO) include a UTF-8 byte order mark
  (`0xEF 0xBB 0xBF`) despite the format being ISO-8859-1. Strip before decoding.
- **`DG = 0`**: seen in Signify/Philips files — the angular step is absent and
  must be inferred from the data

## Python implementation

[pyldt](https://github.com/123VincentB/pyldt) is an open-source Python library
that handles all of the above, available on PyPI as
[`eulumdat-py`](https://pypi.org/project/eulumdat-py/):
```bash
pip install eulumdat-py
```
```python
from pyldt import LdtReader

ldt = LdtReader.read("luminaire.ldt")
print(ldt.header.luminaire_name)
print(f"I(C=0°, γ=0°) = {ldt.intensities[0][0]:.1f} cd/klm")
```

The library expands the intensity matrix to the full `[MC × NG]` range
regardless of ISYM, so the caller never has to deal with symmetry reconstruction.

## References

- LiTG, *EULUMDAT Format Description*, Deutsche Lichttechnische Gesellschaft, 1990
- [pyldt documentation](https://github.com/123VincentB/pyldt/blob/main/docs/eulumdat_format.md)