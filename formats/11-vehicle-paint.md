# 11. Vehicle paint

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

| Item | Finding | Confidence |
|---|---|---|
| colour slot `c8e8c282` on vehicle shaders (`4aa38282` family) | points at an engine-default **white** texture found in no store; the body colour is a runtime paint colour, as in GTA V | confirmed |
| `f151c063` | decal / sticker layer, RGBA | inferred |
| `d37026cb`, `640111c6` | dirt / grime / wear masks | inferred |
| `84d22f74` | fine scratches and flakes | inferred |
| `b9f4bdac` | grime | inferred |
| `fb03ae36` | green-dominant mask (mask pack or normal detail) | unknown |
| `4d56d7fe` / `2bb2ede8` | BC1 normal atlas | inferred |
| `84c69350` / `58b6e707` | BC5 normals | confirmed |
| shader constant buffer (+0x08) | ~0x160 bytes of floats, layout not keyed by the parameter table | unknown |

**Colour tables: not decoded.** The vehicle / mod-kit table is a PSO in
`data.rpf` with **64-bit** struct and field hashes and joaat model names.
Its block table is parsed (120 blocks, 16-aligned, references as in [§8](08-meta.md)); a
vehicle's record points to 8 polymorphic float sub-records, which look like
handling-type data, not colours. Its schema section is not parsed. A pool
of 17,632 ARGB colours in the game-data resource (struct `fc1c7000`, 100
colours per object) may be palettes but is not tied to vehicles.

**Converters must take the paint colour from the user** for now.

## Open questions

- Paint colour tables and liveries; the PSO `LSCH` schema with 64-bit
  hashes; the shader constant buffer.


---

← [10. Interiors (MLO)](10-interiors.md) · [Index](README.md) · [12. Material classes and texture slots](12-materials.md) →
