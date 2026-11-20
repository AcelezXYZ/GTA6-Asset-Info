# 12. Material classes and texture slots

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

Shader and parameter names are hashes with an **unknown hash function**.
About 1,500 candidate names were tried with joaat (both cases), FNV-1/1a,
CRC-32, Murmur3, djb2, sdbm, Adler and byte-swapped forms, with no match.
The compiled shader cache holds no names. 344 shaders over 254 K drawables
(1.83 M shader instances) in `levels.rpf` + `packs.rpf`.

## Shader record bytes +0x38..+0x3F

| Byte | Meaning | Confidence |
|---|---|---|
| +0x3C (byte 4) | render-bucket bit mask (below) | confirmed |
| +0x39 (byte 1) | bucket as an index (1 alpha, 2 decal, 3 cutout, 6 water), **+0x80 for instanced / batch-LOD draws** | confirmed |
| +0x3A..+0x3B | u16, maybe the constant-buffer size | unknown |
| +0x3D | always 0xFF | — |

| Bucket bit | Bucket | Instances | Shaders |
|---|---|---|---|
| 0x01 | opaque | 1.09 M | 193 |
| 0x02 | alpha (glass, bottles, VFX, windows) | 32 K | 44 |
| 0x04 | decal | 596 K | 59 |
| 0x08 | cutout (vegetation, cloth, wire) | 87 K | 90 |
| 0x40 | water | 14.6 K | 2 |
| 0x10, 0x20 | one vehicle shader | — | 1 |

**Emissive has no bucket bit**; only known shader families mark it. A
table of known families adds car_paint, window, glass, emissive, water,
terrain_multilayer.

## Texture slots

| Parameter | Holds | Confidence |
|---|---|---|
| `c8e8c282` | base colour, sRGB | confirmed |
| `1e77bf4c` | normal, BC1 (B = 255) | confirmed |
| `84c69350`, `58b6e707` | normal, BC5 | confirmed |
| `de84c011`, `4c759bf1`, `27dbd2be`, `7e1f7f44`, `1a016c7e` | terrain layer / detail normals | inferred |
| `c075f9b6` | terrain RGB layer-blend mask | inferred |
| `346394ea`, `7d4f12e2`, `7485dafb` | colour layers of the map blend shaders; `7485dafb` has opacity in alpha | inferred |
| `0df47048` | secondary / detail colour, sRGB | inferred |
| `c6f46e79` | packed: **R** binary mask (metalness?), **G** roughness or gloss, **B** AO / cavity, A unused | inferred |
| `739e3d0d` | packed: R region mask, G nearly flat (puddle blobs), B roughness/height-like, A AO-like | inferred |
| `19df25d7` | format 157 (unknown), vehicle shader | unknown |

## Two-layer multilayer shaders (walls, ground pieces, blend decal)

Shaders `b217b263`, `1251cae5`, `cd8ceb5e`, `77d8ab27` (12 parameters) and
`4f0b809b` (terrain blend decal) blend two full texture sets by vertex
colour. Derived from texture sizes and vertex/UV statistics over 4.7 K
materials, not from shader code (as of 2026-11-20).

| Item | 12-parameter family | `4f0b809b` | Confidence |
|---|---|---|---|
| base layer: colour, normal, data, two-channel | p0, p6, p7, p9 | p2, p10, p3, p0 | confirmed (slots of a layer always share an aspect ratio; the layers differ) |
| layer 2: colour, normal, data, two-channel | p4, p10, p5, p2 | p7, p4, p5, p9 | confirmed (same test) |
| overlay (colour + alpha, grime) | p8 | p6 | inferred |
| layer 2 weight | COLOR0.a, the only vertex channel painted over 0..255 | same | inferred |
| layer 2 UV set | TEXCOORD_1 (texel anisotropy matches in 108/117 and 54/72 primitives) | — | inferred |
| base UV set | TEXCOORD_0 (by elimination; TEXCOORD_2 is often constant) | — | inferred, weak |
| COLOR0 r, g, b | r mostly 255, g within 0..45, b a few distinct values | — | unknown meaning |
| data maps (BC7) | R varies, G about constant per texture, B and A vary | — | unknown meaning |
| two-channel maps (BC1) | R = 0, G and B anti-correlated | — | unknown meaning |

The 17-parameter ground shaders (`360e396b`, `b21f1ba5`, `61e09df7`) are a
different, many-layer family and are not mapped yet.

**Normal maps are DirectX convention (green = +Y down)** (confirmed by an
integrability test on 270 of 300 maps). Invert G for glTF/OpenGL.

## Open questions

- The hash function (would name shaders, parameters and collision
  surfaces, [§13](13-collision.md)).
- Whether `c6f46e79` G is roughness or gloss; emissive and glass flags for
  unknown shader families.


---

← [11. Vehicle paint](11-vehicle-paint.md) · [Index](README.md) · [13. Collision and surface categories](13-collision.md) →
