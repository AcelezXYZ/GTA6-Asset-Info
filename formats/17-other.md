# 17. Other formats

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

| Format | State |
|---|---|
| PSO (`PSIN`; sections LSCH/LENM/LSNS/LGID/LRSC/CHKS) | **mixed endianness** (corrected 2026-11-20): section headers are big-endian, and a section's length includes its 8-byte header ([structure/04](../structure/04-pso-objectgraph.md)); the block table inside is little-endian (u32 hash, u32 length, `0xAA` padding), and so are the 64-bit `(offset << 20) \| block` references; hashed field names (64-bit in the vehicle tables); exported raw |
| RBF (`RBF0`) | a new variant with u16 indices and inline names; exported raw |
| `core_bvh4.rpf` | BVH4 collision with packed 24-bit indices; not decoded |
| XML, text | plain; exported as is |
| shader cache (`shaders.rpf`) | one compiled cache, no names; layout notes in [structure/06](../structure/06-models-shaders.md) |
| audio (AWC/ADAT) | container layout, chunk tags and Opus framing in [structure/03](../structure/03-audio-awc.md); not exported by the tool |
| animation (`anim.rpf`) | two families: `0DRP` v122 bodies (cutscene/scenario metadata) in the low region of the archive, and clip dictionaries (bodies whose `+0x10..+0x3F` is all zero) above it; clip names, durations and sequence channels decode, and sampled rotation tracks validate as unit quaternions (3.95 M values, worst \|q\| − 1 about 1.2e-7). The skeleton is not in the clip dictionaries |
| music catalogue (`audio:/config/music.dat`) | compiled parsable data, magic u32 `0x045A88EA` at `0x08` after a root count; type-tagged records with `0xAAAAAAAA` as the null sentinel. Title record (tag `0xAAAAAA0C`): `+0x08` u32 title hash, `+0x0C` char[64] title, `+0x60` u32 credit count, then 8-byte credits. The title hash is joaat of the **lowercased** title with seed `0xC3CCF6AD` (804/804 titles); unseeded joaat of the title is what AWC stream ids use, so the two never coincide |
| PSIN scene placements (`story000_script.rpf`) | 64-byte records at a 16-byte stride inside PSIN tag blocks, 97,504 in 2,017 documents: `+0x00` u32 handle (repeated at `+0x30`), `+0x10` f32 × 3 position (Z-up metres; x repeated at `+0x1C`), `+0x20` f32 × 3 rotation in radians (x repeated at `+0x2C`), `+0x34` u32 archetype = joaat(lowercase name), `+0x38` u64 id (zero in about half the records). A different family from the map-metadata entities in [§9](09-placements.md) |
| movies (`movies.rpf`) | Bink 2: 515 movies, 505 `KB2n` and 10 `KB2j` (magic `KB2` + revision byte); header, frame index and Bink Audio tracks parse; video needs a Bink 2 decoder |
| text (GXT) | `0DRP` type `0x7a` resources: a tag table (`0x101` key table, `0xA8478889` locators, `0x956A9DF5` / `0x10075946` locale-variant groups and items, `0x10` text pages); keys are joaat in 4,096 or 2,048 buckets; 54,118 keys |

---

← [16. Road graph, junctions and flags](16-roads.md) · [Index](README.md)
