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
* Material/color metadata (not yet full covered).

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

---

## Vertex Decompression

Vertices are read through a custom bitstream using the `Unpacker` helper.

Each vertex block begins with:

```
8-bit header:
    top 2 bits = magnitude index (0-3)
    low 6 bits = count-1 of vertices in this block
```

Magnitude determines the bit-width of each coordinate:

```
0 → 8 bits
1 → 10 bits
2 → 13 bits
3 → 16 bits
```

Every vertex contains:

```
X (signed)
Y (signed)
Z (signed)
```

Coordinates are integers, not scaled (i.e., raw model-space units).

---

## Normal Decoding

Normals use a custom spherical encoding.

### Special Case: Axis-Aligned Normals

If `x_raw == -64`, then the next 3 bits encode one of six axis directions:

```
+X, -X, +Y, -Y, +Z, -Z
```

Values returned are floating point unit vectors.

### Regular Case

```
x = unpack(7 bits signed) / 64
y = unpack(7 bits signed) / 64
z_sign = unpack(1 bit)
```

Z is reconstructed using:

```
z = sqrt(1 - x² - y²)
```

If the square root becomes slightly negative due to precision loss, the vector is renormalized.

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

* MBAC uses a tightly packed bitstream, so decoding must be strictly sequential.
* Many header fields are still unknown but handled safely.
* All transformations must remain in fixed‑point; floating‑point math breaks bone chains.
* Normals remain in floating-point because the engine uses them only for lighting.

The next steps are to understand the MTRA file format and fully refactor the converter.