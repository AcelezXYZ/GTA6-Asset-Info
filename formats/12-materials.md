# 12. Material classes and texture slots

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

Shader and parameter names are **joaat of the lower-case name** (confirmed):
`standard` = `bcf03fc1`, `diffusetex` = `c8e8c282`, `bumptex` = `1e77bf4c`.
Where the names come from:

- a shader preset list in `common.rpf` names all 344 shaders;
- uncompressed blocks in `shaders.rpf`, before the compressed cache, hold
  NUL-terminated technique, sampler and texture names and 24-byte binding
  records. Variables are `{u32 hash, u32 same hash, 0x30, …}`; texture
  bindings are `{texture hash, parameter hash, 02 01 xx xx, 00 80 00 00}`,
  e.g. (`diffusetexture`, `diffusetex`). 88 of 89 sampled drawable
  parameter hashes occur in these records;
- a word search: names not stored as text were found by joining likely
  words and testing joaat. Per-layer sets (`lyr0…`, `lyr1…`) check
  themselves; names that differ only in a final digit do not, because joaat
  states collide before the last character. Single matches of long
  multi-word names are noise and were dropped.

The texture-dictionary key at TXD +0x20 is plain joaat of the texture name
(lower case), as in GTA V.

77 of 229 parameters resolve, about 75 % of uses. 344 shaders over 254 K
drawables (1.83 M shader instances) in `levels.rpf` + `packs.rpf`.

Shader names give the families directly (lit signs and windows, e.g.
`ginos_neon_sign`, use `standard_emissive` in the alpha bucket and
`standard_emissive_layer` as cutout): emissive (`standard_emissive*`,
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
| `2c8d7182` | `tintpalettetex` | tint palette **embedded in the model**, on almost every map model: one row of RGBA8 sRGB colours (13 to 2,051, stored linear, padded with `ffbbff`). The texture struct's +0x30 pointer leads to a record whose +8 is the 64-bit store key, so these hashes are not in any `ttpf` store | confirmed |

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
| COLOR0 r, g, b | r and g unknown ([below](#vertex-colour-red-and-green)); b is the tint palette index ([below](#tint-palettes)) | — | r, g unknown |
| data maps (BC7) | R varies, G about constant per texture, B and A vary | — | unknown meaning |
| two-channel maps (BC1) | R = 0, G and B anti-correlated | — | unknown meaning |

The 17-parameter ground shaders are a different family, four layers by
their names (`standard_uber_4lyr_spx_dm_hbb`, `standard_uber_4lyr_pxm_hbb_fixuv1`,
`standard_uber_4lyr_dpm_dm_hbb`), and are not mapped yet.

**Normal maps are DirectX convention (green = +Y down)** (confirmed by an
integrability test on 270 of 300 maps). Invert G for glTF/OpenGL.

### Vertex colour red and green

Sampled on 4,000 models. Both channels are still **unknown**; every test so
far was negative (the export does not use them):

- **Red** looks like a baked shading term. It is mostly 255, about 23 on
  average on trees, and shows soft dark blobs on ground tiles where trees
  stand outside the model. It is about 252 on interior shaders, so it is
  not sky occlusion alone. Ray-cast sky visibility does not explain it,
  either from the model's own geometry or from the placed world around it:
  on two ground tiles cast against every placed model within 120 m, the
  correlation with red is about 0 (−0.02 and 0.06), and with green 0.06
  and 0.14 (confirmed negative result). Nearby instanced trees, the AO
  byte of the grass batches and the distance to the road do not explain it
  either.
- **Green** equals red on 63 % of ordinary vertices. On ground shaders it
  is low and changes from surface to surface (on one street tile grass 0,
  road mid grey, pavement 255), but it also varies widely within one
  collision surface (standard deviation about 54 of 255 on a road), so it
  is not a surface-type mask.

## Material presets (peds)

About 1,500 of the 4,678 meta resources in `packs.rpf` hold material
presets and sets (confirmed layout; field names are hashes):

| Struct | Fields |
|---|---|
| `f795e8ba` preset | name, parent, Category, a list of `37e52277` parameters, and two maps ([§8](08-meta.md), type 0x71): `12f85293` tags (key → u32 list) and `9a289423` |
| `37e52277` parameter | `3eaddcd7` (`StateName`) parameter hash (`44e21c90` = joaat("base")), a colour, `147fdab5` = four u64 texture hashes (`ttpf` keys, [§5](05-ttpf.md)) |
| `a9d57751` material set | name, a list of (slot, preset) pairs; face sets hold four skin variants in the "base" slot |

Tags in `12f85293` are joaat of the lower-case tag name: `002f77d9` =
"pedappearance", `a9473ab4` = "style". Key **`4bfd27dd` lists the drawables
the preset applies to** (joaat of the full drawable name); this is how a
head gets its face ([§7.5](07-models.md#ped-heads-face-textures-confirmed)).
Set and preset names do not resolve.

### Tint palettes

Slot `2c8d7182` (`tintpalettetex`, table above) is a palette embedded in the
model. **Vertex colour blue (`COLOR0.b`, stream 1) indexes it** (confirmed:
on 36 sampled models the index never reaches a filler `ffbbff` entry).
Tinted parts have bright, near-grey base textures (mean saturation 0.05),
so the tint multiplies the base colour (inferred): white pool tiles come out
pool-blue, white cables black, grey or blue.

- **Layout** (confirmed): a palette is 2^n + k texels wide. Each row holds
  256 >> k entries (at most 2^n), and there are 2^n / row length rows; rows
  are colour variants. Widths seen: 13, 20, 37, 66, 69, 129, 131, 256, 260,
  513, 517, 1,024 and 2,051. For example 1,024 = 2^10 + 0 gives 4 rows of
  256 (rows start at 256 / 512 / 768), and 131 = 2^7 + 3 gives 4 rows of
  32. Later rows start with white.
- **The entity's `tintValue`** (`3c852527`, [§9](09-placements.md)) selects
  the row (confirmed: for every palette width, the largest `tintValue`
  placed with it is rows − 1). Row 0 is the default.
- Colour = palette[row × row length + `COLOR0.b`].

**Render check (2026-11-22).** The rule above, multiplied into the base
colour and applied only to primitives whose own shader binds
`tintpalettetex`, was rendered in a 3D engine:

| Finding | Confidence |
|---|---|
| **Row 0 looks plausible.** A pier Ferris wheel hub and plaza come out white / grey, with coloured entries (green, cyan, red at indices 120–125) only on small parts. There is no in-game reference image | medium |
| **Rows above 0 look wrong as read.** Rocks on a 260-wide palette (16 rows × 16) placed with rows 2, 6 and 7 render near black; on the hub's 1,024-wide palette (4 × 256), rows 1–3 are almost all `ffbbff` filler. So the row rule is probably **incomplete**: either `tintValue` does not select the row directly (the clamp at rows − 1 is inferred), or some shaders choose the row differently. Treating the palette as sRGB instead of linear does not fix it | medium |
| **LOD drawables often bind no palette** even when the full model does, and there `COLOR0.b` is not a tint index (the Ferris wheel LOD has b = 0 with about 100 other distinct values) | high |

This contradicts the "confirmed" row selection above for rendering; the
`tintValue` maximum per width still holds, so both are kept until the row
lookup is understood.

**Texture load of map pieces** (measured on a full map export,
2026-11-22; high): a 64 m HD ground tile such as `n02_e01_002_hd` references
71 materials and about 170 textures, about 166 megapixels; big ground tiles
and building shells typically use 120–170 textures each. Loading all of
them when a tile appears stalls a frame, so streaming has to spread the
texture loads.

**Texture hashes in no store.** Embedded tint palettes account for 30,095 of
the 33,770 texture hashes that a full `levels.rpf` map export finds in no
`data/ttdb` store. The other 3,675 are in no store or resource of the
install, the content-pack stores included ([§5](05-ttpf.md); they serve the
packs' own models). They make up about 139 K references in a full map
export; `1e2c9340e14a7d07` and `aa3eeff23a15dc75` are probably
engine defaults (inferred).

## Open questions

- 152 of 229 parameter names (mostly constants), and the four per-layer
  slots of the ground shaders; the collision surface hash
  ([§13](13-collision.md)) is a different, unsolved one.
- The 64-bit texture hash ([§5](05-ttpf.md)): FNV-1/1a 64, xxHash64,
  MurmurHash64A, CRC-64 (ECMA, XZ, ISO) and a 64-bit one-at-a-time over
  46 K known names (texture-dictionary texture names, model names, XML
  identifiers), as given, lower and upper case, match no texture hash or
  either 32-bit half beyond chance.
- Whether `c6f46e79` G is roughness or gloss; emissive and glass flags for
  unknown shader families.
- How tint palette rows above 0 are selected (see the render check under
  [Tint palettes](#tint-palettes)).
- Emissive intensity: no brightness for lamps, neon or lit windows is
  decoded, so a port has to choose its own (the emissive texture is the
  base colour image; brightness is unknown).


---

← [11. Vehicle paint and vehicle tables](11-vehicle-paint.md) · [Index](README.md) · [13. Collision and surface categories](13-collision.md) →
