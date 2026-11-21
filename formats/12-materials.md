# 12. Material classes and texture slots

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

Shader and parameter names are **joaat of the lower-case name** (confirmed):
`standard` = `bcf03fc1`, `diffusetex` = `c8e8c282`, `bumptex` = `1e77bf4c`.
Where the names come from:

- a shader preset list in `common.rpf` names all 344 shaders;
- uncompressed blocks in `shaders.rpf`, before the compressed cache, hold
  NUL-terminated technique, sampler and texture names and 24-byte binding
  records. Texture bindings are `{texture hash, parameter hash, 02 01 xx xx,
  00 80 00 00}`, e.g. (`diffusetexture`, `diffusetex`);
- a word search over likely parts. Per-layer sets (`lyr0…`, `lyr1…`) check
  themselves; names that differ only in a final digit do not, because joaat
  states collide before the last character.

77 of 229 parameters resolve, about 75 % of uses. 344 shaders over 254 K
drawables (1.83 M shader instances) in `levels.rpf` + `packs.rpf`.

Shader names give the families directly: emissive (`standard_emissive*`,
`signs_emissive`, `*_emissive_layer*`, with an `emissivetex` slot), glass
(`standard_vehicle_glass`, `standard_glass_breakable*`), water
(`standard_alpha_water`, `water_micro_geom`, `standard_alpha_waterfall`),
decals (`standard_decal_hbb`), peds (`ped_standard_skin`) and vehicles
(`standard_vehicle_2lyr`).

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

**Emissive has no bucket bit**; only the shader family marks it (names
above). A table of known families adds car_paint, window, glass, emissive,
water, terrain_multilayer.

## Texture slots

| Parameter | Name | Holds | Confidence |
|---|---|---|---|
| `c8e8c282` | `diffusetex` | base colour, sRGB | confirmed |
| `1e77bf4c` | `bumptex` | normal, BC1 (B = 255) | confirmed |
| `84c69350`, `58b6e707` | `lyr0normaltex`, `lyr1normaltex` | normal, BC5 | confirmed |
| `27dbd2be`, `7e1f7f44`, `4c759bf1`, `de84c011`, `1a016c7e` | `bumptexture_layer0`–`3`, `tintbumptexture` | terrain layer / detail normals | inferred |
| `c075f9b6` | `lookuptexture` | terrain RGB layer-blend mask | inferred |
| `346394ea`, `7d4f12e2`, `7485dafb` | `lyr0diffusetex`, `lyr1diffusetex`, `dirtdiffusetex` | colour layers of the map blend shaders; `7485dafb` has opacity in alpha | inferred |
| `0df47048` | `diffusetex2` | secondary / detail colour, sRGB | inferred |
| `c6f46e79` | `speculartex` | packed: **R** binary mask (metalness?), **G** roughness or gloss, **B** AO / cavity, A unused | inferred |
| `739e3d0d` | `speculartex2` | packed data. On wet-capable surfaces: R region mask, G nearly flat (puddle blobs), B detailed, A AO-like. **Most maps are R = G = 0, B ≈ 0.5**, so B is not roughness: read that way, surfaces turn mirror-like. Treat the channel meaning as unknown | inferred |
| `19df25d7` | ? | format 157 (unknown), vehicle shader | unknown |

**Placeholder textures** (confirmed). Many building materials point their
packed-data slot (`739e3d0d`) at a 4 × 4 all-zero texture, and some point
the normal slot at a 4 × 4 black one. Read literally, the first gives
AO = 0 (black walls) and the second a broken normal. Ignore textures of
8 px or less in the packed and normal slots. A few materials have only such
placeholders, base colour included (e.g. one shader of
`dn_20_int_hse_14_exterior`); they draw as black boxes and are probably
non-visual proxies (inferred).

## Two-layer multilayer shaders (walls, ground pieces, blend decal)

Shaders `standard_2lyr_pxm` (`b217b263`), `standard_2lyr_pxm_ground`
(`1251cae5`), `standard_2lyr_ground` (`cd8ceb5e`), `standard_2lyr`
(`77d8ab27`) (12 parameters) and `standard_2lyr_pxm_decal_ground`
(`4f0b809b`, terrain blend decal) blend two full texture sets by vertex
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

The 17-parameter ground shaders are a different family, four layers by
their names (`standard_uber_4lyr_spx_dm_hbb`, `standard_uber_4lyr_pxm_hbb_fixuv1`,
`standard_uber_4lyr_dpm_dm_hbb`), and are not mapped yet.

**Normal maps are DirectX convention (green = +Y down)** (confirmed by an
integrability test on 270 of 300 maps). Invert G for glTF/OpenGL.

## Open questions

- 152 of 229 parameter names; the collision surface hash
  ([§13](13-collision.md)) is a different, unsolved one.
- Whether `c6f46e79` G is roughness or gloss; emissive and glass flags for
  unknown shader families.


---

← [11. Vehicle paint and vehicle tables](11-vehicle-paint.md) · [Index](README.md) · [13. Collision and surface categories](13-collision.md) →
