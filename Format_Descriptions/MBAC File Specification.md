# MBAC Format Specification (Reverse-Engineered)

@author [Zé Gabriel (KidGbzin)](https://github.com/KidGbzin)

This document describes the MBAC model format as understood and implemented in the conversion code we developed. It is based on the public MBAC loader by [Minexew](https://github.com/minexew), but the decoding, refactoring, fixed-point corrections, and all final implementation work were written by [me](https://github.com/KidGbzin). All information here is derived from reverse engineering and may not represent the full official specification, but it is accurate for all MBAC files tested so far.

---

## Overview

MBAC is a compact 3D model format used primarily in Mascot Capsule engines. It stores:

* Vertices (compressed bitstream);
* Normals (spherical encoding);
* Flat and textured polygons;
* Bone hierarchy and transformations;
* Material/color metadata (not fully covered yet).

All transformations and matrices in MBAC use **4.12 fixed‑point format** (1 unit = 4096).

---

## File Header

The file begins with:

```
uint16 magic        # Must be 0x424D ("MB").
uint16 version      # Only version 5 supported.
```

Followed by four bytes indicating the formats used:

```
uint8 vertex_format     # Only 2 supported (compressed vertices).
uint8 normal_format     # Only 2 supported (spherical encoding).
uint8 polygon_format    # Only 3 supported (textured + flat).
uint8 bone_format       # Unknown, ignored.
```

Then:

```
uint16 number_of_vertices
uint16 number_of_tris
uint16 number_of_quads
uint16 number_of_bones

uint16 number_of_flat_tris
uint16 number_of_flat_quads
uint16 number_of_materials
uint16 unknown_bits_21      # Controls material section size.
uint16 number_of_colors     # Used only by flat polygons.
```

---

## Material Section

The material block structure is not fully understood. It consists of:

* `unknown_bits_21` blocks, each containing:

  * Two `uint16` values;
  * Followed by `number_of_materials` pairs of `uint16`.

These values are read and discarded since no MBAC tested uses materials.

``` 
repeat unknown_bits_21 times:
    uint16 unknown_1
    uint16 unknown_2

    repeat number_of_materials times:
        uint16 unknown_3
        uint16 unknown_4
```

---

## Vertex Decompression

When an MBAC file uses vertex format `2`, all vertices are stored inside a compressed bitstream. The bitstream is divided into blocks, and each block begins with a 1-byte header.

The two most significant bits of this header determine the magnitude, which specifies how many bits are used to store each coordinate (X, Y, Z) inside that block.

The magnitude therefore defines the bit-width and numeric range for all vertices encoded in the block.

| Magnitude | Code | Bits per Coordenate | Signed Type   | Numeric Range     |
| --------- | ---- | ------------------- | ------------- | ----------------- |
| 0         | 8    | 8 bits              | sint8         | -128 to +127      |
| 1         | 10   | 10 bits             | sint10        | -512 to +511      |
| 2         | 13   | 13 bits             | sint13        | -4096 to +4095    |
| 3         | 16   | 16 bits             | sint16        | -32768 to +32767  |

---

#### Why magnitudes exist?

Using different magnitudes allows the MBAC encoder to automatically choose the smallest possible precision for each block of vertices:

- Small details: 8 or 10 bits;
- Larger structures: 13 or 16 bits.

This significantly reduces file size while maintaining enough precision for rendering.

---

## Normal Decoding

Normals use a custom spherical encoding.

When an MBAC file uses normal format `2`, all surface normals are stored using a compact spherical encoding.

Unlike vertices (which store X, Y, Z explicitly), this encoding reduces a normal vector to just 15 bits, instead of 36 bytes used by three floats.

This compression works by storing only:

- X (7-bit signed);
- Y (7-bit signed);
- Z sign (1 bit);
- Or a special 3-bit “direction code” for axis-aligned normals.

The decoder reconstructs the final Z component mathematically.

---

Each normal is encoded as one of two forms:

1. **Special Direction Code (when X = -64)**

    If the 7-bit X value equals -64, the normal does not contain numeric X/Y data.
    Instead, the next 3 bits represent one of six cardinal directions:

    | Direction Code | Normal Vector  | Meaning |
    | -------------- | -------------- | ------- |
    | 0              | ( +1,  0,  0 ) | +X axis |
    | 1              | ( -1,  0,  0 ) | -X axis |
    | 2              | (  0, +1,  0 ) | +Y axis |
    | 3              | (  0, -1,  0 ) | -Y axis |
    | 4              | (  0,  0, +1 ) | +Z axis |
    | 5              | (  0,  0, -1 ) | -Z axis |

    This special case is used when the normal is perfectly axis-aligned, saving storage space.

---

2. **Regular Spherical Encoding (any X ≠ -64)**

    If X is not -64, the normal is encoded in compact spherical form:

    | Component | Bits | Type  | Description    |
    | --------- | ---- | ----- | -------------- |
    | X         | 7    | sint7 | X / 64.0       |
    | Y         | 7    | sint7 | Y / 64.0       |
    | Z sign    | 1    | bit   | 0 = +Z, 1 = –Z |

The decoder reconstructs the missing Z component using the unit-length constraint:

```x² + y² + z² = 1 → z = sqrt(1 - x² - y²)```

After computing Z, its sign is applied from the stored 1-bit flag.

If rounding errors make `(1 - x² - y²)` negative (possible due to 7-bit precision), the decoder clamps or falls back to a safe value.

---

#### Bit allocation summary

| Field Type                | Bits | Notes                   |
| ------------------------- | ---- | ----------------------- |
| Special-case flag (X=-64) | 7    | triggers direction mode |
| Direction index           | 3    | only used if X = -64    |
| X                         | 7    | signed, scaled by 1/64  |
| Y                         | 7    | signed, scaled by 1/64  |
| Z sign                    | 1    | 0 = +Z, 1 = -Z          |

Total (regular): 15 bits per normal.
Total (direction code): 10 bits per normal.

---

#### Why spherical compression?

Storing X and Y with only 7 bits each reduces noise but saves a lot of space:

- Float normal: `3 * 32 bits = 96 bits` (12 bytes).
- MBAC spherical normal: `7 + 7 + 1 = 15 bits` (≈ 2 bytes).

This gives 6x reduction in size, essential for early Java ME devices.

---

## Flat Polygons

Flat polygons store only vertex indices and an unused color identifier.

Block header:

```
unknown_bits
vertex_index_bits
color_bits
color_identifier_bits
unused
```

Flat polygons appear first:

* `number_of_flat_tris` triangles
* `number_of_flat_quads` quadrilaterals

All flat polygon color values are ignored because every known MBAC uses textures.

---

## Textured Polygons

Textured polygons have:

```
unknown_bits
bits_per_vertex_index
bits_per_uv
unused
```

Data for each face:

* Vertex indices: encoded with `bits_per_vertex_index`;
* UVs: each U and V encoded with `bits_per_uv`.

UV values are raw integers `(0–(2 ^ bits_per_uv) -1)`.

Two polygon types are stored:

* `number_of_tris` → (v1, v2, v3) + 3 UV pairs;
* `number_of_quads` → (v1, v2, v3, v4) + 4 UV pairs.

---

## Bone Hierarchy

Each bone entry begins with:

```
uint16 vertex_count    # How many vertices it influences.
int16 parent_index     # -1 for root.
```

Followed by a **3x4 fixed-point transform matrix**:

```
12× int16 → (row-major order) → 4.12 fixed point
```

Vertices are assigned to bones sequentially. Meaning:

* bone 0 influences vertices 0..N
* next bone influences the following vertices

The sum of all vertex counts must equal `number_of_vertices`.

### Bone Transformation

Parent matrices are accumulated using **fixed‑point matrix multiplication** (no floats) to avoid precision drift.

Vertex transformation uses:

```
outX = (m00 * x + m01 * y + m02 * z) >> 12 + m03
outY = (m10 * x + m11 * y + m12 * z) >> 12 + m13
outZ = (m20 * x + m21 * y + m22 * z) >> 12 + m23
```

---

## Final Notes

- MBAC uses a tightly packed bitstream, so decoding must be performed strictly in sequence.

- Several header fields remain undocumented, but all of them are safely handled by the decoder.

- All bone transformations must stay in fixed-point; using floating-point breaks parent–child accumulation.

- Normal vectors are converted to floating-point after decoding because Mascot Capsule uses them only for lighting calculations.

---

## Next Steps

The next steps are to fully map the MTRA animation format and perform a complete refactor of the converter with a cleaner architecture and modular decoding pipeline.